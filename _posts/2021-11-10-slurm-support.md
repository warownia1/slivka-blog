---
layout: post
title: Add support for Slurm workload manager
date: 2021-11-10
---

In the recent update we added support for Slurm workload manager.
A new runner type `SlurmRunner` has been added which uses 
*sbatch*, *squeue* and *scancel* commands to manage jobs with Slurm.

# New Jobs

New jobs which are started in the `submit` method are created by
running `sbatch --output=stdout --error=stderr --parsable`
and passed a command to be run using the standard input stream.
The command is created from the template

~~~bash
#!/usr/bin/env sh
touch started
{cmd}
echo $? > finished
~~~

Batch submission is performed by calling `SlurmRunner#submit`
multiple times for each job. It doesn't perform any optimisation
for batches of jobs which may be slow if there are many jobs
and *sbatch* takes time to execute.

## Batch optimisation

Slightly hacky solution is to run an array job in which each command
is included in a separate branch of an if statement using the value
of a `$SLURM_ARRAY_TASK_ID` environment variable to determine the branch
to execute.
We can use Jinja2 templating to create a script that will be run with
*sbatch*

~~~bash
{%- raw -%}
#!/bin/bash
#SBATCH --array=1-{{ commands|length }}
#SBATCH --output=/dev/null
#SBATCH --error=/dev/null

{% for command in commands %}
if [ $SLURM_ARRAY_TASK_ID -eq {{ loop.index }} ]; then
  mkdir -p {{ command.cwd }}
  pushd {{ command.cwd }}
  srun --output=stdout --error=stderr {{ command.args|to_bash }}
  popd
fi
{% endfor %}
{% endraw %}
~~~

The number of jobs in the array is provided by the length of
the commands list. Standard output and error of the *sbatch* command
doesn't contain any useful information and can be safely redirected
to */dev/null*. Then, for each command in the list of the commands
we create an if branch where we compare the value of
`$SLURM_ARRAY_TASK_ID` to the current loop index (starting from 1).
Inside the *if* block we create a job directory and *pushd* into it,
so that stdout and stderr files are created inside the working directory,
not the directory where *sbatch* was executed. Then, the command
can be finally executed with *srun* redirecting output and error
streams to *stdout* and *stderr* respectively.

# Status check

The status check is performed periodically using
~~~sh
squeue --array --format=%i %t --noheader --states=all --user=<username>
~~~

`--array` ensures that jobs in the array are shown individually;
`--format=%i %t` will format each line as *&lt;job id&gt; &lt;status&gt;*;
`--noheader` removes the table header; `--states=all` forces display
of all job states; `--user=<username>` shows only jobs for the specified
user. The name of the current user can be obtained in the Python code 
using
~~~python
username = pwd.getpwuid(os.getuid()).pw_name
~~~

To avoid bashing *squeue* too often we use the same solution
as with *qstat* where we cache the result for 5 seconds.

# Cancellation

Jobs are cancelled by running `scancel <job_id> ...`.
