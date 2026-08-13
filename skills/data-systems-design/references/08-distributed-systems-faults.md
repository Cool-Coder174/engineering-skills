# The Trouble with Distributed Systems

**Source:** DDIA Ch. 8, p. 273.

The purpose of this chapter is pessimism: to establish what you may **not** assume once
more than one machine is involved. Every optimistic assumption listed here has produced
production incidents.

---

## 1. Faults and partial failures

A single computer is mostly deterministic: it works or it doesn't. A distributed system
has **partial failure** — some parts work, some don't, **and it is nondeterministic**. An
operation involving multiple nodes may succeed on some and fail on others, and you often
cannot find out which.

Two different worlds:
- **Supercomputing / HPC:** treat partial failure as total failure; checkpoint and restart.
- **Internet services / cloud:** must stay up; nodes are commodity and fail often; the
  network is shared, multi-tenant, and unreliable. **This is the model you must assume.**

You must build a reliable system from unreliable components, and you must **know exactly
which guarantees you're relying on**.

---

## 2. Unreliable networks

Shared-nothing systems communicate over an asynchronous packet network with **no
guarantees**: a message may be lost, delayed arbitrarily, delivered out of order, or
duplicated. The response may be lost or delayed too.

**When you send a request and get no reply, you cannot distinguish:**
1. the request was lost,
2. the request is queued and will be delivered later,
3. the remote node crashed,
4. the remote node paused and will respond later,
5. the response was lost,
6. the response is delayed.

**This is the fundamental constraint.** All you can do is time out — and a timeout does
not tell you whether the operation was executed. This is why **every remote operation with
side effects must be idempotent or protected by a deduplication key**.

### 2.1 Network faults are common
They happen in real datacenters — switch upgrades, misconfigurations, faulty NICs (which
sometimes drop inbound packets while still sending, so the node appears alive to itself),
partitions, and rare-but-real behaviors like nodes being able to send but not receive.
Handling them does not necessarily mean tolerating them; it means **you have decided and
tested how the software reacts**. Deliberately trigger them (chaos engineering) to find
out.

### 2.2 Detecting faults
- A TCP RST/FIN or an ICMP "port unreachable" is a **fast, useful signal** — use it where
  available.
- A process crash where the OS is still up can be reported by a supervisor.
- **But if the node's power failed or the network is down, you get nothing.** You are left
  with timeouts.

### 2.3 Timeouts and unbounded delays
- **Too long:** users wait, and failure detection is slow.
- **Too short:** you declare a healthy-but-slow node dead. Then the work is performed
  **twice** (once on each node), and shifting its load to other nodes during a load spike
  can cascade into a system-wide failure.

