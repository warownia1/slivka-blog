---
layout: post
title: Automated tests
date: 2021-12-13
---

As the jobs are being executed, the scheduler checks for any
exceptions or other problems with runners and writes the current
service status to the database to be available through the api.
However, if the service is not used frequently the reported status
becomes stale. Additionally, some administrators may prefer a more
systematic tests for their services than reporting the result of
the last run.

This update is intended to address those issues by introducing
automatic, periodic service tests. For that we create a new thread
started along with the scheduler's main thread which runs the
tests with all available runners at set intervals. The run result
is then used to update the service state in the database.

The tests are defined in the service configuration file. For that,
we added a new *tests* section containing an array of test objects
each consisting of the mapping of parameters used in the test and
timeout for the test. Each test defined in the the array is applied
to each runner for that service.
Here is a simplified schema for the tests definition:
~~~json
"tests": [
  {
    "parameters": {
      "<param>": "<value>",
      "<param>": "<value>",
      "<param>": "<value>"
    },
    "timeout": 10
  }
]
~~~
Contrary to the form fields, the values specified in the tests
are passed directly to the command arguments builder and does not
pass through any parsers.
