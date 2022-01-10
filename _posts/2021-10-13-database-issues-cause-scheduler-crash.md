---
layout: post
title: Database connection issues crashes the scheduler
date: 2021-10-13
---

It is time to address an issue which was present since the beginning
namely, handling database connection errors. At the moment, all
database exceptions were unhandled and caused the entire application
to crash.

The question is at which point should the exception be
captured and the request be retried. If the try/except clause
is too broad, then the program would not only retry the database
transaction, but also all the code contained in the block.
This is not a big deal for data retrieval, but it becomes a problem
when the data is pushed after job submission. Database error would
cause job duplicates to be repeatedly created until the error is
resolved. On the other hand, making the try/except scope too narrow
will negatively affect readability.

A simple helper function that can be used as a wrapper to the unsafe
calls which retries the calls until successful can be used on each
database-related subroutine.

~~~python
def retry_call(f, exception=Exception, handler=None):
  while True:
    try:
      return f()
    except exceptions as e:
      if handler and handler(e):
        raise
~~~

For now, we wrapped every single database call in a wrapper function
which is then passed to the retry_call function. This way, if any
database exception occurs, only the call which caused it will be
repeated. Unfortunately, it greatly reduced core readability and
needs further improvements.
