---
layout: post
title: HMMER services are now added to slivka-bio
date: 2021-03-18
---

So here they are, five new tools added to slivka-bio services.
They are the commonly used HMMER programs: hmmbuild, hmmalign,
hmmsearch, jackhmmer and phmmer.
They are so far the most extensively documented bioinformatic
tools.
The services passed basic tests and should be ready to go.

Databases
=========

Normally, hmmsearch, jackhmemr and phmmer require the users to
provide the datbase to perform the search on. This would be very
unpractical though for the slivka users to upload such large files
therefore the users will have a choice of one of the pre-existing
database files. This will save a lot of bandwidth and simplify
the usage.

Issues
======

Unfortunately, the new services brought a new issue to
the slivka system. Some of the parameters of the hmmer programs
require other parameters to be set. Currently, slivka does not
support validating parameters that depend on each other.
A temporary solution is to erase the default value of the conflicting
parameters and write the constraints into the parameter description.
Ultimately, we may need to add a feature that allows to define the
relations between parameters and conditionally disable some fields.
