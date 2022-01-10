---
layout: post
title: Reporting service health status
date: 2021-04-28
---

Both users and server administrators need to know when the service
is not working properly. Currently, the database stores an entry for
health status for each service and runner. However, updating that
entry to represent the actual state of the service is a bit tricky
in the current implementation of the scheduler.

# Overview

The job's life-cycle as seen be the scheduler can be divided into
two stages.
 - The job request is started by `Runner#start()`
   - A new working directory is created for the job
   - List of arguments is prepared
   - Process execution is delegated to `Runner#submit()`
 - Job status is updated using `Runner#check_status()`
   - Running jobs are fetched from the database
   - Concrete implementation of the `Runner` polls the queuing system
   - Job states are updated in the database.

During those stages multiple errors may arise, especially during
submission and status check as those methods call external processes.
I should be interested in capturing `OSError`s only as they are
related to communication with other programs and processes.
Other exception types should only be caused by code bugs and should
not be consideres as malfunctioning service but fatal errors.

Possible problems during `start()`:
 - file not found - the executable does not exist, method raises
   `FileNotFoundError`
 - permissions error - slivka does not have permission to run,
   method raises `PermissionError`
 - connection problems - could not contact queuing system,
   method raises `ConnectionError`

This part is a bit tricky as the successfully executed
method does not guarantee that the job was started. One example is
Univa Grid Engine where `qsub` is used to run a wrapper
script (the program is started indirectly).
`qsub` command will return successfully even if starting
the actual program failed. No error is detected at this point
phase, but the job finishes immediately with exit > 125.
It creates a false message that the program started properly.

Possible problems during `check_status()` are similar to the above
 - connection problems
 - child proces broken - method may raise `ChildProcessError`

The "tricky part" mentioned before can be detected here by checking
the job's exit code. If it is 127 or 128 it may indicate that the
job was not started.

# Updating service health state

Let's ignore the problem of jobs started indirectly for now.
Currently, the service state is updated when `start()` raises
an exception or returns successfully as this is where the majority
of failures occur. Once the program starts it'll likely work properly.
However, if any error occurs during status check it will be then 
overriden by a subsequent successful start and vice versa.
Errors occurring in different stages should be tracked separately
and the most severe is the actual representation of the service health.

This requires having a separate system to keep track of the reported
errors and update the database accordingly.

From the scheduler's perspective, any issues with starting the
program should cause `start()` to fail. Practicaly, it might be
difficult to accomplish as it would require waiting for the process
for some arbitrary amount of time and checking it's exit code.