Delay is unbounded because packet networks use **dynamic resource partitioning** —
queueing at switches, at the receiving OS when all cores are busy, at the VM level (a
paused VM's packets queue), and TCP's own retransmission and flow control.

**Practical rule:** measure the distribution of round-trip times over a long period across
many machines, and set timeouts from the observed distribution — or better, use adaptive
timeouts (Phi Accrual failure detectors, TCP-style RTT estimation with jitter). A
hard-coded 30-second timeout copied from a tutorial is a design decision no one made.

**Also required:** every outbound call needs *some* timeout. A missing timeout is an
unbounded resource leak that eventually exhausts the connection or thread pool and takes
down the caller — the single most common cascading-failure mechanism.

---

## 3. Unreliable clocks

Each machine has its own quartz oscillator, drifting, synchronized (imperfectly) by NTP.

### 3.1 Two kinds of clock — do not mix them up

| Clock | Use for | Never use for |
|---|---|---|
| **Time-of-day clock** (`System.currentTimeMillis`, `clock_gettime(CLOCK_REALTIME)`) | Displaying dates, correlating with external systems | Measuring elapsed time; ordering events |
| **Monotonic clock** (`System.nanoTime`, `clock_gettime(CLOCK_MONOTONIC)`) | Measuring durations, timeouts, rate limits | Absolute timestamps; comparing across machines |

A time-of-day clock **can jump backwards** when NTP steps it, or when a leap second is
handled badly (this has crashed many systems). Measuring an interval by subtracting two
wall-clock readings can yield a negative or wildly wrong duration.

### 3.2 Accuracy is worse than you think
Clock drift, firewalled/misconfigured NTP, network delay bounding NTP accuracy, leap
seconds, VM clock jumps on suspend/resume, and untrusted device clocks all contribute.
Achieving tight accuracy is possible (GPS/PTP) but requires significant, deliberate effort
and monitoring. **Assume tens of milliseconds of skew at best and, occasionally, minutes.**

### 3.3 Relying on synchronized clocks is dangerous
The danger is that clock problems are **silent**: a wrong clock does not crash, it quietly
produces wrong results. Consequences:

- **Last-write-wins conflict resolution silently drops writes** when the "later" write has
  an earlier timestamp due to skew — and it cannot distinguish genuinely concurrent writes
  from sequential ones. Use version vectors instead (see `05-replication.md`).
- **Timestamps cannot order events across nodes.** Use logical clocks (Lamport timestamps,
  version vectors) for ordering — they are safe by construction.
- If you *must* use clocks for correctness (e.g. Spanner's TrueTime), you need a
  **confidence interval**, and you must wait out the uncertainty. Almost no one has that.

### 3.4 Process pauses
A thread can be paused for an arbitrary, unbounded time at any point:
stop-the-world GC (seconds to minutes on large heaps), VM suspend/live migration, laptop
lid close, synchronous disk I/O and page faults from swapping, and `SIGSTOP`.

**Consequence:** a node holding a lease can pause past its expiry, wake up believing it is
still the leader, and issue a write. `if (lease.isValid()) { doWrite(); }` is unsafe —
the pause can happen between the check and the write.

There is no general fix inside the process. The safe pattern is:

> **Fencing tokens.** The lock/lease service issues a **monotonically increasing number**
> with each grant. Every write to the protected resource carries its token. The **resource**
> remembers the highest token it has processed and **rejects any write with a lower token**.

Crucially, the **resource must enforce this** — clients checking their own lock status is
not sufficient, because a paused client believes its lock is valid. If the resource does
not natively support fencing, approximate it (e.g. embed the token in the object key or
filename, or use a conditional write on a version column).

ZooKeeper's `zxid` or node `cversion` work as fencing tokens because they are guaranteed
monotonically increasing.

---

## 4. Knowledge, truth, and lies

- **The truth is defined by the majority.** A node cannot trust its own judgment about its
  own status. A quorum of nodes decides who is alive and who is leader. A node that has
  been declared dead must comply, even if it feels fine.
- **Byzantine faults** — nodes that lie or send arbitrary corrupted messages — are out of
  scope for most systems: within a single trusted datacenter, the standard assumption is
  that nodes are **unreliable but honest**. If your system spans mutually untrusting
  parties (public networks, blockchains, aerospace with radiation), you need Byzantine
  fault tolerance, which is a much heavier design.
  - What you *should* borrow regardless: checksums on data at rest and in transit, input
    validation and sanitization at every trust boundary, and never trusting a client-
    supplied value that has security or correctness meaning (price, user ID, quantity).

### 4.1 System model — state yours
Timing assumptions:
- **Synchronous:** bounded delay, pauses, and clock error. Unrealistic.
- **Partially synchronous:** behaves synchronously most of the time, occasionally exceeds
  bounds. **This is realistic and is the model you should assume.**
- **Asynchronous:** no timing assumptions; no clocks or timeouts allowed.

Node failure assumptions:
- **Crash-stop:** a node that fails never comes back.
- **Crash-recovery:** nodes may crash and restart, retaining only what was written to
  stable storage. **This is realistic.**
- **Byzantine:** nodes may do anything.

**Safety vs. liveness:** safety = "nothing bad happens" (must hold always; a violation
cannot be undone). Liveness = "something good eventually happens" (may be temporarily
violated). Distributed algorithms are normally required to preserve safety under all
timing assumptions, while liveness holds only under conditions like "a quorum is
reachable". Write your requirements this way — it makes degradation decisions obvious.

---

## 5. Implementation patterns that follow

| Assumption you must not make | What to do instead |
|---|---|
| The network is reliable | Timeout + bounded retry + backoff with jitter on every call |
| A timeout means it didn't happen | Idempotency key; dedup at the point of effect |
| Latency is low / bounded | Explicit timeouts, circuit breakers, bulkheads, load shedding |
| Retries are safe | Make the operation idempotent, or use a unique constraint |
| Clocks are synchronized | Monotonic clocks for durations, logical clocks for ordering |
| I still hold the lock | Fencing token enforced by the resource |
| The node is dead because it didn't reply | Quorum decides; expect the "dead" node to come back |
| The queue will drain | Bound every queue; define shedding behavior when full |
| Everyone retried at once by coincidence | Jittered backoff; retry budgets to prevent retry storms |

**Retry storm caution:** naive retries multiply load on an already struggling dependency.
Use a **retry budget** (cap retries as a fraction of total requests), circuit breakers, and
never retry at multiple layers of the stack simultaneously — nested retries multiply.

---

## 6. Review checklist

- [ ] Every network call has an explicit timeout, and the value is justified
- [ ] Retries use exponential backoff **with jitter** and a bounded attempt count
- [ ] Retries are not nested at multiple layers (client + SDK + gateway)
- [ ] A retry budget or circuit breaker exists for each critical dependency
- [ ] Every retryable operation with side effects is idempotent or has a dedup key
- [ ] Ambiguous-outcome handling exists (timed-out write: verify, don't blindly retry)
- [ ] Durations measured with a monotonic clock, never wall-clock subtraction
- [ ] No correctness logic depends on cross-node wall-clock comparison
- [ ] LWW-by-timestamp is not used where lost writes are unacceptable
- [ ] Distributed locks/leases are backed by fencing tokens enforced at the resource
- [ ] Leadership is decided by quorum, not self-assessment
- [ ] Every queue and buffer is bounded, with defined behavior when full
- [ ] Checksums / validation at trust boundaries; client-supplied values never trusted
- [ ] System model written down: partially synchronous, crash-recovery, non-Byzantine
- [ ] Requirements split into safety (always) and liveness (under stated conditions)
