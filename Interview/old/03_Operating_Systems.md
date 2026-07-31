# Operating Systems (OS) — Conceptual Interview Guide (HPE)

> OS is the topic where interviewers dig into your fundamentals: processes, scheduling, synchronization, deadlocks, and memory. Read for understanding, then drill the Q&A bank. The starred topics (⭐) are the ones asked most often.

---

## 1. What is an Operating System?

**One-line answer:** An operating system is system software that acts as an intermediary between the user/applications and the computer hardware, managing resources and providing services.

**Main functions (name these):**
- **Process management** — creating, scheduling, and terminating processes.
- **Memory management** — allocating and freeing memory, virtual memory.
- **File system management** — storing, organizing, and accessing files.
- **Device/I/O management** — managing hardware devices via drivers.
- **Security & protection** — access control, user authentication.
- **CPU scheduling** — deciding which process runs when.

**Kernel** — the core of the OS that runs with full privileges and directly manages hardware. Types: monolithic (everything in kernel space — Linux) vs microkernel (minimal kernel, services in user space).

**User mode vs Kernel mode:** The CPU runs in two modes. User mode is restricted (applications run here). Kernel mode has full hardware access (OS code runs here). Switching from user to kernel mode happens via a **system call**.

**System call** — the interface through which a program requests a service from the OS kernel (e.g., reading a file, creating a process). Examples: `fork()`, `read()`, `write()`, `exec()`.

---

## 2. Process vs Thread ⭐ (most common OS question)

- **Process** — a program in execution; an independent unit with its own memory space (code, data, heap, stack), resources, and a Process Control Block (PCB).
- **Thread** — a lightweight unit of execution *within* a process. Threads of the same process share the process's memory (code, data, heap) but have their own stack and registers.

| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Own separate memory | Shares process memory |
| Creation cost | Heavy | Lightweight |
| Communication | IPC needed (slow) | Shared memory (fast) |
| Crash impact | Isolated | Can crash whole process |
| Context switch | Expensive | Cheaper |

**Multithreading benefits:** responsiveness, resource sharing, parallelism on multi-core CPUs, economy.

- **Program vs Process:** A program is a passive set of instructions on disk. A process is a program in active execution with a state.

---

## 3. Process States & PCB

**Process states (the lifecycle):**
- **New** — process is being created.
- **Ready** — waiting to be assigned to a CPU.
- **Running** — instructions are being executed.
- **Waiting/Blocked** — waiting for an event (like I/O).
- **Terminated** — finished execution.

**PCB (Process Control Block)** — a data structure the OS keeps for each process, storing: process ID (PID), process state, program counter, CPU registers, memory limits, scheduling info, and open files. It's the "identity card" of a process.

**Context Switch** — saving the state (PCB) of the currently running process and loading the state of the next process so the CPU can switch between them. It's overhead (no useful work happens during it) but enables multitasking.

---

## 4. CPU Scheduling ⭐

**Why:** In multiprogramming, many processes are ready; the scheduler decides which runs next to maximize CPU utilization.

**Key metrics:**
- **Burst time** — CPU time a process needs.
- **Arrival time** — when a process enters the ready queue.
- **Waiting time** — time spent waiting in the ready queue.
- **Turnaround time** — total time from arrival to completion (= waiting + burst).
- **Response time** — time from arrival to first CPU response.
- **Throughput** — processes completed per unit time.

**Preemptive vs Non-preemptive:**
- **Preemptive** — the OS can forcibly take the CPU from a running process (e.g., Round Robin). Better responsiveness.
- **Non-preemptive** — a process keeps the CPU until it finishes or blocks (e.g., FCFS, SJF non-preemptive).

**Scheduling algorithms:**

| Algorithm | How it works | Notes |
|-----------|-------------|-------|
| **FCFS** (First Come First Serve) | Runs in arrival order | Simple; suffers *convoy effect* (short jobs wait behind long ones) |
| **SJF** (Shortest Job First) | Shortest burst runs first | Optimal average waiting time; can starve long jobs; needs burst prediction |
| **SRTF** (Shortest Remaining Time First) | Preemptive SJF | Even better avg wait; more overhead |
| **Round Robin (RR)** | Each process gets a fixed time quantum in turn | Fair, good for time-sharing; performance depends on quantum size |
| **Priority Scheduling** | Highest priority runs first | Can cause *starvation*; solved by *aging* |

