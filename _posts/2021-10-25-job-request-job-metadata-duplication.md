---
layout: post
title: Data duplication in JobRequest and JobMetadata
date: 2021-10-25
---

When a new job request is sent to slivka, it creates a new `JobRequest`
object in the database containing information about the service,
parameters, status, runner etc. Then, once the job is started another
object - `JobMetadata` - which contains service, runner, 
working directory, job id and status, is added to the database.

First problem is data duplication. Most of the fields in the
`JobMetadata` document are duplicates of those of the `JobRequest`.
This requires updating both documents if any parameter changes.

Second problem is more subtle. It is treating mongodb as a relational
database. In this case, `JobMetadata` should have a one-to-one relation
with `JobRequest` entry.

Instead, we should use the advantages of document-oriented
approach, remove the `JobMetadata` collection entirely and add
more structure to the `JobRequest` documents. They can be given
a `job` property which will store all non-duplicate fields of the
`JobMetadata` document. This way we get rid of the duplicates
and the relations between the documents.

The new structure of the `JobRequest` is as follows:

~~~json
{
  "service": "<service name>",
  "runner": "<runner name>",
  "inputs": {
    "<parameters>": "<values>"
  },
  "timestamp": "<start time>",
  "completion_time": "<end time>",
  "status": "<job status>",
  "job": {
    "work_dir": "<working directory>",
    "job_id": "<job id>"
  }
}
~~~