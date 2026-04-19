# Multi-Container Runtime with Kernel Memory Monitor

## Team Information
- **Name:** Sanand Kumar Shetty  
  **SRN:** PES2UG24CS438  

- **Name:** Shiva Sandesh  
  **SRN:** PES2UG24CS469  

---

## Project Overview

This project implements a lightweight Linux container runtime in C, combined with a kernel-space memory monitor. The system supports running multiple containers simultaneously under a centralized supervisor while enforcing memory constraints at the kernel level.

The design demonstrates key operating system concepts including process isolation, inter-process communication (IPC), synchronization, scheduling behavior, and kernel-user space interaction.

---

## System Architecture

The project consists of two major components:

### 1. User-Space Runtime and Supervisor (`engine.c`)

* Acts as a long-running parent process
* Manages lifecycle of multiple containers
* Provides a CLI interface for interaction
* Maintains metadata for all containers
* Handles logging through a bounded-buffer mechanism
* Communicates with the kernel module using `ioctl`

### 2. Kernel-Space Memory Monitor (`monitor.c`)

* Implemented as a Linux Kernel Module (LKM)
* Tracks container processes using their host PIDs
* Enforces:

  * **Soft memory limits** (warning)
  * **Hard memory limits** (process termination)
* Maintains process tracking using a kernel linked list
* Ensures thread-safe access using kernel synchronization primitives

---

## Features

* Multi-container execution with isolation
* Namespace-based process and filesystem separation
* Supervisor-based container lifecycle management
* CLI-driven interaction model
* Dual IPC design:

  * Control channel (CLI ↔ Supervisor)
  * Logging channel (Container → Supervisor)
* Concurrent logging using producer-consumer model
* Kernel-enforced memory limits (soft & hard)
* Scheduling experiments for performance analysis
* Proper resource cleanup (no zombies, no leaks)

---

## Container Isolation

Each container runs with:

* **PID namespace** – isolated process IDs
* **UTS namespace** – isolated hostname
* **Mount namespace** – isolated filesystem
* **Separate root filesystem** using `chroot`

Each container operates independently with its own writable root filesystem derived from a base image.

---

## CLI Interface

The runtime provides the following commands:

```bash
engine supervisor <base-rootfs>
engine start <id> <container-rootfs> <command> [options]
engine run   <id> <container-rootfs> <command> [options]
engine ps
engine logs <id>
engine stop <id>
```

### Command Behavior

* `supervisor` → Starts the container manager daemon
* `start` → Launches a container in background
* `run` → Launches container and waits for completion
* `ps` → Displays container metadata
* `logs` → Shows container output logs
* `stop` → Gracefully stops a running container

### Optional Flags

* `--soft-mib N` → Soft memory limit (default: 40 MiB)
* `--hard-mib N` → Hard memory limit (default: 64 MiB)
* `--nice N` → Adjust scheduling priority

---

## Logging System

The project uses a **bounded-buffer logging design**:

* Containers write output to pipes
* Supervisor reads via producer threads
* Data is stored in a shared buffer
* Consumer threads write logs to files

### Guarantees

* No log loss during abrupt termination
* No deadlocks under high load
* Proper synchronization between threads
* Clean shutdown with buffer flush

---

## Memory Monitoring

The kernel module enforces memory policies:

* **Soft Limit**

  * Logs warning when exceeded
* **Hard Limit**

  * Terminates process (SIGKILL)

### Tracking

* Processes registered via `ioctl`
* Stored in kernel linked list
* Periodically monitored using RSS values

### Termination Classification

* Normal exit
* Manually stopped
* Killed due to hard limit

---

## Scheduling Experiments

The runtime is used to study Linux scheduling behavior:

* CPU-bound vs I/O-bound workloads
* Effect of `nice` values on execution
* Comparison of execution times and responsiveness

### Observations

* Higher priority processes receive more CPU time
* I/O-bound processes remain responsive
* Scheduler balances fairness and throughput

---

## Build and Run

### 1. Build the Project

```bash
make
```

### 2. Load Kernel Module

```bash
sudo insmod monitor.ko
```

### 3. Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

### 4. Create Container Filesystems

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### 5. Start Containers

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta ./rootfs-beta /bin/sh
```

### 6. Check Containers

```bash
sudo ./engine ps
```

### 7. View Logs

```bash
sudo ./engine logs alpha
```

### 8. Stop Containers

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

### 9. Unload Kernel Module

```bash
sudo rmmod monitor
```

---

## Engineering Analysis

### 1. Isolation Mechanisms

The system uses Linux namespaces to isolate containers:

* PID namespace isolates process trees
* Mount namespace isolates filesystem view
* UTS namespace isolates hostname

However, all containers share the same kernel, making them lightweight compared to virtual machines.

---

### 2. Supervisor and Process Lifecycle

A persistent supervisor:

* Tracks all containers centrally
* Handles process creation using `clone()`
* Reaps child processes using `SIGCHLD`
* Maintains container metadata

This prevents zombie processes and enables controlled lifecycle management.

---

### 3. IPC and Synchronization

Two IPC mechanisms are used:

* **Control IPC** → CLI ↔ Supervisor
* **Logging IPC** → Containers → Supervisor

Synchronization is handled using:

* Mutexes for shared data
* Condition variables for buffer coordination

This prevents:

* Race conditions
* Data corruption
* Deadlocks

---

### 4. Memory Management

* RSS (Resident Set Size) measures physical memory usage
* Soft limit provides early warning
* Hard limit enforces strict control

Kernel-level enforcement ensures:

* Accurate tracking
* Immediate action
* Protection against user-space bypass

---

### 5. Scheduling Behavior

Experiments demonstrate:

* Priority-based CPU allocation using `nice`
* Fair scheduling across processes
* Tradeoff between responsiveness and throughput

---

## Design Decisions and Tradeoffs

| Component      | Decision                            | Tradeoff                                 |
| -------------- | ----------------------------------- | ---------------------------------------- |
| Isolation      | Used `chroot`                       | Easier but less secure than `pivot_root` |
| Supervisor     | Single central process              | Simpler but potential bottleneck         |
| Logging        | Bounded buffer                      | Requires careful synchronization         |
| IPC            | Separate control & logging channels | More complexity but cleaner design       |
| Memory Control | Kernel module                       | Higher complexity but better enforcement |

---

## Conclusion

This project provides a practical implementation of core OS concepts including containerization, memory management, scheduling, and IPC. It demonstrates how user-space and kernel-space components can work together to build a controlled and observable execution environment
