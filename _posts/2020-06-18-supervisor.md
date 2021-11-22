---
layout: post
name: supervisor
title: Unicorn-inspired PicoLisp daemon to spawn and manage worker processes
---

You can [get it on GitHub](https://github.com/aw/picolisp-supervisor).

This program mimics functionality of [Unicorn](https://yhbt.net/unicorn/), without being limited to HTTP applications.

The included `supervisor.l` can be used to spawn multiple _worker_ processes which perform tasks in an infinite loop.

![Supervisor](https://user-images.githubusercontent.com/153401/85007713-68f87b80-b14b-11ea-840e-aed11b992a72.png)

  1. [Requirements](#requirements)
  2. [Getting Started](#getting-started)
  3. [Usage](#usage)
  4. [Note and Limitations](#notes-and-limitations)

# Requirements

  * PicoLisp 32-bit/64-bit `v17.12` to `v20.6.29`
  * Linux or UNIX-like OS

# Getting Started

This library is written in pure PicoLisp and contains **no external dependencies**.

To ensure everything works on your system, run the tests first: `make check`

### Try the test app

Launch the Supervisor with: `./supervisor.l --app testapp.l`

You'll see output showing 1 worker _"Performing a task"_ in a loop. Wait until it completes a task or two, then press `Ctrl+C` or send an `INT` signal to tell the parent process to exit, along with its worker.

```
# example output
./supervisor.l --app testapp.l --workers 2
parent process ready pid=814
spawning 2 missing workers:
worker[0] spawning..
worker[1] spawning..
worker[0] spawned pid=815
worker[0] pid=815 do this after forking
worker[0] ready
worker[0] pid=815 Performing a task: sleeping for 12 seconds
worker[1] spawned pid=816
worker[1] pid=816 do this after forking
worker[1] ready
worker[1] pid=816 Performing a task: sleeping for 9 seconds
worker[1] pid=816 Performing a task: sleeping for 16 seconds
^Cworker[1] exited
worker[0] exited
parent exited
```

Feel free to observe the example code in [testapp.l](testapp.l).

# Usage

The supervisor runs in the foreground and simply manages the worker processes. If a worker (child process) exits, it will be re-spawned automatically by the supervisor. The supervisor periodically checks for missing workers, depending on the `*SV_POLL_TIMEOUT` variable.

```
# supervisor.l
Usage:                    ./supervisor.l --app <yourapp> [option] [arguments]

Example:                  ./supervisor.l --app app.l --workers 4 --poll 1

Options:
--help                    show this help message and exit

--app <yourapp>           Filename of the app which contains (worker-start)
--poll <seconds>          Number of seconds to poll for missing workers (default: 30)
--preload                 Load the app in the parent before forking the worker process (default: No)
--workers <number>        Number of workers to spawn (default: 1)
```

## Options

* `--app`: This option accepts 1 argument, a PicoLisp `.l` file. It will be `(load)`ed in the forked process of each worker (each time a worker is spawned), unless `--preload` is specified. Once the process is forked, the `(worker-start)` function will be called automatically, so that needs to be defined in your worker app.
* `--preload`: This options takes no arguments. If `--preload` is specified, the worker app will be loaded only once, in the parent process. Its code will be inherited by each worker process as it's spawned. This should be more memory efficient (and faster) for large applications, but prevents changing the worker app "on the fly" (without restarting the supervisor).
* `--poll`: This option accepts 1 argument, the number of seconds where the `parent` process will sleep before checking for any missing workers. For processes which take a long time to complete, or for non-busy servers, it's probably safe to set the polling interval a bit higher (ex: `60` seconds). If there's a need to know almost "right away" when a worker is missing, it is safe to set it to `1`.
* `--workers`: This option accepts 1 argument, the number of workers which should be spawned. The supervisor will remember this number and always ensure that it's maintained. If 3 or 4 workers happen to exit, the supervisor will notice and respawn 3 workers.

# Notes and limitations

This section will explain some important technical details about the code, and limitations on what this app can and can't do.

## Technical notes

  * All global variables are prefixed with `*SV_`, and functions are prefixed with `sv-`.
  * Similar to [Unicorn](https://yhbt.net/unicorn/), there are `(before-fork)` and `(after-fork)` hooks which will be called if they're defined in your app (totally optional). Of course, `(before-fork)` happens in the parent process, right before the child is forked, and `(after-fork)` happens in the child process, right after it's forked.
  * The `(before-fork)` hook will only be called when `--preload` is provided, since there's no way to call the function before the code is even loaded.
  * The unique sequential ID _number_ of the worker process will be sent as the one and only argument to `(before-fork)` and `(after-fork)`. This can be used in the app to conditionally perform tasks based on its ID (ex: worker ID 0 could verify the integrity of a database, while the other workers simply query it).
  * The supervisor is quite verbose, but this is necessary to see the status of what the workers are doing. Standard *NIX tools can be used to redirect output to a log file or `/dev/null` if needed.
  * Every time a worker loops on a task, it checks if the parent is still there. If not, it will exit cleanly on its own. This allows the parent to be stopped with `kill -9` (or `kill -KILL`), and the workers will continue their work and exit cleanly when they're done.
  * Sending a regular kill signal to the parent (ex: `kill -15` or `kill -TERM`) will terminate all workers and the parent immediately.

## Limitations

  * This program was used in production for over 4 years and has been tested extensively under various loads. However it was modified heavily for public release and may contain bugs. Please use at your own risk.
  * This program is not an _exact copy_ of Unicorn, and differs in many areas. It is also missing many features of Unicorn such as extra signal handling and hot code-reloading. There is a plan to add those features in the future.
  * There are other ways to do something similar to `supervisor.l` in PicoLisp, feel free to use those techniques if you prefer.

