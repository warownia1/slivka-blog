---
layout: post
title: Progress Update
---

Issues
------

- Issue: File serving should not be performed by slivka wsgi application
  as it could block workers from a limited pool for a long time
  rendering the entire application unresponsive to new requests.

  Solution: URL location and path to the directory containing 
  uploaded and output files can be specified in the settings.
  This URLs and paths can be set to match the proxy server configuration,
  so it can provide the files directly.

- Issue: Settings file location was not set when the slivka module was imported
  for the first time causing problems with global settings.
  Lazy settings loading had to be implemented to circumvent the issue.

  Solution: Settings proxy object deferrs the variables initialization until any
  parameter is accessed for the first time. Then settings variables are initialized
  from the file specified in `SLIVKA_SETTINGS` environment variable.

- Issue: Communication with the local queue was error prone.
  The message encoded its header and the content length in the first 16 bytes.
  Once the socket was returned as readable by `select`, the script immediately tried
  to read the first 16 bytes + the rest of the content. If it was not available
  immediately, it raised a BlockingIOError. Current solution required the message
  to be received in chunks and buffered before being processed increasing the
  complexity even further.

  Solution: Using the asynchronous sockets provided by `asyncio` library
  allowed for more readable and linear code with context switching.
  Using redis as a message broker was an alternative, but it would introduce
  additional and heavyweight dependency. Finally, asyncio was combined with
  lightweight ZMQ messaging system which allowed for simple and robust code.

- Issue: Variable placeholders in command definitions were mixing `$var`
  and `{var}` syntax.

  Solution: The syntax was changed to `${var}` or `$var` to mimic bash variables
  referencing. Also, environment variables can be used in environment
  configuration as well as in command definition. Environemnt variable substitution
  occurrs before the value interpolation to avoid injections.

- Issue: New Runner instances were created for each job and stored the job's
  metadata making it more difficult to restore the jobs after system restart.

  Solution: Runners are instantiated for each configuration on startup and
  are separate from the job. Additional abstraction makes implementing
  custom runners easier.

- Issue: Concurrent access to the SQLite database caused OperationalError
  and database locking. Splitting transactions into smaller pieces did not
  solve the issue. Considered solutions involved:
  - Using redis for cache and messages and SQLite for persistent storage.
    Redis would be a heavyweight dependency.
    Synchronization between redis and SQLite would potentially be a problem.
  - Using ZMQ for messaging and SQLite for persistent storage.
    Every request would go through the scheduler which would be a bottleneck.
    Additionally, if we wanted to parallelise access to the database we would run
    into the same problem as before.
  - Writing data to a separate json file for each job would allow simple
    read-only access parallelization at the cost of searching and indexing.
    New requests or results could not be easily discovered and
    required additional inter-process messaging system.

  Solution: Using MongoDB is a lightweight(ish) alternative to SQLite
  which stores data as JSON objects that nicely map to Python data structures.
  Accessing multiple documents concurrently is no longer an issue.


Test Coverage
-------------
- Form fields are all covered except FileField which requires additional database and filesystem
  mocking.
- Form validation and saving is covered except input files.
- Runner base, job submission delegation and command line contruction, is fully covered

Needs testing
-------------
- File field needs to have been tested with mock uploaded file and mock job results
- Form saving with input files needs to be tested
- Scheduler has currently no unit tests
- Server app has no unit tests
- Integration tests between scheduler and runners are needed.
