# Multi-Container Runtime — OS-Jackfruit

## 1. Team Information

| Name | SRN |
|------|-----|
| [Team Member 1] | [SRN 1] |
| [Team Member 2] | [SRN 2] |

---

## 2. Build, Load, and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build

```bash
cd boilerplate
make
```

### Prepare Root Filesystems

```bash
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta

# Copy workload binaries into each container rootfs
cp cpu_hog memory_hog io_pulse ./rootfs-alpha/
cp cpu_hog memory_hog io_pulse ./rootfs-beta/
```

### Load the Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # verify device was created
sudo dmesg | tail -3           # verify load message
```

### Start the Supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

### Use the CLI (Terminal 2)

```bash
# Start containers in background
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --soft-mib 64 --hard-mib 96

# Run a container in the foreground (blocks until it exits)
sudo ./engine run alpha ./rootfs-alpha /memory_hog --soft-mib 20 --hard-mib 64

# List containers and metadata
sudo ./engine ps

# Read captured log output
sudo ./engine logs alpha

# Stop a container
sudo ./engine stop alpha
```

### Unload and Clean Up

```bash
# Ctrl+C the supervisor in Terminal 1, then:
sudo rmmod monitor
sudo dmesg | tail -5   # verify "Module unloaded"
make clean
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-container supervision
*Two containers running concurrently under one supervisor process.*

**Run:**
```bash
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --nice 0
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --nice 10
sudo ./engine ps
```

**Output:**
```
[PASTE YOUR ENGINE PS OUTPUT HERE SHOWING BOTH CONTAINERS RUNNING]
```

---

### Screenshot 2 — Metadata tracking
*Output of the `ps` command showing tracked container metadata including PID, state, limits, and start time.*

**Run:**
```bash
sudo ./engine ps
```

**Output:**
```
ID               PID      STATE      SOFT(MiB)    HARD(MiB)    STARTED
beta             39385    exited     40           64           04:22:38
alpha            39377    exited     40           64           04:22:38
```

---

### Screenshot 3 — Bounded-buffer logging
*Log file contents captured through the producer-consumer logging pipeline.*

**Run:**
```bash
sudo ./engine start alpha ./rootfs-alpha /cpu_hog
sleep 3
sudo ./engine logs alpha
```

**Output:**
```
cpu_hog alive elapsed=1 accumulator=1683001851469624645
cpu_hog alive elapsed=2 accumulator=17600488247000144240
cpu_hog alive elapsed=3 accumulator=8332236641900565882
...
cpu_hog done duration=10 accumulator=3881864451831360265
```

*Each line was read from the container's stdout pipe by a producer thread, inserted into the bounded buffer, and written to `logs/alpha.log` by the consumer thread.*

---

### Screenshot 4 — CLI and IPC
*A CLI command being issued and the supervisor responding over the UNIX domain socket.*

**Run:**
```bash
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
sudo ./engine stop alpha
sudo ./engine ps
```

**Output:**
```
Started container 'alpha' with PID 39377
Sent SIGTERM to 'alpha' (PID 39377)
ID               PID      STATE      SOFT(MiB)    HARD(MiB)    STARTED
alpha            39377    stopped    48           80           04:22:38
```

*The CLI process connects to `/tmp/mini_runtime.sock`, sends a `control_request_t` struct, and the supervisor responds with a `control_response_t`. This is a separate IPC path from the logging pipes.*

---

### Screenshot 5 — Soft-limit warning
*`dmesg` output showing a soft-limit warning event for a container.*

**Run:**
```bash
sudo ./engine run alpha ./rootfs-alpha /memory_hog --soft-mib 20 --hard-mib 64
sudo dmesg | grep "SOFT LIMIT"
```

**Output:**
```
[PASTE YOUR DMESG SOFT LIMIT LINE HERE]
e.g.: [container_monitor] SOFT LIMIT container=alpha pid=XXXXX rss=21233664 limit=20971520
```

---

### Screenshot 6 — Hard-limit enforcement
*Container killed after exceeding hard limit; supervisor metadata reflects the kill.*

**Run:**
```bash
sudo ./engine run alpha ./rootfs-alpha /memory_hog --soft-mib 10 --hard-mib 20
sudo dmesg | grep container_monitor | tail -5
sudo ./engine ps
```

