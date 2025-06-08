---
title: "Understanding the Go Scheduler: How Goroutines Are Managed"
date: 2025-06-08T20:24:58+05:30
description: "Go is known for its simplicity and its powerful built-in concurrency model. At the heart of this concurrency is the concept of goroutines and the  Go scheduler. In this post, we'll explore how the Go scheduler works, what makes goroutines so efficient, and how they are mapped to system threads using Go's N:M scheduling model."
tags: [golang, concurrency, scheduler, os]
---

Go is known for its simplicity and its powerful built-in concurrency model. At
the heart of this concurrency is the concept of __goroutines__ and the 
__Go scheduler__. In this post, we'll explore how the Go scheduler works, what
makes goroutines so efficient, and how they are mapped to system threads using
Go's __N:M scheduling model__.


---

### Goroutines: Lightweight Threads

Goroutines are lightweight threads managed by the Go runtime. They allow
developers to write concurrent programs easily with a simple API:

```go
go func() {
    // concurrent execution
}()
```

You can start a goroutine by prefixing a function call with the go keyword.
Here's an example:
```go
func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
	go say("world")
	say("hello")
}
```

---

### Go Runtime and Binary Structure

When we compile a Go program, the resulting binary consists of two logical
parts:
- __Application code__: Your own Go code
- __Go runtime__: Manages goroutines, memory, system calls, and scheduling

For example, when we use the `go` statement, it gets translated into a call to
the runtime:
```go
runtime.newProc(...)
```

This is where a new goroutine is created. The Go runtime then works with the
__operating system__, which ultimately runs the threads on physical 
__CPU cores__.

---

### Mapping Goroutines to OS Threads

```sql
+------------------+
|  Goroutines (G)  |  <- User space
+------------------+
        ↓
+------------------+
|   OS Threads (M) |  <- Kernel space
+------------------+
        ↓
+------------------+
|  CPU Cores (C)   |  <- Hardware
+------------------+
```

Key points:
- Goroutines live in __user space__
- OS threads (M) live in __kernel space__
- CPUs execute OS threads
- You can have more __goroutines than OS threads, and more OS threads than CPU cores__

---

### N:M Scheduling Model

Go uses an __N:M scheduler__, meaning:
- N goroutines
- M OS threads
- Goroutines are __multiplexed__ over OS threads

##### Why is this useful?
Suppose a goroutine blocks on a system call (e.g., network I/O). The OS thread
it's on also blocks. But with N:M scheduling, the Go scheduler can 
__move other goroutines__ to another available thread so they can continue
running.

---

### Scheduling with Runqueues

To manage goroutines, Go uses runqueues.
- __Global Runqueue__: A shared queue of goroutines
- __Local Runqueue__: Each OS thread (technically, each P or "Processor" in Go)
has a local runqueue of up to __256 goroutines__

Here's a simplified diagram of scheduling:
```sql
      Global Runqueue
     +------------------+
     |  G1, G2, G3, ... |
     +------------------+
              |
              |    +-----------------+
              +--> |  OS Thread M1   | --> CPU Core 1
                   |  Local: G4, G5  |
                   +-----------------+
              |    +-----------------+
              +--> |  OS Thread M2   | --> CPU Core 2
                   |  Local: G6, G7  |
                   +-----------------+

```

The scheduler prefers __local runqueues__ (faster access) but will fall back to
the __global runqueue__ when needed.

---

### GOMAXPROCS and Execution Limits

The `GOMAXPROCS` variable limits the number of operating system threads that can
execute __user-level Go code simultaneously__. There is no limit to the number
of threads that can be blocked in system calls on behalf of Go code; those do
__not__ count against the `GOMAXPROCS` limit.

In other words, `GOMAXPROCS` limits the number of threads that are actively
__executing__ code. Threads blocked in system calls or otherwise idle are not
counted. This means you can have a small number of executing threads and many
others blocked in system calls.

Each executing thread maintains __per-thread state__, including a 
__local runqueue__. But blocked threads don’t need a runqueue, as the goroutines
associated with them can be picked up and run by others. Maintaining state for
idle threads is inefficient both in memory and in execution potential.

