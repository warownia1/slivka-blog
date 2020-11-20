---
layout: post
title: Monitoring service state
date: "2020-11-16"
---

Each service state needs to be monitored so that the current state can be checked by the front end users. We currently added a `slivka.db.documents.ServiceState` class which is stored in the database and represents the current state of each service and runner.

The issue we are currently facing is how and when to update the entry in the database.

Current approach
----------------

The service status is based on whether the job was successfully submitted to the queuing system. Each failed submission (which raised an exception) increments the counter and, when the limit is reached, the runner is marked as unoperational.

However, the state ofthe service is not based solely on the job submission, as there are other issues which my become apparent during execution even if the job was submitted successfully.

Problem overview
----------------

The status stored in the database should reflect

 - queue submission errors e.g.
    - server submission errors
    - qsub access issues (missing command, wrong permissions)
 - job status check e.g.
    - error contacting server
    - qstat issues (missing command, wrong permissions)
 - job execution error e.g.
    - lack of +x permission
    - command not found

The first two are indicated by the exception raised by `Runner#submit`
and `Runner#check_status` respectively. The third one will be more
difficult to capture as the missing executable results in the non 0
return code from the shell, which is no different than the job completing
unsuccessfully. One option is to check for 127 exit code, but it might
yield inconsistent results on different platforms and shells. 

Additional problem comes with storing the status data in the database.
There is one entry for each service and runner and potentially successful
job submission will overwrite failure message from previous job execution.