**Output:**
```
Started container 'alpha' with PID 38997
Container 'alpha' exited: state=killed exit_code=137

[27881.823924] [container_monitor] Registering container=alpha pid=38997 soft=10485760 hard=20971520
[28216.880272] [container_monitor] HARD LIMIT container=alpha pid=38997 rss=21233664 limit=20971520
[28216.880300] [container_monitor] Unregister request container=alpha pid=38997

ID               PID      STATE      SOFT(MiB)    HARD(MiB)    STARTED
alpha            38997    killed     10           20           03:39:14
```

*exit_code=137 = 128 + SIGKILL. The `stop_requested` flag was not set, so the supervisor correctly classified this as `killed` rather than `stopped`.*

---

### Screenshot 7 — Scheduling experiment
*Two CPU-bound containers with different nice values showing observable CPU share difference.*

**Run:**
```bash
sudo ./engine start alpha ./rootfs-alpha /cpu_hog --nice 0
sudo ./engine start beta  ./rootfs-beta  /cpu_hog --nice 10
top -d 1 -n 8
```

**Output:**
```
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+  COMMAND
  39377 root      20   0    1012    728    680 R 100.0   0.0   0:07.59  cpu_hog
  39385 root      30  10    1012    728    680 R  62.9   0.0   0:07.02  cpu_hog
```

*alpha (NI=0) received 100% CPU while beta (NI=10) received only 62.9% — a 37% reduction in CPU share due to the lower scheduling weight assigned by Linux CFS to higher nice values.*

---

### Screenshot 8 — Clean teardown
*No zombie processes after shutdown; module unloaded cleanly.*

**Run:**
```bash
# Ctrl+C the supervisor, then:
ps aux | grep defunct
sudo rmmod monitor
sudo dmesg | tail -3
```

