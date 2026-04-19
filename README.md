# Multi-Container Runtime — OS-Jackfruit

## 1. Team Information

| Name | SRN |
|------|-----|
| Kushal G | PES1UG24AM145 |
| Kaustubh Akash | PES1UG24AM131 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites
- Ubuntu 22.04 or 24.04 VM
- Secure Boot OFF
- Dependencies: `sudo apt install -y build-essential linux-headers-$(uname -r)`

### Setup

```bash
git clone https://github.com/kushal1310xviii/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
cp cpu_hog memory_hog io_pulse rootfs-alpha/
cp cpu_hog memory_hog io_pulse rootfs-beta/
```

### Build

```bash
sudo make
```

### Load Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
sudo dmesg | tail -3
```

### Start Supervisor (Terminal 1 — keep running)

```bash
sudo ./engine supervisor ./rootfs-base
```

### CLI Usage (Terminal 2)

```bash
# Start a container in background
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 10"

# Start a container and wait for it to finish
sudo ./engine run alpha ./rootfs-alpha "/cpu_hog 10"

# List all containers and their metadata
sudo ./engine ps

# View container logs
sudo ./engine logs alpha

# Stop a running container
sudo ./engine stop alpha
```

### Memory Limit Test

```bash
sudo ./engine start memtest ./rootfs-alpha "/memory_hog 8 500" --soft-mib 20 --hard-mib 40
# Wait 8 seconds
sudo dmesg | grep container_monitor
sudo ./engine ps
```

### Scheduler Experiment

```bash
sudo ./engine start cpu-high ./rootfs-alpha "/cpu_hog 20" --nice 0
sudo ./engine start cpu-low ./rootfs-beta "/cpu_hog 20" --nice 10
```

### Cleanup

```bash
# Press Ctrl+C in Terminal 1 to stop supervisor
sudo umount rootfs-alpha/proc 2>/dev/null
sudo umount rootfs-beta/proc 2>/dev/null
sudo rmmod monitor
sudo dmesg | tail -5
```

---

## 3. Demo Screenshots

### Screenshot 1 — Supervisor started
![Screenshot 1](screenshots/2.png)
*Supervisor started with control socket at /tmp/mini_runtime.sock and /dev/container_monitor device confirmed*

### Screenshot 2 — Container start and metadata
![Screenshot 2](screenshots/4.png)
*Container alpha started with pid=4677, ps showing ID, PID, STATE, SOFT and HARD memory limits*

### Screenshot 3 — Logs and stop
![Screenshot 3](screenshots/5.png)
*logs command showing alpha log output, stop command sending SIGTERM to alpha pid=4677*

### Screenshot 4 — Memory limit enforcement
![Screenshot 4](screenshots/8.png)
*memtest container started, dmesg showing container_monitor registering and tracking containers, ps showing memtest and alpha*

### Screenshot 5 — Scheduler experiment running
![Screenshot 5](screenshots/11.png)
*cpu-high (nice 0) and cpu-low (nice 10) both running simultaneously, ps showing all active containers*

### Screenshot 6 — Scheduler experiment stop
![Screenshot 6](screenshots/13.png)
*SIGTERM sent to cpu-high pid=4726 and cpu-low pid=4732, demonstrating clean container shutdown*

### Screenshot 7 — Clean supervisor teardown
![Screenshot 7](screenshots/15.png)
*Supervisor clean exit, rmmod monitor, dmesg showing Module unloaded*

### Screenshot 8 — Full kernel module history
![Screenshot 8](screenshots/17.png)
*Complete dmesg history showing all container registrations, process exits, and final Module unloaded*

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux namespaces are kernel features that give each process its own
view of system resources. Our runtime uses three namespace types via
`clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`.

**PID namespace** (`CLONE_NEWPID`): Each container sees its own PID
number space starting at PID 1. The host kernel still tracks the real
host PID, but inside the container, the process believes it is PID 1.
This prevents containers from seeing or signaling each other's
processes by PID.

**UTS namespace** (`CLONE_NEWUTS`): Each container gets its own
hostname. We call `sethostname(container_id)` inside the child so
each container identifies itself by its assigned name. The host
hostname is unaffected.

**Mount namespace** (`CLONE_NEWNS`): Each container gets its own
filesystem mount table. We call `chroot()` into the container's
assigned Alpine rootfs directory, then mount `/proc` inside it. This
means the container can only see files under its own rootfs — it
cannot access host files outside that directory.

What the host kernel still shares with all containers: the same
kernel code, kernel memory, network stack (we do not use
`CLONE_NEWNET`), and the same physical CPU and RAM. Namespaces
provide isolation of views, not isolation of resources.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because containers are child
processes — when a child exits, only its parent can reap it with
`wait()`. Without a persistent parent, exited containers become
zombies that hold PID table entries until the parent exits.

Our supervisor uses `clone()` to create each container child with new
namespaces. It installs a `SIGCHLD` handler that calls
`waitpid(-1, WNOHANG)` to reap all exited children immediately
without blocking. For each reaped PID, it updates the container
metadata (state, exit code, exit signal) under a mutex.

The `stop_requested` flag is set before sending `SIGTERM` to a
container. This lets the `SIGCHLD` handler distinguish between a
manually stopped container (`stopped` state) and one killed by the
kernel memory monitor (`killed` state).

### 4.3 IPC, Threads, and Synchronization

Our project uses two IPC paths:

**Path A — Logging (pipes):** Each container's stdout and stderr are
connected to a pipe via `dup2()`. The supervisor reads from the read
end in a dedicated producer thread per container. Producers push
`log_item_t` structs into a shared bounded circular buffer. A single
consumer thread pops items and writes them to per-container log files.

The bounded buffer uses a `pthread_mutex_t` to protect the head,
tail, and count fields. Two `pthread_cond_t` variables (`not_empty`,
`not_full`) implement blocking: producers wait on `not_full` when the
buffer is full, consumers wait on `not_empty` when it is empty. This
prevents busy-waiting and guarantees no data is lost or corrupted.

Without the mutex, two producers could simultaneously read the same
`tail` index and overwrite each other's data. Without condition
variables, producers would busy-spin when the buffer is full, wasting
CPU.

**Path B — Control channel (UNIX domain socket):** CLI client
processes connect to `/tmp/mini_runtime.sock`, send a
`control_request_t` struct, and receive a `control_response_t`. This
is separate from the logging pipes because it carries structured
command/response messages rather than raw byte streams, and it needs
a request-reply pattern rather than one-way data flow.

The container metadata linked list is protected by a separate
`pthread_mutex_t` (`metadata_lock`). This is separate from the buffer
lock to avoid holding both locks simultaneously, which would risk
deadlock.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the amount of physical RAM currently
occupied by a process's pages. It does not measure memory that has
been allocated but not yet touched (lazy allocation), memory mapped
but swapped out, or shared library pages counted multiple times across
processes.

Soft and hard limits serve different purposes. A soft limit is a
warning threshold — the process is still running but the operator is
informed it is approaching its budget. A hard limit is an enforcement
threshold — the process is terminated when it exceeds it. This
two-level design lets operators detect gradual memory growth before it
becomes critical.

Memory enforcement must live in kernel space because a misbehaving
process cannot be trusted to monitor and limit itself. A user-space
monitor could be blocked, killed, or bypassed by the very process it
is trying to limit. The kernel timer runs independently of any user
process and cannot be blocked by container activity.

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS), which assigns CPU
time proportional to each process's weight. The `nice` value controls
weight: nice 0 gets full weight, nice 10 gets approximately 25% of
the weight of nice 0 when competing for CPU.

**Experiment 1 — CPU-bound with different priorities:**
Both `cpu-high` (nice 0) and `cpu-low` (nice 10) ran for exactly 20
seconds wall-clock time. CFS gave `cpu-high` more CPU time slices
per second, meaning it completed more computation per second than
`cpu-low`. On a single-CPU VM, the lower-priority container received
fewer time slices when competing.

**Experiment 2 — CPU-bound vs I/O-bound:**
`cpuwork` (cpu_hog, 15s) and `iowork` (io_pulse, 30 iterations x
100ms sleep) ran concurrently. The I/O-bound process spent most of
its time sleeping between writes, voluntarily yielding the CPU. CFS
rewarded it with high responsiveness. The CPU-bound process received
the CPU during the I/O process sleep periods. This demonstrates CFS
fairness: I/O-bound tasks get low latency, CPU-bound tasks get high
throughput during idle periods.

---

## 5. Design Decisions and Tradeoffs

**Namespace isolation:** We used `chroot()` rather than `pivot_root()`
for filesystem isolation. Tradeoff: `chroot` is simpler but can
theoretically be escaped by a privileged process. `pivot_root` is
more secure but requires unmounting the old root. For a course project
demonstrating isolation concepts, `chroot` is the right call.

**Supervisor architecture:** A single long-running supervisor process
owns all containers. Tradeoff: if the supervisor crashes, all
containers lose their parent. An alternative is a per-container
monitor process. We chose a single supervisor because it simplifies
shared state and is easier to reason about for correctness.

**IPC/logging:** We used a single shared bounded buffer with one
consumer thread rather than per-container buffers. Tradeoff: a slow
log write can delay all containers log flushing. The benefit is
simpler synchronization — one mutex, one set of condition variables,
one thread to join on shutdown.

**Kernel monitor:** We chose a mutex over a spinlock for the monitored
list. Tradeoff: mutexes can sleep, which is not allowed in some
interrupt contexts. However, our timer callback runs in a context
where sleeping is permitted. A spinlock would be faster for very short
critical sections but risks priority inversion. Mutex is the safer
and more readable choice here.

**Scheduling experiments:** We used `nice` values rather than CPU
affinity to demonstrate scheduling differences. Tradeoff: `nice`
affects priority within CFS but both processes can still run on all
CPUs. CPU affinity would give stronger isolation but would not
demonstrate the CFS weight mechanism we wanted to show.

---

## 6. Scheduler Experiment Results

### Experiment 1: CPU-bound containers with different nice values

| Container | Nice Value | Duration | Observation |
|-----------|-----------|----------|-------------|
| cpu-high  | 0         | 20s      | Full CFS weight, more CPU time slices |
| cpu-low   | 10        | 20s      | Reduced CFS weight, fewer time slices |

Both containers ran for the same wall-clock duration because
`cpu_hog` measures elapsed time internally. The difference is in
CPU throughput: cpu-high completed more loop iterations per second
than cpu-low when both were competing for the CPU.

### Experiment 2: CPU-bound vs I/O-bound

| Container | Type      | Duration        | Behavior |
|-----------|-----------|-----------------|----------|
| cpuwork   | CPU-bound | 15s             | Continuous CPU usage |
| iowork    | I/O-bound | ~3s active time | Slept 100ms between iterations |

The I/O-bound process completed all 30 iterations while the CPU-bound
process was still running. CFS gave the I/O process immediate CPU
access each time it woke from sleep because it had accumulated CPU
credit during its sleep periods.

**Conclusion:** Linux CFS achieves fairness by tracking virtual
runtime. Processes that use less CPU accumulate less virtual runtime
and are scheduled sooner. Processes that use more CPU are penalized
by higher virtual runtime but still get their fair share over time.
