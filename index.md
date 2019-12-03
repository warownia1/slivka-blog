3/12/19 - Slivka client and Jalview Interoperability 
====================================================

Currently, SlivkaWSDiscoverer iterates through all services and checks for
the classifier matching. Depending on the classifiers, either SlivkaMsaInstance
or SlivkaAnnotationServiceInstance is created.
The functionality can be extended using input and output files information 
to create a more fine-grained groups of services and have better control over them.

Job submission:
---------------
When the job is submitted, SlivkaWSInstance (a superclass of SlivkaMsaInstance and
SlivkaAnnotationServiceInstance) finds the first field with type `FieldType.FILE`.
If present, it builds a fasta file from the passed sequences, uploads the stream
as a file and use it as an input field value.
Next, additional options are passed to the slivka-client form and the job request
is sent to the server.

Isuue: it automatically assumes a single file fasta input
and other file input options are not available at the moment.
It may be solved by constructing web service instances from component blocks
instead of subclassing in a similar way the Slivka forms are the aggregates
of the form fields and all value validation and parsing is delegated to the 
components.

State updates:
--------------
Whenever the `updateStatus` is called, the state check is issued to the client
and the result is inserted into the `WsJob`'s state field.
The `updateJobProgress` method monitors the log and error log files for any
additional content and appends it to the `WsJob`'s job status. It returns
thether a new content was added to the job status. The convention
for slivka-jalview interoperability requires the log files to be labelled
"log" and "error-log" for standard and error log respectively. 

Output retrieval:
-----------------
Since there is no stardardised way to determine which files contain the output
data, jalview relies on the declared media types to determine the data contained
in the file.
The `SlivkaMsaServiceInstance$getAlignmentFor` assumes the output file to have either
"application/clustal" or "application/fasta" media type and reads them as such.
In case of the `SlivkaAnnotationServiceInstance$getAnnotationResult`, the seeked 
types are "application/jalview-annotations" or "application/jalview-features".
If any of those are present, they are parsed and included into the alignment.

2/12/19 - Preparing for release
===============================

We need to implement some functionality which may be needed in the final version
and document the technical detauls and operation of Slivka..

Removing finished jobs from the local queue
-------------------------------------------
The mapping of job ids to the process ids is never garbage collected
and might grow in size indefinitely as the new jobs are added.
Possible solution is to send a "release" signal from the scheduler
to remove the jobs which have been collected and will never be
checked again. There still is a problem when the scheduler "forgets"
to acknowledge that the job has been collected e.g. due to the crash.
This solution can be combined with using a limited-size dictionaries.
When the dict size reaches a certain threshold, the oldest entries will
be deleted. They can be identified by using OrderedDict or by using
the respected insertion order introduced in Python 3.6+.

Classifiers: Managing input and output files  
--------------------------------------------
At the moment, the services are discovered and recognised by their 
"Operation" classifier. Additional classifiers defining the input and output
may be added to create more specific service definitions e.g.
Takes/Input :: Data :: Alignment :: Sequence alignment (protein)
Returns/Output :: Data :: Sequence features :: Protein features 
The client application can then make a decision based on those classifiers
and show, hide or group certain services together.
Slivka distribution for Jalview may also use meaningful field names to
help the client recognise the input fields and pass appropriate files to them.

Suggestion: Modular slivka packages
-----------------------------------
Slivka packages could be shipped individually instead of relying on
collections of multiple tools as git repositories. Each package would
be shipped as a tarball or zip archive containing the configuration
files, metadata (e.g. manifest file) and installation procedure (Makefile).
Manage.py would be invoked with `install <tarball.tar.gz>` and automatically
add the service into the Slivka project.
This would, however, limit configuration reusability, especially when it
comes to the runner configurations which are not transferrable between the machines. 

Suggestion: Each configuration may contain two parts (files). Static data
containing information about command line parameters, input fields, environment
variables and output files. Run configurations describing available runners,
their parameters and limits. 


27/11/19 - Process Daemonisation
================================

The schedueler, server and the local queue should be damonisable so they can be run
by init.d scripts as unix daemons. Gunicorn includes daemonisation out-of-the-box
with its `--daemon` flag which can be passed to `execvp`. Similarly uWSGI has
`--daemonize` flag. In case of the scheduler and the local queue, daemonisation
must be handled by the Python script.

Each cli entry will accept additional `--daemon` and `--pid-file` options
to start as a daemon process. If daemon flag is set, the process will fork, and
exit from the parent process. Then the new process starts a new session and
detaches from the terminal input and output.


Current Progress Update
=======================

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


5/11/19 - Retrying jobs on Failure
==================================

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

