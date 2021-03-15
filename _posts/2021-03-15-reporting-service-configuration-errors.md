---
layout: post
title: Service configuration errors reporting
date: 2021-03-15
---

A little update on error reporting of configuration files.

As five new services (namely HMMER tools) are coming in the next
slivka-bio release, there is a need for a more explicit configuration
errors reporting. So far, I relied on the ``jsonschema.validate``
method to throw appropriate exception and meaningful message
if there are any syntax problems with the configuration file.
However, those messages are very technical and mostly meaningless
for someone who writes the config files and have no idea how the
file is being further processed. Additionally, jsonschema have no
idea where the validated object is coming from to properly report
the file where the error occurred.

The improvement is to catch ``ValidationError`` as soon as possible
and wrap it in another exception that can provide additional
information where the misconfigured field is. As the exception
propagates up the call stack we can report the file and the position
where the issue occurred and throw a more informative error
containing the file and the location of the error.

This approach is currently added to the service form configuration
but can be easily used with other parts of the config file.

