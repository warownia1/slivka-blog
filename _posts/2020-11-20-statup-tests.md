---
layout: post
title: Testing services on startup
date: "2020-11-20"
---

Following the previous post, in addition to having real-time state
of the services, there is also a need for running all service tests
on startup in order to catch issues as early as possible.

Test syntax and location
------------------------

Since each test is specific to its corresponding service and in most
cases is identical for each runner, it's most intuitive to place it
alongside the service definition file.

The test consists of the input values which will be passed to the
service and the expected output values. Additionally, a timeout
can be specified to indicate the time after which the test should
be cancelled. A sample test definition may look like this:

~~~yaml
inputs:
  input: $SLIVKA_HOME/testdata/sequences.fa
  dealign: yes
  iterations: 1
output-files:
  - path: output.txt
    match-file: $SLIVKA_HOME/testdata/clustalo-output.fa
  - path: stat.log
  - path: stderr
timeout: 10
~~~

Running tests
-------------

The tests should be run by the scheduler class as it has a full access
to all the runners available. It should be also useful to have a
`Scheduler#set_test_interval` method which schedules tests for repeated
execution in background.

