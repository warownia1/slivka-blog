---
layout: post
title: Design improvements - the onion architecture
date: 2022-01-08
---

A new year is a good time to take a step back and analyse the structure of the code.
Recently, I took interest in principles behind the Clean Architecture which may
help me write better quality code. Some of the design ideas were already known to me
or were intuitive enough that I unknowingly implemented them when writing Slivka code.
Yet, there are many areas in Slivka design that are lacking. Many of my solutions and
design choices were inspired by Django framework that I used in my previous projects
and considered an exemplar. Now, I started to see that in many areas Django focuses
on delivering code quickly, but not necessarily of a good quality.

Abstracting storage access
==========================

One of the rules in the onion architecture is separation of the core business
logic from the implementation details. In this case, the implementation detail
is how the data is persisted. Formerly, we used a relational database through a
SQLAlchemy interface. Now, we use mongo database. The change of the database
required multiple modifications to the application code, as the database
calls were scattered across many different places. This could have been avoided
if the data were accessed through a universal interface and concrete implementations
using either sql or mongo were plugged-in afterwards. 

Currently, database access is not quite separate from the application logic. The
CRUD operations are managed by the functions in the `slivka.db.helpers` module.
However, the database models and domain entities are still the same objects.
Also, more complex database queries are performed directly in the application layer.

Is it really bad that database models and domain entities are mixed together
and the application logic is tightly coupled to the database used?
It depends. Having the database access implementation separated from the core
logic would definitely allow swapping the former with no hassle when needed, but
changes to the underlying database system rarely happen and the effort put into
swapping the database would be put into factoring database calls out from the
logic code instead.
Also, having database models separate from the domain objects brings
extra challenge of keeping those two synchronised. The logic layer would need to
perform the same operation twice, for example: updating the status of the
accepted job requests would require changing the domain objects
that the scheduler operates on and then requesting the storage to change the status
of accepted job requests in the database.

The benefits of database separation would not only be the cleaner code
with clean domain boundaries which will be easier to read, maintain, and test. 

The REST server can benefit from the separation as it doesn't need
to know where the data about jobs come from. Currently they are database
records, but contacting the scheduler process to get current job information
using ZMQ is a viable option as well. Data Access Objects offer greater
flexibility, changing data source would be a matter of plugging a
different DAO implementation without any changes to the application logic.
Configuration i.e. service configuration and status could also be accessed
using DAOs. Depending on the implementation, they would fetch service
information from the file, database or ask the scheduler to provide
the services. It makes it much easier to implement dynamic service
discovery during server operation.

The scheduler can also benefit from data layer separation. First,
the scheduler logic will not be tainted with database calls and
protective measures in case of database malfunction. Second, the data
source will no longer be limited to the database. An alternative 
for sending i.e. job requests I strongly consider is using ZMQ.
The logic layer should be oblivious to where the input data comes from
and should only focus on processing it. On the other hand, DAOs will
be responsible for providing that data either from the database or a ZMQ
socket or an in memory queue.


Introducing use cases
=====================

Use cases a.k.a. interactors, in the onion architecture, sit in the application
logic layer and determine the core behaviour of the system which should be
independent of frameworks used. The frameworks located in the outermost layer
interact with the system by calling those use cases.

In slivka, currently, the application logic is integrated with the web framework.
It might be sufficient for a small application which slivka is, but few drawbacks
has already became apparent during the development process:

- no clear listing of operations that the system can perform
- data presentation is mixed with application logic making it slightly harder 
  to comprehend
- the same logic must be repeated in each external interface e.g.
  result retrieval logic is repeated in both rest api view and website view.
  Adding e.g. CLI would require repeating the same logic again.
- logic cannot be tested without going through the framework

All interactions with the application logic should be moved to
use case functions, so that they can be used from different entry points
(including tests) without code repetition. The separation is good
regardless if we opt for the onion architecture or not.

Splitting scheduler into layers
===============================

The piece that may require refactoring the most is the scheduler
(which should actually be called a dispatcher). The biggest problem
with it is that it's a monolith that is impossible to (unit) test.

