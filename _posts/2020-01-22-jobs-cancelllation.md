---
layout: post
title: Requesting job cancellation
date: "2020-01-22"
---

Job cancellation is already implemented on the runner side.
The `Runner` interface required job cancellation method which
is overridden in each runner subclass.
Now the scheduler needs to call the `cancel` method for the
cancelled jobs.

## Synchronous approach
The scheduler will search the database for any cancellation request(`JobState.CANCEL`) in its main loop before the call to `update_running_jobs`.
For each job with `CANCEL` state, the scheduler requests the runner to cancel the job and changes its state to `CANCELLING` to avoid multiple calls to `cancel`.
When the job is fully cancelled by the Runner, it should respond with the job status `INTERRUPTED`.

## Asynchronous approach
The scheduler may wait on a separate thread for any signals or socket communication from the server to cancel the job.
This approach, however, requires adding synchronous communication between the web server and the scheduler (cancel request needs to be acknowledged by the scheduler).
It could create a bottleneck where the multi-threaded server will be limited by the single-threaded scheduler's speed.
