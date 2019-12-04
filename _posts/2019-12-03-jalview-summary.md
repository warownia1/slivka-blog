---
layout: post
title: Jalview Interoperability - Summary
---

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