---

### Introducing Processors (P)

To solve these inefficiencies, Go introduces a new abstraction: the 
__Processor (P)__.

A `P` is a __heap-allocated structure__ responsible for executing Go code. It is
associated with an OS thread (M) and maintains the __local runqueue__. This
allows the state that was previously maintained per-thread to now be bound to a
processor instead.

The Go scheduler becomes __work-stealing__: if one processor runs out of
goroutines, it can __steal work__ from another processor’s runqueue. This is now
scalable, because:
- You only need to check a __bounded number__ of processors (equal to GOMAXPROCS)
- You don’t check all OS threads (which can be unbounded)

So now the architecture looks like this:
```sql
+---------------------------+
|    Global Runqueue (G)    |
+---------------------------+
              |
         +----+----+----+----+
         |         |         |
     +---v--+   +--v---+   +--v---+
     |  P1  |   |  P2  |   |  P3  | ... (GOMAXPROCS)
     | RunQ |   | RunQ |   | RunQ |
     +--+---+   +--+---+   +--+---+
        |          |          |
     +--v--+    +--v--+    +--v--+
     |  M1 |    |  M2 |    |  M3 |
     +-----+    +-----+    +-----+

```

---

### Fairness and the Convoy Effect

Before moving on, let’s discuss __fairness__ in scheduling.

Imagine a supermarket with a single cashier and a line of customers. Each
customer has items to check out. If one customer has __many items__, the rest
will wait longer. This is known as the __convoy effect__—one long task delays
everyone else.

```lua
+--------+      +--------+      +--------+
| Cust 1 | ---> | Cust 2 | ---> | Cust 3 |
| 20 itm |      |  2 itm |      |  5 itm |
+--------+      +--------+      +--------+
     |
   Cashier (Resource)
```

In FIFO scheduling, this can lead to unfair resource distribution.

---

### Preemption: Breaking the Convoy

To combat this, we use __preemption__—temporarily stopping a task so others can run. There are two kinds:
1. __Cooperative Preemption__: The task __voluntarily__ yields the resource.
    - Efficient, but not reliable if tasks misbehave.
2. __Non-Cooperative Preemption__: The scheduler __forcefully__ stops a task after a fixed time slice.
    - Ensures fairness, though it introduces context-switching overhead.

Go uses __non-cooperative__ preemption to ensure that 
__long-running goroutines__ don’t hog the CPU. The scheduler cuts them off after
a time slice and reschedules them, allowing other goroutines to proceed.

---

### How the Scheduler Picks a Goroutine to Run

Now that we have enough context, a natural question arises: __how does the__
__scheduler choose which goroutine to run__?

Each processor (P) has a local runqueue, and there is also a global runqueue
shared by all processors. Let's say the currently executing goroutine on a
processor completes or exits by calling into the runtime. At this point, the
processor must find a new goroutine to execute.

The __first place__ it checks is its __local runqueue__. Accessing this queue
does not involve any locking, so it's fast and contention-free. If goroutines
are available there, one is dequeued and begins execution. This path is the
fastest and most efficient.

But what happens if the local runqueue is __empty__?

---

### Global Runqueue and Load Sharing

If the processor’s local runqueue is empty, it next checks the __global runqueue__.
Because this queue is shared across processors, accessing it requires 
__acquiring a lock__, which introduces contention. To minimize this, Go employs
some clever strategies.

When a processor steals work from the global runqueue, it 
__doesn't just take one goroutine__. Instead, it takes a batch proportional to:
```go
len(globalRunQueue) / GOMAXPROCS
```

This strategy ensures fairness across processors and reduces frequent locking by
allowing each processor to grab a fair share of work.

Once the batch is moved into the local runqueue, the processor resumes by
executing from there.

---

### Checking the Netpoller for Asynchronous I/O

If the global runqueue is also empty, the scheduler looks for another source of
goroutines: the __netpoller__.

