---
layout: post
title: Retrying jobs on Failure
date: "2019-11-05"
---

When the job submission fails, it should not immediately be discarded as failed.
Instead, use a incremental backoff counter to retry at incrementally longer
delays until the job can be safely considered failed.
The counter's `next` method returns the iterations left until the next retry
attempt should be made. If the job failed, counter's `failure` method should
be called right away, otherwawys the job will be considered successful.
The counter also have the attempts limit after which it stops incrementing
and indicates that no further retry attempts should be made.

```python
if counter.next() == 0:
  try:
    # submit requests 
  except:
    counter.failure()
    if counter.give_up:
        for request in requests: request.status = ERROR
  else:
    for request in requests: request.status = QUEUED
else:
  # skip iteration
```
