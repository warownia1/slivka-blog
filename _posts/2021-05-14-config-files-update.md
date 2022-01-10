---
layout: post
title: Configuration files update
date: 2021-05-14
---

With the next version of slivka 0.8 I plan to introduce backwards
incompatible changes that would clean up and organise all changes
that accumulated over the development course. These changes will
affect configuration files and web API.

# Main configuration

The current syntax of the configuration file went through multiple
changes, fields were added and removed in the process and their
logical structure was lost. This change will organise parameters
into more explicit categories.

~~~yaml
version: "0.3"

directory.uploads: ./media/uploads
directory.jobs: ./media/jobs
directory.logs: ./log
directory.services: ./services

server.prefix: slivka
server.host: 0.0.0.0:8000
server.path.uploads: /media/uploads
server.path.jobs: /media/jobs

local_queue.host: 127.0.0.1:3397

mongodb.host: 127.0.0.1:27017
mongodb.username: example_user
mongodb.password: example_password
mongodb.database: example_db
~~~

Organising the parameters into a tree will also be supported.

~~~yaml
version: "0.3"
directory:
  uploads: ./media/uploads
  jobs: ./media/jobs
  logs: ./log
  services: ./services
server:
  prefix: slivka
  host: 0.0.0.:8000
  path:
    uploads: /media/uploads
    jobs: /media/jobs
local_queue:
  host: 127.0.0.1:3397
mongodb:
  host: 127.0.0.1:27017
  username: example_user
  password: example_password
  database: example_db
~~~

# Service configuration

## Service description

The changes to the service configuration file will be more cosmetic.
Most importantly, the top level object will now have *description*,
*author*, *license* and *version* properties

## Parameters

The old *form* property will be replaced by *properties* to better
reflect what it contains. The old name was confusing to people.
In the parameter definitions I opt for less nesting as those parameters
are flattened anyway during parsing. Therefore, the *value* will be
removed altogether and it's properties will be moved one level up.

~~~yaml
<field name>:
  label: <field label>
  description: <field description>
  type: <field type>
  default: <default value>
  min: <minimum>
  max: <maximum>
  ...
~~~

This change also makes it more explicit what parameters are passed
to the field object constructor.

Additionally, *multiple* parameter that was deprecated in version
0.7 will be removed in favour of adding square brackets to the type.

**Suggestion**: extend the *BooleanField* with *if-true* and
*if-false* properties that will be substituted for the field
value.

## Command definition

To avoid name repetition, the *command* parameter will be renamed to
*run* and the *baseCommand* parameter will be renamed to *command*
for clarity. *inputs* will be renamed to *args* since the old name
might be confusing. Following, the *arguments*  property will be
removed due to its redundancy and *outputs* will be moved out
of the scope, as output files are not relevant for running
the command line program. Also, *type* will be removed from 
the argument definition as all values are now internally converter
and stored as strings or array of strings before being passed to the
command line construction functions.

## Output files

The list of output files will be removed from the command definition
and be a top-level property. The syntax will remain unchanged.

## Runners and limiters

As runners and limiters are tightly connected they will be grouped
under the same property *execution*. It will contain *runners* and
*limiter* properties that will work as before. Limiter might need
to be renamed, as this name do not reflect its purpose.

## Tests

There is currently no plan for where the service tests should be placed.
Either in the service definition file or in a separate location along
with the test data. This is to be discussed with the group.


# REST API

Some minor changes will be introduced to the web api as well. They'll
include more information about services and submitted job parameters.
 