The netpoller is responsible for __asynchronous I/O__, especially network
operations. Instead of blocking an OS thread while waiting for data, goroutines
register with the netpoller. When the I/O is ready, the netpoller wakes the
goroutine.

So, if the global and local runqueues are empty, the processor checks the
__netpoller__. If any goroutines are ready due to I/O events, they’re picked up
and executed.

This asynchronous strategy is also used for __timers__ and __channel operations__,
providing non-blocking behavior without wasting OS resources.

---

### Work Stealing from Other Processors

If the netpoller also yields no work, the scheduler moves to its __last resort: stealing work__ from other processors.

Here's how it works:
1. A processor randomly selects another processor.
2. If the selected processor has goroutines in its local runqueue, the idle processor __steals half__ of them.
3. If the chosen processor has no work (or the picker chooses itself), the process __retries up to 4 times__.
3. Once goroutines are stolen, they are added to the local runqueue and execution resumes.

This __bounded and randomized stealing__ avoids inefficiencies seen in unbounded
thread pools and helps balance work across all available processors.

---

###  Preemption and Go 1.14: Time-Sliced Scheduling

A significant enhancement came in __Go 1.14__ with the introduction of
__non-cooperative (asynchronous) preemption__.

Before Go 1.14, goroutines were only preempted __cooperatively__, meaning they
had to call into the runtime (e.g., via function calls) to yield control.
Long-running goroutines without such calls could __starve__ other goroutines.

With Go 1.14, each goroutine is granted a __10ms time slice__. If it exceeds
this without yielding, the runtime __forcefully issues a preemption request__.

Preemption is implemented via a __user-space signal__ (`SIGURG`) sent to the OS
thread running the goroutine. When received, this signal triggers the goroutine
to pause and yield execution.

This ensures __fair CPU time__ for all goroutines, improves responsiveness, and
mitigates starvation.

Another important question is: __who actually sends the preemption signal__, and
where does it originate?

This responsibility falls on a long-running daemon in the Go runtime called
`sysmon`. This special goroutine operates __without a processor (P)__-meaning it
does not interfere with user-level Go code execution. The reason it doesn't
require a processor is that it performs critical background tasks for the
runtime, two of which are particularly important.

The __first responsibility__ is managing __preemption__. When `sysmon` observes that a
goroutine has been running continuously for more than __10 milliseconds__, it takes
action. Specifically, it sends a `SIGURG signal` to the OS thread executing the
long-running goroutine. This signal serves as a user-space preemption request,
prompting the runtime to forcefully yield that goroutine.

But what happens to a __preempted goroutine__, especially one that's running a __tight or infinite loop__?

Once it's preempted, the scheduler must decide where to place it for future
execution. If we simply put it back into the __local runqueue__ of the current
processor, it could cause problems. For instance, imagine the other goroutines
in the local runqueue are all short-lived. If the preempted goroutine is
inserted back into this queue, it will likely be picked up immediately
again—leading to repeated preemption and __starvation of shorter tasks__.

To prevent this, Go moves the __preempted goroutine to the global runqueue__
instead. This ensures that it is scheduled only __after other goroutines have had__
__their chance to run__, maintaining __system fairness__ and improving responsiveness.
It helps avoid a situation where one CPU-bound goroutine can monopolize
execution time while others are left waiting.

This design choice reinforces the Go scheduler's emphasis on __efficiency and fairness__, making it resilient against long-running or poorly-behaved goroutines.

---

### Goroutine Spawn Placement: FIFO vs. LIFO

Another important question to consider is: __when a goroutine spawns other__
__goroutines, where do the newly spawned goroutines go__?

The answer is that these __spawned goroutines are placed into the local runqueue__
of the processor (P) that spawned them. At this point, a design decision
arises-__should these new goroutines be added to the tail of the queue__ in a
FIFO (First In, First Out) manner, or can something more efficient be done?


#### FIFO: Fairness at the Cost of Locality

A pure __FIFO approach__ ensures fairness by allowing older goroutines to run first.
However, it performs __poorly in terms of locality__-newly spawned goroutines that
are likely to interact soon (e.g., a sender and receiver) would be pushed to the
back, incurring unnecessary delay.

