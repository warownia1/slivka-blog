---
layout: post
title: Extracting runner logic to an interface.
date: 2021-12-02
---

In order to improve code clarity and testability and to make writing
custom runners easier, we split the routines dealing with
command preparation from the workload manager interface.

Achieved goals:
 - command arguments and environment can be tested without
   creating stub subclasses;
 - a simple mock object can be used as a runner to test calls;
 - all internal parameters are removed from the runner constructor;
 - runners can be tested in isolation without the need for the
   full command configuration;
 - there is a clear and simple interface which needs to be subclassed
   when creating a new runner.

# Implementation

All the methods responsible for parsing the command line arguments and
creating job environment are moved to a new *CommandStarter* class.
These objects will work universally for all command runners and will
delegate calls that need action from the underlying execution system
to the attached command runner.

The old *Runner* is stripped off of all methods that doesn't involve
an external execution system and the remaining are changed to purely abstract
methods. Also, single job method variants, which were never used, are
removed in favour of using batch variants only.
Here is the new command runner class that should be extended by
concrete runner implementations.

~~~python
Command = namedtuple("Command", "args, cwd, env")
Job = namedtuple("Job", "id, cwd")

class BaseCommandRunner(ABC):
  @abstractmethod
  def start(self, commands: List[Command]) -> List[Job]:
    ...

  @abstractmethod
  def status(self, jobs: List[Job]) -> List[JobStatus]:
    ...

  @abstractmethod
  def cancel(self, jobs: List[Job]):
    ...
~~~
