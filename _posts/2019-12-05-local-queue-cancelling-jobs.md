---
layout: post
title: Cancelling Local Queue Jobs
date: "2019-12-05 17:03:53 +0000"
---
Even though the jobs sent to the local queue are meant to be quick and
lightweight, we need to be able to cancel and delete them.
Interrupting jobs can be accomplished in two ways:

- Send `SIGINT` directly to the process corresponding to the running job id
  This approach can be easily fit into the current codebase. However, it poses
  certain problems:
  
  - additional handler to the subprocess needs to be stored and accessible
    to the entire class, so it can be interrupted from any other places.
  - other components may interfere with workers interrupting jobs while 
    being processed
  - cancelling jobs which has not beed started yet would require extra hacking

- Cancel the worker which is currently processing the job or change status directly
  if there is none. This requires changing the codebase so that the a new worker
  is created for each job and disposed right after.

  - this solution is twice slower than the current one
  - it still needs extra checks whether the job was already started or not
    (if the job is running cancel the worker, otherwise just change the state) 
  - worker will be solely responsible for the process which is a good thing