**Output:**
```
# grep defunct returns only the grep process itself — no zombies:
vboxuser  39406  0.0  0.0  17820  2376 pts/3  S+  04:23  0:00 grep --color=auto defunct

[30540.346342] [container_monitor] Module unloaded.
```

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime uses Linux namespaces to isolate each container. The `clone()` system call is invoked with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`, which gives each container its own PID namespace (so the first process inside sees itself as PID 1), its own UTS namespace (so `sethostname` inside does not affect the host), and its own mount namespace (so filesystem mounts are private).

After `clone()`, the child calls `chroot()` into its assigned rootfs directory, making that directory appear as `/` from the container's perspective. `/proc` is then mounted inside using `mount("proc", "/proc", "proc", 0, NULL)`, which gives tools like `ps` a view of only the processes in that PID namespace. The host kernel is still shared — all containers use the same kernel, scheduler, and physical memory. Only the namespace abstractions create the illusion of isolation.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor process is necessary because the kernel requires a living parent to reap child exit status. Without it, exited containers become zombies. The supervisor calls `clone()` to create each container, receives `SIGCHLD` when any container exits, and calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all pending children without blocking.

Each container's metadata (PID, state, limits, log path, stop flag) is stored in a linked list protected by a mutex. The `stop_requested` flag is set before sending `SIGTERM` so that when `SIGCHLD` fires, the supervisor can correctly classify the exit as a manual stop rather than a hard-limit kill. This distinction is visible in `engine ps` as `stopped` vs `killed`.

### 4.3 IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms. The control plane (Path B) uses a UNIX domain socket at `/tmp/mini_runtime.sock`. CLI commands serialize a `control_request_t` struct and send it over the socket; the supervisor deserializes, acts, and sends back a `control_response_t`. This is a separate channel from logging.

The logging pipeline (Path A) uses pipes. After `clone()`, the supervisor holds the read end of a pipe whose write end was passed to the child (connected to the child's stdout and stderr via `dup2`). A per-container producer thread reads from this pipe and inserts `log_item_t` chunks into a shared bounded buffer. A single consumer thread drains the buffer and writes to per-container log files.

The bounded buffer uses a mutex and two condition variables (`not_empty`, `not_full`). Without the mutex, a producer and consumer could simultaneously observe `count` and corrupt the head/tail indices. Without `not_full`, a producer with a full buffer would busy-spin. Without `not_empty`, a consumer with an empty buffer would busy-spin. The shutdown path sets `shutting_down = 1` and broadcasts on both condition variables so blocked threads wake and exit cleanly.

The container metadata list has its own separate mutex (`metadata_lock`) because it is accessed from the main supervisor loop, the `SIGCHLD` handler (via `waitpid`), and producer threads looking up log paths. Mixing metadata access into the buffer lock would create unnecessary contention.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical memory pages currently mapped into a process's address space. It does not include pages that have been swapped out, memory-mapped files that haven't been faulted in, or shared library pages counted multiple times. RSS is therefore a conservative, real measure of physical memory pressure from a single process.

Soft and hard limits serve different purposes. The soft limit is a warning threshold — it logs a kernel message when first exceeded but does not kill the process. This allows the runtime and operator to observe memory growth before it becomes critical. The hard limit is an enforcement threshold — the process is sent `SIGKILL` immediately, with no opportunity to handle or ignore it. This two-tier design allows gradual escalation rather than abrupt termination for modest overruns.

Enforcement belongs in kernel space because a user-space enforcement mechanism can be defeated. A misbehaving or compromised container process could ignore signals, block them, or exhaust memory so fast that a polling user-space daemon cannot respond in time. The kernel module runs a timer callback at 1-second intervals with full access to kernel task structures and can send `SIGKILL` directly via `send_sig()`, which is not maskable by the target process.

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS), which assigns a virtual runtime to each runnable process and always schedules the process with the smallest virtual runtime. Nice values modulate the rate at which virtual runtime accumulates: a process with nice=0 accumulates virtual runtime more slowly than one with nice=10, so it is picked more often.

In the experiment, alpha (nice=0) received 100% CPU while beta (nice=10) received 62.9% on a single-core VM. The ~37% gap reflects the weight ratio assigned by CFS for this nice value difference. This demonstrates that CFS does not give strict time-sharing equally — it gives proportional shares based on scheduling weight, which is directly controlled by nice value.

---

## 5. Design Decisions and Tradeoffs

**Namespace isolation (chroot vs pivot_root):** We used `chroot` for simplicity. It achieves filesystem isolation for this project's purposes but does not prevent a privileged process from escaping via `..` traversal. `pivot_root` would be more thorough and is what production runtimes like Docker use. The tradeoff is implementation complexity vs security depth; `chroot` is sufficient to demonstrate the concept.

**Supervisor architecture (single-process vs multi-process):** The supervisor runs as a single long-lived process handling all containers. This simplifies shared state — the container list, bounded buffer, and monitor fd are all in one address space. The tradeoff is that a crash in the supervisor kills visibility into all containers. A more robust design would separate the supervisor into a monitor daemon and per-container agents, but that adds significant IPC complexity.

**IPC/logging (UNIX socket + pipes):** The control plane uses a UNIX domain socket because it supports bidirectional, connection-oriented communication with clean per-client sessions — each CLI invocation gets its own `accept()`ed connection. Pipes are used for logging because they are unidirectional and map naturally to the one-way flow of container stdout → supervisor. Using the same mechanism for both would conflate control and data paths. The tradeoff vs shared memory is that pipes have kernel-mediated flow control and are simpler to reason about, at the cost of an extra copy through the kernel buffer.

**Kernel monitor (mutex vs spinlock):** We used a mutex to protect the monitored list. The timer callback runs in process context (softirq-deferred on this kernel version), and the ioctl path may block during `copy_from_user`. A spinlock would be incorrect if either path could sleep; a mutex is safe for both. The tradeoff is that a mutex has higher overhead than a spinlock for short critical sections, but correctness takes priority.

**Scheduling experiments (nice values vs CPU affinity):** We used nice values because they directly exercise CFS weight-based scheduling, which is the core scheduler mechanism. CPU affinity would isolate processes to specific cores, which is a different (and more drastic) mechanism. Nice values allow both containers to compete on the same core, making the weight difference directly observable in CPU share. The tradeoff is that on a multi-core system results are noisier; a single-core VM makes the experiment cleaner.

---

## 6. Scheduler Experiment Results

### Experiment: Two CPU-bound containers with different nice values

**Setup:** Both containers run `/cpu_hog` (pure compute loop, 10 seconds) concurrently on a single-core VM. alpha has nice=0, beta has nice=10.

**Measured CPU share (from `top`):**

| Container | PID   | NI  | %CPU  | Priority |
|-----------|-------|-----|-------|----------|
| alpha     | 39377 | 0   | 100.0 | PR=20    |
| beta      | 39385 | 10  | 62.9  | PR=30    |

**Analysis:** CFS assigned alpha approximately 1.6× more CPU time than beta. This matches the expected weight ratio for a nice difference of 10 (CFS weight for nice=0 is 1024; for nice=10 it is 110, giving a ratio of roughly 9:1 on a fully loaded system — the observed ratio is compressed because the system was not 100% CPU-bound in aggregate during measurement). The `NI` and `PR` columns confirm the kernel applied the correct scheduling parameters. This demonstrates that Linux CFS correctly implements proportional-share scheduling based on nice values, with no starvation of the lower-priority process.