- **Starvation** — a low-priority process waits indefinitely. **Aging** — gradually increasing the priority of waiting processes to prevent starvation.
- **Convoy effect** — in FCFS, short processes stuck behind a long one, hurting throughput.

---

## 5. Concurrency & Synchronization ⭐

**Race condition** — when multiple processes/threads access shared data concurrently and the outcome depends on the timing/order of execution. This causes bugs.

**Critical section** — the part of code that accesses shared resources and must not be executed by more than one process at a time.

**The critical section problem** requires a solution that satisfies three conditions:
1. **Mutual exclusion** — only one process in the critical section at a time.
2. **Progress** — if no one is in the CS, a waiting process should be able to enter.
3. **Bounded waiting** — a process shouldn't wait forever to enter.

**Synchronization tools:**

- **Mutex (Mutual Exclusion lock)** — a lock that allows only one thread into the critical section. It has ownership — only the thread that locked it can unlock it. Binary (locked/unlocked).

- **Semaphore** — an integer variable used for signaling, accessed via `wait()` (P, decrement) and `signal()` (V, increment).
  - **Binary semaphore** — value 0 or 1 (similar to a mutex but no ownership).
  - **Counting semaphore** — can be any non-negative integer; used to control access to a resource with multiple instances.

**Mutex vs Semaphore (common question):**
- A mutex is a *locking* mechanism (one owner, mutual exclusion). A semaphore is a *signaling* mechanism (can allow multiple, no ownership). A mutex is essentially a binary semaphore with ownership.

**Classic synchronization problems** (be able to name them):
- **Producer–Consumer (bounded buffer)** — producers add items, consumers remove them; synchronize a shared buffer.
- **Readers–Writers** — multiple readers OK simultaneously, but writers need exclusive access.
- **Dining Philosophers** — models deadlock and resource sharing with philosophers competing for forks.

---

## 6. Deadlock ⭐ (almost always asked)

**Definition:** A situation where a set of processes are blocked forever because each is holding a resource and waiting for a resource held by another in the set.

**The 4 Coffman conditions (ALL must hold for deadlock):**
1. **Mutual exclusion** — resources can't be shared; only one process uses a resource at a time.
2. **Hold and wait** — a process holds at least one resource while waiting for others.
3. **No preemption** — resources can't be forcibly taken; released only voluntarily.
4. **Circular wait** — a circular chain of processes each waiting for a resource held by the next.

**Handling deadlock (4 strategies):**

1. **Prevention** — ensure at least one Coffman condition can never hold (e.g., request all resources at once to break hold-and-wait; impose ordering on resources to break circular wait).
2. **Avoidance** — dynamically check that granting a resource keeps the system in a *safe state*. Uses the **Banker's Algorithm** (allocates only if the system stays safe — i.e., there's a sequence in which all processes can finish).
3. **Detection & Recovery** — allow deadlocks, detect them (resource-allocation graph / wait-for graph), then recover (kill processes or preempt resources).
4. **Ignore (Ostrich algorithm)** — pretend deadlocks don't happen; used by most general-purpose OSes (Windows, Linux) because deadlocks are rare and handling is costly.

- **Safe state** — a state where the OS can allocate resources to each process in some order and still avoid deadlock.
- **Deadlock vs Starvation:** In deadlock, processes are *permanently* blocked waiting on each other. In starvation, a process *could* run but keeps getting passed over (e.g., low priority).

---

## 7. Memory Management ⭐

**Goal:** manage the allocation of physical memory (RAM) to processes efficiently.

- **Logical (virtual) address** — generated by the CPU/program.
- **Physical address** — actual location in RAM.
- **MMU (Memory Management Unit)** — hardware that translates logical to physical addresses at runtime.

**Contiguous allocation & fragmentation:**
- **Internal fragmentation** — allocated memory is slightly larger than needed; the leftover *inside* a block is wasted.
- **External fragmentation** — enough total free memory exists but it's split into non-contiguous chunks too small to use. Solved by **compaction** or by paging.

