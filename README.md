# Unix Shell with Job Control

A fully functional Unix shell implementation in C++ built from scratch on Linux, supporting foreground and background job execution, process group management, and POSIX signal handling. Validated for correctness against a reference implementation across 16 structured regression trace files.

---

## Overview

A Unix shell is an interactive command-line interpreter that sits between the user and the operating system — parsing commands, forking child processes, managing their lifecycle, and handling asynchronous signals. This project implements that from the ground up, including the non-trivial parts: signal-safe job control, race condition prevention, and correct process group isolation.

The shell is written in C++ and targets Ubuntu 22.04 / Linux. It was validated trace-by-trace against a reference implementation (`tshref`) to ensure identical behavior across all supported features.

---

## Features

- **Command execution** — parses and executes any program by pathname with full argument support
- **Foreground jobs** — shell waits for foreground process to complete before accepting next command
- **Background jobs** — processes launched with `&` run concurrently without blocking the shell
- **Job control built-ins** — `jobs`, `fg`, `bg`, `quit`
- **Signal forwarding** — `ctrl-c` sends `SIGINT` and `ctrl-z` sends `SIGTSTP` to the entire foreground process group
- **Zombie reaping** — all child processes are properly reaped via `SIGCHLD` handler
- **Process group isolation** — child processes are placed in their own process groups via `setpgid` to prevent signal leakage to the shell itself
- **Race condition prevention** — `SIGCHLD` is blocked between `fork` and `addjob` to prevent the child from being reaped before it is added to the job list
- **Job ID and PID addressing** — jobs can be referenced by either PID or JID (prefixed with `%`)

---

## Implementation

The shell is implemented across the following functions in `tsh.cc`:

| Function | Description |
|---|---|
| `eval` | Parses command line, forks child, manages foreground/background execution and signal blocking |
| `builtin_cmd` | Handles built-in commands: `quit`, `jobs`, `fg`, `bg` |
| `do_bgfg` | Implements `fg` and `bg` — sends `SIGCONT` and transitions job state |
| `waitfg` | Busy-waits for a foreground job to complete using a sleep loop |
| `sigchld_handler` | Reaps terminated and stopped children via `waitpid` with `WNOHANG | WUNTRACED` |
| `sigint_handler` | Forwards `SIGINT` to the foreground process group |
| `sigtstp_handler` | Forwards `SIGTSTP` to the foreground process group |

---

## Key Technical Concepts

**Process forking and exec** — each non-builtin command forks a child process (`fork`) and replaces its image with the target program (`execve`). The parent either waits for it (`waitfg`) or adds it to the background job list.

**Process group isolation** — immediately after forking, the child calls `setpgid(0, 0)` to place itself in a new process group. This ensures that `ctrl-c` and `ctrl-z` only affect the foreground job, not the shell itself.

**Signal blocking to prevent race conditions** — `SIGCHLD` is blocked with `sigprocmask` before `fork` and unblocked after `addjob`. Without this, a fast-exiting child could be reaped by `sigchld_handler` and removed from the job list before the parent has a chance to add it — a classic TOCTOU race condition.

**Signal-safe job reaping** — all child reaping happens in `sigchld_handler` using a single `waitpid` call with `WNOHANG | WUNTRACED`, handling both terminated and stopped children. This avoids the complexity and bugs that arise from splitting reaping logic between the handler and `waitfg`.

**Zombie prevention** — every child process is eventually reaped by the `SIGCHLD` handler, preventing accumulation of zombie processes in the process table.

---

## Build & Run

```bash
# Build
make

# Run the shell
./tsh

# Run against a specific trace file
./sdriver.pl -t trace01.txt -s ./tsh -a "-p"

# Shorthand for running a trace
make test01

# Compare output against reference shell
make rtest01

# Run all traces and check correctness
python3 shellAutograder.py
```

---

## Testing

Correctness was validated against 16 structured trace files of increasing complexity:

| Traces | Coverage |
|---|---|
| 01-03 | Basic command execution and builtins |
| 04-05 | Background jobs and job listing |
| 06-08 | Signal handling (SIGINT, SIGTSTP) |
| 09-10 | Job state transitions (fg/bg) |
| 11-13 | Process group behavior and ps output |
| 14-16 | Full integration — combined job control, signals, and edge cases |

The shell produces output identical to the reference implementation (`tshref`) on all traces. PIDs differ across runs by design.

---

## Project Structure

```
.
├── tsh.cc               # Shell implementation — all core logic
├── helper_routines.cc   # Provided helper functions (job list management, etc.)
├── tshref               # Reference implementation binary for comparison
├── sdriver.pl           # Trace-driven shell testing harness
├── trace{01-16}.txt     # Regression test trace files
├── tshref.out           # Reference output for all traces
├── shellAutograder.py   # Autograder script
└── Makefile
```

---

## Skills Demonstrated

- Systems programming in C++ on Linux
- POSIX process management: `fork`, `execve`, `waitpid`, `setpgid`, `kill`
- Signal handling: `SIGCHLD`, `SIGINT`, `SIGTSTP`, `SIGCONT`
- Race condition identification and prevention using `sigprocmask`
- Process group and job control concepts
- Regression testing against a reference implementation
- Debugging concurrent processes with GDB on Linux
