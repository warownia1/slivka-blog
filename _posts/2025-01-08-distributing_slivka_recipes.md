---
layout: post
title: Distributing slivka recipes
---

One of the tasks that remains to be solved is creating the system for distributing slivka recipes to be reused by others.
Currently, the entire project including services and configurations is distributed on github at bartongoup/slivka-bio.
This is quite practical for replicating the configuration used on Dundee Resources server, but not useful if you want to install just a few services or merge slivka-bio into an existing slivka project. However, I'd like to avoid creating the entire distribution platform and rely on existing solutions.

## Outline the problem

The first thing is to identify what data is universal to all users and which are specific to the system running slivka. If we want service definitions to be reusable, they should ideally only contain data that is not specific to the system.

The data containet in the slivka-bio files can be roughly divided into the following groups:

 - service definition files
   - service metadata (portable)
   - program inputs, outputs and command line arguments (portable)
   - presets (likely portable)
   - environment variables (usually not portable)
   - runners (not portable)
   - test data (portable)
 - executable scripts and binaries
   - processing scripts (portable)
   - compiled binaries and wrappers (not portable)
   - auxiliary data files used by some executables (portable)
 - top-level settings (not portable)
 - http server files
   - *wsgi* application entry script (no need to distribute)
   - *openapi* and *redoc* static files (no need to distribute)

From that list we can tell that service metadata, inputs, outputs, command line parameters of the tool can be safely distributed an reused in different system configurations as long as the version of the tool matches. Presets and test data can be provided with the service definition too but should allow for customization. The same goes for the auxiliary files. Most users may go with the defaults, but custom presets, tests and adding auxiliary data files must be honoured.

On the other hand, runners are very specific to the platform where slivka is installed and should not be a part of the service definition. Instead, they should be configured on each system individually according to the available resources, tools and the system architecture.

The binaries executed by the services should match the service definition in terms of accepted arguments and version, however, compiled files are not usually transferrable between different system and their location may differ. Instead of a path to the executable, we may use recipes that automatically install the right version of the tool on the target system. It could be a conda environment, a bash installer or a makefile, or maybe even a docker file.
Having separate environments for each tool also allows hosting multiple versions of the same tool without name collisions.
A much simpler solution would be to provide a default command that assumes the binary is on the _PATH_ that can be overriden in a more specific config file.

## Todo

 - Group together service definitions and their auxiliary files and separate services from each other such that each can be installed and used separately.
 - Provide installation procedures for binaries. It can be an exported conda environment file, a docker file or a bash script or a makefile installer.
 - Install programs in separate environments, that way multiple versions of the same tools can be used simultaneously.
 - Do not provide runner configurations with the service definition. The configurations the service is run with should be given from the outside instead of being dictated by the service. If backwards compatibility is essential, the *execution* section of the config can be a template or the import of an external file.
 - Do not provide top-level settings. Those should be configured on each system individually.

## New service definition

First, let's rearrange the directory structure and give each service its own directory. The service auxiliary files can be grouped toghether with services into those directories instead of storing them all in the top-level directory. We can also append the version number to each service file name to ensure unique names. This way, the *services* directory can be distributed using e.g. version-control systems without interfering with system-specific configurations.

~~~
project
├── services
│   ├── example
│   │   ├── example-1.0.service.yaml
│   │   ├── example-1.1.service.yaml
│   │   ├── data.json
│   │   └── testdata
│   │       ├── test-input.txt
│   │       └── test-input2.txt
│   └── otherservice
│       └── otherservice-2.0.service.yaml
├── config.yaml
└── wsgi.py
~~~

### Service file

Service definition can be stripped off of all system specific settings and variables.
Here is an example content of the *example-1.1.service.yaml* file. Since the additional files are located alongside the service files and the directory structure above that directory is not granted, it's more portable to have a `$SERVICE_DIR` variable poiting to the directory containing this service file instead of using `$SLIVKA_HOME` variable.

~~~yaml
# services/example/example-1.1.service.yaml
---
slivka-version: "0.9"
name: Example service
description: Service description
author: John Smith
version: "1.1"
license: Apache 2.0

command: example-executable

args:
  input-file:
    arg: --infile $(value)
  opt:
    arg: --opt $(value)
  aux-data:
    arg: --aux $(value)
    default: $SERVICE_DIR/data.json
  cpu-count:
    arg: --cpus $(value)
  # etc.

inputs:
  input-file:
    type: file
    name: Input file
  opt:
    type: text
    name: Text option
  # etc.

outputs:
  std-log:
    path: stdout
    name: Standard log
    media-type: text/plain
  err-log:
    path: stderr
    name: Error log
    media-type: text/plain
  output:
    path: output.txt
    name: Output data
    media-type: text/plain
  # etc.

tests:
  - inputs:
      input-file: $SERVICE_DIR/testdata/test-input.txt
      opt: value1
    timeout: 60
  - inputs:
      input-file: $SERVICE_DIR/testdata/test-input2.txt
      opt: value2
    timeout: 60
~~~

If we simply moved the remaining properties to a separate file, it would look like this:

~~~yaml
---
service: example-1.1

env:
  PATH: $PATH:$HOME/.local/bin
  MYVAR: variable_value

execution:
  runners:
    local-queue:
      type: SlivkaQueueRunner
      consts:
        cpu-count: 1
    slurm:
      type: SlurmRunner
      consts:
        cpu-count: 4
      sbatchargs:
        "--ntasks 1 --cpus-per-task 4"

  selector: scripts.selector.example_selector
...
~~~

### Profiles file