**Paging** ⭐
- Physical memory is divided into fixed-size **frames**; logical memory into equal-size **pages**. A **page table** maps pages to frames.
- Eliminates external fragmentation (any free frame can be used).
- Address = page number + offset. The page table gives the frame; combine with offset for the physical address.
- **TLB (Translation Lookaside Buffer)** — a small, fast cache of recent page-table entries to speed up address translation.

**Segmentation**
- Memory divided into variable-size **segments** based on logical divisions (code, stack, data). Each segment has a base and limit.
- Matches the programmer's view but can cause external fragmentation.

**Paging vs Segmentation:** Paging uses fixed-size blocks (avoids external fragmentation, causes internal); segmentation uses variable-size logical blocks (avoids internal, causes external).

---

## 8. Virtual Memory & Paging ⭐

**Virtual memory** — a technique that lets a process use more memory than physically available by keeping only the needed parts in RAM and the rest on disk (in a swap space / page file). Gives the illusion of a large, contiguous address space.

- **Demand paging** — pages are loaded into RAM only when needed (on demand), not all at once.
- **Page fault** — occurs when a process accesses a page that's not currently in RAM. The OS must fetch it from disk (expensive). Steps: trap to OS → find the page on disk → load into a free frame → update page table → restart the instruction.

**Page Replacement Algorithms** (when RAM is full and a new page is needed):

| Algorithm | Rule | Notes |
|-----------|------|-------|
| **FIFO** | Replace the oldest loaded page | Simple; suffers *Belady's anomaly* (more frames can cause more faults) |
| **LRU** (Least Recently Used) | Replace the page unused for the longest time | Good approximation of optimal; costly to track exactly |
| **Optimal (OPT)** | Replace the page not needed for the longest future time | Theoretical benchmark — can't implement (needs future knowledge) |

- **Belady's Anomaly** — the counterintuitive case where increasing the number of frames *increases* page faults (happens with FIFO).

**Thrashing** — a state where the system spends more time swapping pages in and out (paging) than executing processes, because processes don't have enough frames. Performance collapses. Handled by reducing the degree of multiprogramming, or the **working set model** (keep each process's actively used pages in memory).

- **Locality of reference** — programs tend to access a small set of pages repeatedly (spatial and temporal locality); this is why paging and caching work.

---

## 9. Inter-Process Communication (IPC)

Since processes have separate memory, they communicate via IPC:
- **Shared memory** — processes share a region of memory (fast; needs synchronization).
- **Message passing** — processes exchange messages via the kernel (slower; no shared memory needed).
- Other mechanisms: **pipes**, **sockets**, **signals**, **message queues**.

---

## 10. File Systems & I/O (lighter)

- **File system** — organizes and stores files on disk; manages metadata, directories, permissions. Examples: NTFS, ext4, FAT32.
- **File allocation methods:** contiguous, linked, indexed.
- **I/O methods:** programmed I/O, interrupt-driven I/O, **DMA (Direct Memory Access)** — device transfers data to memory directly without CPU involvement for each byte.
- **Interrupt** — a signal to the CPU that an event needs attention, causing it to pause and run an interrupt handler.

---

## 11. Interview Q&A Bank (self-test)

**Q: What is an operating system and its main functions?**
System software that acts as an interface between hardware and the user/applications, managing resources. Main functions: process management, memory management, file management, I/O/device management, security, and CPU scheduling.

**Q: Process vs Thread?**
A process is an independent program in execution with its own memory space. A thread is a lightweight unit within a process that shares the process's memory but has its own stack and registers. Threads are cheaper to create and switch; a process crash is isolated while a thread crash can bring down the whole process.

**Q: What is a context switch?**
Saving the state of the current process (in its PCB) and loading the state of the next process so the CPU can switch between them. It's overhead but enables multitasking.

**Q: What is a PCB?**
Process Control Block — a structure the OS keeps per process storing PID, state, program counter, registers, memory info, scheduling info, and open files.

**Q: What is a race condition?**
When multiple threads/processes access shared data concurrently and the result depends on execution order/timing, causing inconsistent outcomes.

**Q: What is a critical section?**
The code segment that accesses shared resources and must be executed by only one process at a time. Its solution needs mutual exclusion, progress, and bounded waiting.

