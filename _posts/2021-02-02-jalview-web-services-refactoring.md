---
layout: post
title: Refactoring Jalview web services
date: 2021-02-02
---

The goals
=========

In the current state, the web services code is quite entangled.
GUi elements are mixed with job management and server communication.
The first milestone is the separation of concerns and encapsulation.
Each class should be responsible for one task exclusively. By dividing
the code into smaller components one will be able to work with small
pieces of code, one at the time, without the need to understad the whole.
In the end, making changes or creating new services should be the matter
of changing/adding just a few classes and plugging them to the rest of 
the system seamlessly.

This leads us to the next goal which is modularity. The ideal scenario
would be where web services are completely modular and every component is
re-pluggable and can be substituted for a different one without the loss
of functionality. This allows creating services dynamically based solely
on service metadata. Jalview would be able to pick-up new, unknown services
without the need to modify it's source code.

Current Approach
================

Currently I re-implemented several core classes and interfaces to see
how everything connects together.

`JalviewWebServiceI` -- the bridge between Jalview and web services
client such as jabaws of slivka-java-client. It creates an abstraction
layer so that other parts of Jalview web services are not calling
ws clients directly. It exposes methods for obtaining service metadata
and for submitting and monitoring jobs and retrieving results.
Instances will be created by it's corresponding `WebServiceProvider`
A concrete implementation would be similar to `SlivkaWSInstance`
class. 

`WebServiceWorker` -- the class representnig a job or a collection of jobs
submitted to the web service. It provides the means for preparing the
sequences and parameters from the raw data captured from an alignment view.
It then uses `JalviewWebServiceI` instance to poll the job status and retrieve
the results. It can have listeners attached that would listen to job state
changes, log messages and result files. It might be a good idea for the worker
to implement `java.util.concurrent.CompletionStage`

`WSJob` -- class used by `WebServiceWorkerI` to store job information such as
job id, state, server etc. The job objects are passed to the worker's
`startJob` and `pollJob` methods to execute respective actions and update
their state.

The idea is that the the worker performs operations and jobs store state.
This way the jobs can be serialized and deserialized between Jalview restarts
and data is kept separate from the operations.

**Alternative 1.** Get rid of workers entirely and create a job for every
submission instead of a worker. The job class will define `prepare()`
`start()`, `poll()`, `getResult()` and `cancel()` methods. 
The Job should also be capable of containing other jobs as its sub-jobs.
Compound jobs will take place of the workers.

**Alternative 2.** Use workers as static (semi-static) job factory.
There would be a single worker instance per align frame which will be
used to create and perform actions on jobs.
