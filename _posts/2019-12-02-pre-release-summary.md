---
layout: post
title: Pre-release Summary
date: "2019-12-02"
---

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