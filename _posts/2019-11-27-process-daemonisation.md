---
layout: post
title: Process Daemonisation
---

The schedueler, server and the local queue should be damonisable so they can be run
by init.d scripts as unix daemons. Gunicorn includes daemonisation out-of-the-box
with its `--daemon` flag which can be passed to `execvp`. Similarly uWSGI has
`--daemonize` flag. In case of the scheduler and the local queue, daemonisation
must be handled by the Python script.

Each cli entry will accept additional `--daemon` and `--pid-file` options
to start as a daemon process. If daemon flag is set, the process will fork, and
exit from the parent process. Then the new process starts a new session and
detaches from the terminal input and output.