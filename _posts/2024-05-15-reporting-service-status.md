---
layout: post
title: Reporting service status
date: 2024-05-15
---

One of the requested features was monitoring and reporting the availability
status of each service. This feature is present in JABAWS and involves running
test sets automatically every 10 minutes. The tests can also be triggered by users
through the website. Each entry shows the name of the service, version, service
status with test output and additional service details. I would like to
replicate that feature and possibly improve on it. I'd like to remove
the user's ability to start tests as it opens the door for DoS attacks.
Secondly I'd like to optimize service testing to conserve computational power
used for running tests.

## Using user jobs to monitor status

My initial implementation was using jobs run by the users to determine the service
status. Any failures during job executions were recorded in the database and
were used to infer the current service status. That way, the front-end users
could see the service status from the last time it was used. Assuming the service is used
regularly, we could skip running test jobs saving resources.
The database stores entries containing the service name, the timestamp and the last
status with an error message for each service which updated by the scheduler every
time jobs are run.

The first problem I encountered was determining the status of the service in
case there is more than one runner associated with the service. In case of two
runners of which one stopped working, if the program shows the status of
the service according to the last batch of jobs, the status will be *ok* or
*down* depending on which runner was called more recently. That doesn't accurately
reflect the actual status of the service where one of the runners is completely
inoperative.

The solution I used involved tracking the status of each runner individually and
then report the most severe error as the overall service status. If all statuses of the
runners for a particular service are *ok* then the service status is *ok*; if
one of them changes to *warning* then the overall service status is *warning* etc.

While it fixed the problem with multiple runners almost identical problem
appeared. The runners could fail at different stages of the job execution. Just
like failing to start jobs with one runner and succeeding with the other doesn't
make the service fully functional, failing at one execution stage and passing
the other doesn't make it operational either. E.g. the runner may successfully
submit jobs to the queueing system, but all the jobs fail just after due to the
lack of permission to execute the program; now the successful submission is
competing with job status errors. Here is a non-exhaustive list of issues which
I encountered on my way:

- Runner may fail to submit the job to the queuing system, which typically
causes an exception when starting jobs. It could be caused by an invalid runner
configuration or an improper setup of a queuing system. Those problems prevent
the service from being run altogether.
- The job was sent successfully but couldn't be started e.g. due to missing
executable, permissions or invalid arguments. The first two problems can be
identified by 127 and 126 status codes respectively if the process was started
with bash. The invalid arguments cannot be easily differentiated from other
non-zero status codes returned by the process. Those problems are caused by
invalid service configuration and permanently prevent services from running.
- The job was started successfully but the user input was invalid and the
process returned a non-zero status code.  Those should not be considered service
errors as they are caused by malformed input and not the service.
- The job was successfully sent to the queuing system but no worker collected
the job, or the worker is dropping jobs. It could be either a temporary
malfunction or a permanent issue. It could be hard to determine what happens in
the queuing system by simply watching the job status.
- The job run successfully but status checks fail. It could be caused by
improper implementation of the runner or a scheduling system being temporarily
down. Depending on the origin of the problem, the service should be reported to
be down or having temporary issues.

A next logical step is to record status for each execution stage separately and
report that the runner is functional only if all of the stages completed
successfully. For example, stages can be split into
*submission*, *startup*, *execution*, *polling*. That way, changes
to the status of one stage does not effect the others.

## Periodic test jobs

Relying on user jobs has several disadvantages. If the service is not used
frequently, it's also not being tested. We won't be able to know about errors
until someone tries running the service and fails. Other disadvantage is that
user jobs can be vastly different and unpredictable. Having a set of standard jobs
that would be used to test services thoroughly is much more reliable.

It brings us back to having the test jobs periodically run by the system in the
background (e.g. separate thread) to monitor system health status. Those jobs
could be defined in the service configuration files and run by the scheduler in
the set time intervals using available runners. A new problem is how to fit the
new status monitoring into the existing system. Remember that we already store a
status for each execution stage of each runner and adding one more entry to the
database would complicate things.

The question arises: "should results of test jobs override the status inferred
from the user jobs?" Consider a corner case in which test jobs fail but user
jobs complete successfully. Should the service be considered stable or not in
such case? What about the opposite, where user jobs fail, but test jobs pass?

Relying solely on tests jobs may be an easy and sufficiently reliable solution,
but it continuously wastes computational resources. Set testing rate too high
and the server will be continuously busy running useless computations.  Set it
too low and the status will be too slow to update and get stale.  Users may get
pretty annoyed trying to run the service reported as functional and finding out
it's broken or vice versa, avoiding a service which crashed a few hours ago, but
is in a perfect shape now.

A feasible balance is finding a heuristic algorithm that adjusts the testing
frequency based on fail/success rate of user jobs and previous tests.