#### LIFO: Locality at the Cost of Fairness

Conversely, __LIFO (Last In, First Out)__ enhances locality by running the newest
goroutines first, but this comes at the cost of fairness—older goroutines can
starve.

#### Go's Hybrid Strategy: Time Slice Inheritance

Go addresses this trade-off through practical optimizations informed by common
usage patterns. A frequent pattern involves goroutines communicating over
__channels__, such as a sender and a receiver exchanging data.

Let’s consider a case with __unbuffered channels__. If the receiver runs first
and blocks waiting for data, the sender must run next to unblock it. However, in
a strict FIFO queue, the receiver would be pushed to the back and might wait a
long time, especially if the runqueue is filled with long-running goroutines.

To avoid this, Go allows new goroutines to be added __to the head of the queue__,
improving responsiveness and locality. But this raises a risk: the newly spawned
goroutines could continually preempt older ones, causing starvation.

To mitigate this, Go uses __time slice inheritance__. When a goroutine spawns
another, the new goroutine __inherits the remaining time slice__ of its parent. For
example, if a goroutine spawns another at the 3ms mark of its 10ms slice, the
child gets 7ms.

This allows both parent and child to share the same 10ms time slice, balancing
locality and fairness. After this slice is exhausted, other goroutines get a
chance to run.

---

### Global Runqueue Starvation: Periodic Polling

Even with time slice inheritance, goroutines in the __global runqueue might starve__
if processors don’t periodically check the global queue.

To solve this, Go occasionally __polls the global runqueue__. But how often should
this polling occur?

The polling frequency is controlled by a counter called `schedtick`, which
increments every time a runnable goroutine is found and executed. If `schedtick`
is a multiple of 61, the global runqueue is checked.

---

### Blocking System Calls and Handoff

What happens when a __goroutine is blocked on a system call__? In this case, the
thread is unavailable to execute other goroutines—even though there's work in
the local or global runqueue.

To handle this, Go uses __handoff__:
1. The processor and thread are disassociated using `releasep()`.
2. A new or existing thread is assigned to the processor.
3. The processor resumes scheduling other goroutines.

This handoff is expensive, particularly if new threads must be created
frequently. Therefore, __Go avoids immediate handoff unless it knows the system__
__call will block for a long time__ (e.g., cgo calls).

For short system calls, the runtime marks the processor as in a system call
state and waits. If the call returns quickly, no thread switching is needed. If
not, the `sysmon` goroutine eventually detects the prolonged block and initiates
the handoff.

---

### Returning from System Calls

When a goroutine returns from a system call:
- The scheduler __tries to assign it to its original processor__ to preserve
locality.
- If the original processor is busy, it tries an idle processor.
- If none are available, the goroutine is put back into the global runqueue for
other processors to pick up.

This strategy optimizes for cache locality and reduces unnecessary context
switching.

---

### Conclusion

The Go scheduler is a finely tuned, hybrid system designed to balance
performance, fairness, and responsiveness. By blending FIFO and LIFO strategies,
leveraging time slice inheritance, and periodically polling the global runqueue,
it ensures that goroutines are scheduled efficiently without starving others.

Additionally, features like the sysmon daemon, preemption signals, and the
handoff mechanism for system calls demonstrate the scheduler's adaptability in
real-world scenarios, especially in handling concurrency bottlenecks and
maximizing CPU utilization.

Understanding these internal mechanisms provides a deep appreciation for Go's
concurrency model and offers insights into writing more performant and
predictable concurrent programs. The elegance of Go's scheduler lies not just in
its algorithms, but in its practical optimizations rooted in common usage
patterns, achieving a delicate balance between theoretical soundness and
engineering pragmatism.

### Resources:

- [Queues, Fairness, and The Go Scheduler](https://www.youtube.com/watch?v=wQpC99Xu1U4&list=PL2ntRZ1ySWBfulCVQD6EaU8c-GM56aUU7&index=14)
- [Scheduling In Go](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)