**Q: Mutex vs Semaphore?**
A mutex is a locking mechanism with ownership — only the locking thread can unlock it, allowing one thread in the critical section. A semaphore is a signaling mechanism (an integer with wait/signal); a counting semaphore can allow multiple accesses and has no ownership.

**Q: What is a deadlock and its four conditions?**
A state where processes are blocked forever, each holding a resource and waiting for another held by the next. The four Coffman conditions (all required): mutual exclusion, hold and wait, no preemption, and circular wait.

**Q: How do you handle deadlocks?**
Prevention (break one condition), avoidance (Banker's algorithm keeping a safe state), detection and recovery, or ignoring it (Ostrich algorithm, used by most general OSes).

**Q: What is the Banker's algorithm?**
A deadlock-avoidance algorithm that grants a resource request only if the system remains in a safe state — i.e., there exists an order in which all processes can complete.

**Q: Deadlock vs Starvation?**
In deadlock, processes wait permanently on each other. In starvation, a process can run but is indefinitely denied resources (e.g., low priority); aging solves starvation.

**Q: FCFS vs SJF vs Round Robin?**
FCFS runs in arrival order (simple, convoy effect). SJF runs the shortest job first (optimal average wait but can starve long jobs). Round Robin gives each process a fixed time quantum in turn (fair, good for time-sharing).

**Q: What is starvation and aging?**
Starvation is a process waiting indefinitely (usually low priority). Aging gradually raises a waiting process's priority to eventually let it run.

**Q: What is paging?**
Dividing physical memory into fixed-size frames and logical memory into equal-size pages, mapped by a page table. It eliminates external fragmentation.

**Q: Paging vs Segmentation?**
Paging uses fixed-size blocks (no external fragmentation, some internal). Segmentation uses variable-size logical segments matching the program structure (no internal fragmentation, but external).

**Q: Internal vs external fragmentation?**
Internal: wasted space inside an allocated block (allocated more than needed). External: enough total free memory, but scattered in non-contiguous chunks too small to use.

**Q: What is virtual memory?**
A technique that lets a process use more memory than physically available by keeping active pages in RAM and the rest on disk, giving the illusion of a large address space. Uses demand paging.

**Q: What is a page fault?**
An event when a process accesses a page not currently in RAM; the OS fetches it from disk, updates the page table, and restarts the instruction.

**Q: What is thrashing?**
When the system spends more time paging (swapping pages in/out) than executing, due to insufficient frames per process. Fixed by reducing multiprogramming or using the working-set model.

**Q: LRU vs FIFO vs Optimal page replacement?**
FIFO replaces the oldest page (simple, can suffer Belady's anomaly). LRU replaces the least recently used (good, practical). Optimal replaces the page not needed longest in the future (theoretical benchmark, unimplementable).

**Q: What is Belady's anomaly?**
The counterintuitive situation where increasing the number of frames increases page faults, seen with FIFO.

**Q: What is a TLB?**
Translation Lookaside Buffer — a fast cache of recent page-table entries to speed up virtual-to-physical address translation.

**Q: User mode vs kernel mode?**
Kernel mode has full hardware access and runs OS code; user mode is restricted and runs applications. Switching to kernel mode happens through a system call.

**Q: What is a system call?**
The interface a program uses to request a service from the OS kernel, like file I/O or process creation (e.g., fork, read, write, exec).

**Q: Preemptive vs non-preemptive scheduling?**
Preemptive scheduling can forcibly take the CPU from a running process (Round Robin). Non-preemptive lets a process keep the CPU until it finishes or blocks (FCFS).

---

### Quick-recall cheat sheet
- Process = own memory; Thread = shared memory, own stack.
- Deadlock needs all 4: mutual exclusion, hold & wait, no preemption, circular wait.
- Handle deadlock: prevent / avoid (Banker's) / detect & recover / ignore.
- Mutex = lock with ownership; Semaphore = signal (counting allows many).
- Paging = fixed frames (kills external frag). Segmentation = variable logical segments.
- Virtual memory = demand paging + swap. Page fault = fetch from disk.
- Thrashing = too much paging, too little work.
- Page replacement: FIFO / LRU / Optimal. Belady's anomaly → FIFO.
