---
layout: post
title: Distributing messages in slivka, feat. Redis
date: 2022-09-23
---

Messaging systems are a solution for the maintenance and scalability
issues of distributed systems. The aim is to write less complicated,
loosely coupled concurrent systems which are not a nightmare to
maintain and scale up. The principle is simple: you break your program
into smaller pieces (nodes) which perform their jobs independently
from the rest of the system. The nodes communicate with one another
through the messaging system. The lack of tight connections between
the nodes allows workers to appear and disappear without changing the
overall topology of the system.

Slivka is not dealing with problems of large distributed systems
because of its small size. However, it can benefit from the solutions
provided by the queuing systems as it relies on asynchronous data
exchange between its subcomponents. In fact, message queues are
already present in the slivka application. The first noteworthy
message exchange appears between the REST server and the scheduler
process. The multiple web workers, running in different threads or
processes concurrently send new job requests to a single scheduler
process. The number of workers should be able to go up or down without
affecting the rest of the system, so they use a central hub (mongo
database) to which they push messages that are later picked up and
processed by the scheduler. Even though the mongo database is not
strictly a queuing system, in this case, it provides both persistency
and, a bit clunky, message queue. The second case is the exchange of
command line parameters between the scheduler and a local execution
queue. Here, the main program uses ZeroMQ to send messages to the
local execution queue with new jobs or status requests. The loose
coupling of the component resulting from the use of a message queue
really proved itself. Despite making many changes to the rest of the
system I could always reuse this small piece of code without any
adjustments.

One major change which I'd like to introduce before the version 1.0
release is replacing the use of the mongo database to exchange
messages between the REST server and the scheduler with a proper
messaging queue. The database is not an inherently bad solution from
the front-end server perspective, MongoDB offers great options for
concurrent reading and writing, so multiple workers can easily push
messages to the database at the same time. Additionally, job requests
are automatically stored in persistent storage. There are however
several downsides to it. Disk-based storage is significantly slower
than an in-memory queue, which combined with a blocking nature of
database transactions may not handle large throughputs well. Moreover,
things don't look that nice on the receiving end, the database-based
queue does not notify the receiver of arriving messages. The scheduler
process must periodically query the database for new entries which is
neither elegant nor optimal and introduces unnecessary waiting periods
of several seconds which could be better spent running jobs.

Let's contrast it with a generic queue (I'm not jumping to any
specific queue implementation just yet) where one party pushes
messages onto one end and the other pulls them from the other end of
the queue. The requirements we have for the queuing system are the
following:

- multiple workers must be able to push to the queue in parallel
- pushing should not be affected by slow I/O
- requests should ultimately be persisted
- receiving end should be notified of new messages, so it can work reactively

## Overview of Redis

Before we analyse the utility of Redis in slivka software, let's give
a quick overview of what Redis is. According to the [Redis website], 
Redis is an "open source in-memory data
store used (...) as a database, cache, streaming engine, and message
broker". It can performantly store and retrieve data in form of key
and value pairs, making it a great choice for a fast data cache. It
supports data types such as strings, lists, sets, hashes and more. It
also features message broker capabilities and can be used as a central
hub for exchanging messages between or within processes.

[Redis website]: (https://redis.io)

In this post, I mainly focus on messaging patterns that Redis
provides.  A nice overview can also be found in this [Message Queue in
Redis] blog post. Although I show them in the context of Redis, the
patterns are reusable with different queuing systems.

[Message Queue in Redis]: https://selectfrom.dev/message-queue-in-redis-9efe0de2c39c


## PUB/SUB

The simplest (and probably least useful) pattern is a
publisher/subscriber pattern in which there are one or more publishers
producing messages to messaging channels that subscribers subscribe
to. Whenever a message is posted to the channel, it is distributed to
all subscribers listening to that channel at the moment. While the
pattern nicely separates data sources from receivers, the side effect
is that the publishers may send messages to the void, if there is no
receiver observing the channel on the other end. This behaviour is
unacceptable for two reasons: requests cannot just disappear and
should wait for being processed, and each request should be delivered
to exactly one receiver, not all of them at once.

## FIFO Queue

A significantly more useful pattern is a first-in-first-out queue
pattern. Redis has great support for queues with the linked list data
structure. In principle, producers push messages to the end of the
list (using the `RPUSH` command) while one or more consumers wait for
these messages to appear (using the `BLPOP` command). If the consumer
is not present or can't keep up with incoming messages they stay in
the queue waiting to be processed. This is good for small and simple
cases. The downside is once the message is taken from the queue, it's
gone from the database. There is no persistent history of messages and
if a process claims a message and dies, that message is gone forever.
Let's compare how the FIFO queue meets our criteria:

[x] multiple workers can push messages in parallel
[x] pushing is not affected by slow I/O
[ ] retrieved requests disappear
[x] receiving end is notified of new messages

It meets all requirements except keeping a history of job requests,
therefore it needs additional backing of a persistent database.

## Stream

A stream is a data structure introduced in Redis version 5.0. It acts
as an append-only log of events which can be read in order by
consumers or consumer groups. Similar to the pub/sub all events are
distributed to all available consumers, but contrary to pub/sub, the
events are stored in a queue, so new consumers can either start
listening to new events or browse the history of old events they
haven't seen yet. The old messages never disappear from the stream. It
is also possible to partition messages to different consumer groups
such that events assigned to one consumer group do not appear to other
groups. It allows replicating a behaviour of the queue pattern, where
messages are distributed to particular consumers, not broadcast to all
of them. An in-depth tutorial on how to use Redis streams can be found
on the [Redis Streams tutorial] page of the Redis documentation. The
streams check out all our criteria:

[x] multiple workers can push messages in parallel
[x] pushing is not affected by slow I/O, stored in memory
[x] events stream persists in the database
[x] receiving end is notified of new messages

[Redis Streams tutorial]: (https://redis.io/docs/data-types/streams-tutorial/)

## Do we need Redis?

Is the Redis database with its streams a perfect solution? Not really.
Let's start with the pros of using Redis with slivka:

- in-memory database Redis provides is a great solution for caching
  data; e.g. the scheduler can store the current job status and the
  REST server can quickly retrieve the cached value;
- streams provide a robust alternative to the queue with inherent
  history of events;
- Redis can be an almost drop-in replacement of a MongoDB.

Cons:

- slivka is meant to be small, adding a broker that requires separate
  configuration and maintenance adds a large footprint
- using an in-memory database for persistent data can eventually lead
  to large memory usage, an alternative is to use TTL on every stored
  value, but in such case, long-term data must be archived on a
  secondary disk-based database.

Since slivka is not an enormous distributed system requiring
a large data throughput and parallelism, using Redis might be an
overkill. A solutions targeted at small to medium application may
actually work better in our case. One of those is ZeroMQ, which is
a small, yet powerful and portable queuing system, which works with
no extra setup and no broker.