Current usage proved that execution configurations are often repeated across multiple services and it makes sense to define them in a single file and reuse. Currently, it's done by specifying a `!include <filename:path>` directive which includes parts of another yaml into the document. This can be formalized by creating a dedicated `profiles.yaml` file, a similar approach to how nexflow manages sets of configuration attributes (see [Nextflow docs: Config profiles](https://www.nextflow.io/docs/latest/config.html#config-profiles)).
A profile file could contain configuration parameters which are common for multiple services. The `profiles.yaml` file could be located in the top-level directory alongside the main configuration e.g.

~~~
project
├── services
│   └── ...
├── config.yaml
├── profiles.yaml
└── wsgi.py
~~~

~~~yaml
# example of profiles.yaml

profiles:
  local:
    type: SlivkaQueueRunner

  short-queue-4G:
    type: LSFRunner
    parameters:
      bsubargs: -n 1 -q short -R "rusage[mem=4GB]"

  long-queue-16G:
    type: LSFRunner
    parameters:
      bsubargs: -n 1 -q long -R "rusage[mem=16GB]"

  long-queue-gpu-16G:
    type: LSFRunner
    parameters:
      bsubargs: -n 1 -q long -R "rusage[mem=16]" -gpu "num=1"
  # etc.
~~~

### Combine services with profiles

The final bit is a glue between the service files and profiles. The data which is neither portable across systems, nor shareable across multiple services. This include things like environment variables, selector scripts, additional runner or command line options. I'm not sure if those should be split to multiple files, one file per service, or a single files containing all services.

~~~yaml
service: example-1.1

env:
  MYVAR: variable_value

runners:
  local:
    profile: local
    env:
      PATH: $PATH:$HOME/.local/bin
    command: $HOME/.local/bin/example-executable
    selector-options:
      max-file-size: 10kB

  short-queue:
    profile: short-queue-4G
    env:
      PATH: $PATH:/apps
    command: /usr/local/bin/example-executable-1.1
    inputs:
      cpu-count: 2
    selector-options:
      max-file-size: 1000kB

selector: scripts.selectors.example_selector
~~~

Let's explain the files one step at the time. In the first line we tell that this configuration should be applied to the `example-1.1` service.

~~~yaml
service: example-1.1
~~~

After that, we instruct slivka to set environment variable `MYVAR` to `variable_value` for all runners.

~~~yaml
env:
  MYVAR: variable_value
~~~

The configuration of each runner refers to the configuration from the `profiles.yaml` file. In this case, it's instructed to use the `local` profile as a base.

~~~yaml
profile: local
~~~

Next, we can specify environment variables which are specific to this runner only. In can be useful if `PATH`s are different on different cluster nodes or certain variables must be set when running with an execution system. Those environment variables are appended after the variables defined in the top-level of the document so it's possible to define defaults which are overriden by more specific settings.
In this particular case append `$HOME/.local/bin` to the `$PATH` when using the local queue, but `/apps` when using LSF short queue.

~~~yaml
# local runner
env:
  PATH: $PATH:$HOME/.local/bin

# short-queue runner
env:
  PATH: $PATH:/apps
~~~

We can also override the base command which is important if the binaries are installed in different locations on different systems e.g. sending jobs to other compute nodes or running them inside VMs which may have differing mount points. This option overrides the executable name provided in the `example-1.1.service.yaml` file. I decided to move the base command out of the service definition file, because different systems use different binary paths.

~~~yaml
# local runner
command: $HOME/.local/bin/example-executable

# short-queue runner
command: /usr/local/bin/example-executable-1.1
~~~

Optionally, you can provide additional inputs to the command based on the runner selected. A use case for that is when programs expect available resources such as number of cores, gpu or memory to be specified in the command line. The input may be hidden from the front-end user and populated by the runner configuration as demonstrated with the `cpu-count` option that is specified as an argument, but not as a service input in the `example-1.1.service.yaml` file. Here, we explicitly set the value of the `cpu-count` to `2` when the *short-queue* is used, and leave it empty for the *local-queue*

~~~yaml
inputs:
  cpu-count: 2
~~~

If the selector callable is parametrized, you can provide the context options in the runner configuration that the selector may use when choosing the runner. The options depend on the implementation of the selector and are not built into slivka.

~~~yaml
# local runner
selector-options:
  max-file-size: 10kB

# short-queue runner
selector-options:
  max-file-size: 1000kB
~~~

Here is an example selector that would apply the `max-file-size` limit to the file given in the `input-file` parameter and return the first runner which meets the requirement.

~~~python
import os
from slivka.scheduler.scheduler import SelectorContext

def example_selector(inputs, context: SelectorContext):
    size = os.stat(inputs['input-file']).st_size
    for runner_name in context.runners:
        options = context.runner_options.get(runner_name)
        max_size = options.get('max-file-size', "0")
        if max_size.endswith("kB"):
            max_size = int(max_size[:-2]) * 1000
        else:
            max_size = int(max_size)
        if size <= max_size:
            return runner_name
~~~

## Summary

Getting all the system-specific settings out of the service definition file is a must if we want portable and reusable service files. This way, the community can create and share configurations without everyone having to deal with annoying edit conflicts every time there’s an update.

Profiles have already shown how handy they can be. Since the same runner options often get used across multiple services, keeping them in a separate file makes maintenance much easier. It also sets you up for smoother migrations to other execution systems if the need ever comes up.

The main disadvantage of the solution shown above is having related options scatterred across multiple files. The service inputs, outputs and command line arguments sit in one file, but the command itself, environment variables and sometimes runner-specific inputs are in a different file that also references runer profiles from yet another file. i'm afraid that it can badly impact the readability.

Regarding the selector scripts, the main idea of context options is to make them more reusable. One case are selectors that count the number and lengts of sequences in fasta files. Instead of implementing similar selectors for each service that takes fasta sequences, we can have one that is parametrized with `max-sequence-length` and `max-sequence-count`. 