In one of the recent updates I extracted and separated the code
responsible for constructing the command line arguments and sending
the jobs to the external workload manager. The separation was a good
thing as it allowed to use and test command line builder and workload
manager interfaces in isolation.

The next step should be splitting the scheduler further into smaller
pieces where each can be run and tested independently of the rest of
the system. I hope that after organising the scheduler's pieces properly,
all elements that currently feel out-of-place (e.g. runner status 
monitoring or job cancellation) will fit into their right place
intuitively.
It should be rebuilt from the centre outwards, starting with the
domain entities and logic and ending with peripherals such as
external data storage, messaging queue and external job managers.
In its core, the scheduler should:

- receive new job requests
  - scheduler should be oblivious to how the data arrived
  - a separate component should deliver new requests to the scheduler
    either by pushing of polling.
    it's an implementation detail how the data arrives (either ZMQ
    or database polling).
- group unprocessed requests to appropriate runners
  - grouping should be delegated to a separate component/class
    that should be testable in isolation
  - unprocessed requests include new requests as well as those which
    failed to run in the previous iteration
- submit jobs to external execution systems
  - scheduler works with abstract runners only, concrete implementations
    are provided to it from the outside
- monitor jobs and communicate state changes
  - the monitor can be extracted to a separate class as well

Fetching new requests from the database should preferably be extracted
to a dedicated layer. Currently, the database calls are scattered all over the
scheduler's code each wrapped with retry-on-failure function which
significantly impede code clarity and may interfere with proper execution.
Moving all of that to a separate layer, however, has a risk of introducing extra
complexity and race conditions, especially if the requests are to be fetched
in a separate thread and pushed to the scheduler asynchronously.
Adding job cancellation requests to the mix complicates everything even
further. The benefits of the separation would be more organised data
flow and testability (for now, the scheduler cannot be tested without
a database). Other benefit, which may prove useful in the future, is
ability to replace the database with ZMQ for data exchange.

Inversion of control
====================

In onion architecture, the application logic and domain entities sit
in the central layer and can only communicate with external layers
through interfaces. This mostly affects database and filesystem access
that application logic would normally call directly. Instead, the
concrete implementations should be given to the interactors from the
outside i.e. as a call argument or injected during setup by an injector.

Currently, all components use database directly and are, hence, tightly
coupled to it. It has a negative impact on testing which cannot be easily
performed. The operation of the components must be tested by examining
side effects on the database rather than inspecting calls and return values.
This makes the system less reliable overall.

Using some form of dependency injection would also greatly help to
write tests which won't need to monkey-patch the
database object in the `slivka.db` module. It will also solve the problem
"when and how to pass dependencies (database and configuration) to
the application functions". This responsibility can be moved to a
dependency injection configured on startup, that will inject required
objects on demand. Notable DI libraries are [python-inject](https://pypi.org/project/Inject/)
and [kink](https://pypi.org/project/kink/).


Conclusions
===========

Will slivka benefit from fully switching to the onion architecture?
I think the answer is: no. The onion architecture is certainly
a powerful tool that helps to maintain big and complex systems which
slivka is not. Implementing all the layers, interactions, isolations
etc. and maintaining them would take significantly more work than
developing the system itself. 
The more suitable architecture pattern would be a layered architecture
that slivka already tends towards. This is more suitable approach for
small systems where layers can be coupled more tightly without much
impact on maintainability.
The current project structure should be reviewed with layered
architecture in mind, notably:
split larger monoliths into smaller components with well-defined
responsibilities and draw clear boundaries between application layers.

In short, for the REST server:

- detach application logic from the api views; logic should be fully
  usable without the existence of the framework
- abstract database (any data) access using Data Access Objects;
  the data the server uses might not necessarily come from the database,
  exchanging data directly with the scheduler with ZMQ is a viable
  alternative

for the scheduler:

- split the monolith into smaller, testable components
- divide components into layers
  - data exchange + persistency
  - domain logic
  - external infrastructure interface

if the scheduler and it's components can be used programmatically
without mocking the database, it's a good sign.
