# Integration Points: The Number One Killer

**Sources:** *Release It! Design and Deploy Production-Ready Software*, Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 4 "Stability Antipatterns", Sec. 4.1 "Integration Points",
pp. 46–60. Ch. 9 "Capacity Antipatterns", Sec. 9.9 "Integration Point Latency", pp. 199–200.
Page numbers are the PDF page numbers of the extracted source.

Read this file when you add an outbound call. Read it when you review one. Read it when a
vendor call behaves in a way that your error handling did not expect.

---

## 1. The claim

Nygard names the largest risk to stability in one sentence. "Integration points are the
number-one killer of systems." (Sec. 4.1, p. 46.)

The mechanism is simple. Every socket, process, pipe, and remote procedure call can hang.
Database calls can hang too. Every feed can hang the system, crash it, or deliver a load
impulse at the worst moment. (Sec. 4.1, p. 46.)

**The failure is not optional.** "Every integration point will eventually fail in some way,
and you need to be prepared for that failure." (Sec. 4.1, p. 59.)

**Count the integration points before you defend them.** You cannot protect a call that you
have not listed. Nygard's "How Many Feeds?" story shows a team that could not enumerate its
own feeds well enough to open the firewall rules. (Sec. 4.1, p. 47.) Section 10 tells the
story. Catalog entry `I-41` names the defect.

**A firewall rule is an integration point.** A hole in the firewall exists to call some other
system. (Sec. 14.1, p. 243.) Search the firewall rules, the outbound hosts, and the SDK list
in the manifest. Each entry is a call that can hang.

---

## 2. The six ways an integration point fails

Nygard's own summary states the range. You will not receive a clean error inside the
protocol. You will see a protocol violation, a slow response, or a hang. (Sec. 4.1, p. 59.)

| Failure class | What happens on the wire | What the caller sees | Time to detect with no defense | Required defense |
|---|---|---|---|---|
| The remote host refuses the connection | Nothing listens on the port. The host returns a TCP reset packet. | An exception, or `-1` in C | Under ten milliseconds when both machines share a switch (p. 48) | Normal error handling. Count the failure in the breaker. |
| The connection waits in the listen queue | The caller sent SYN. No SYN/ACK returns. | Nothing. No error and no answer. | Minutes. The operating system decides. Up to ten minutes. (p. 49) | An explicit connect timeout. A circuit breaker. |
| The remote host accepts and never answers | The handshake finished. No response bytes arrive. | Nothing. The socket stays open. | Forever. In Java `read` blocks by default. (p. 49) | An explicit read timeout. A deadline. |
| The answer arrives slowly | Bytes arrive late, or one byte at a time | A valid answer that arrives too late | The caller's own patience | A deadline near the observed latency distribution |
| The answer violates the protocol | Bytes arrive with the wrong shape | A parse error, or a silently wrong value | Never, when the code does not check | Status check, content type check, schema check, size cap |
| The answer is a well formed error | The vendor returns a defined error response | A defined error response | Immediately | Classify the error as retriable or permanent. Feed the breaker. |

**The ordering rule.** "Clearly, a slow response is a lot worse than no response."
(Sec. 4.1, p. 50.) A refusal holds no thread on either side. A slow answer holds one thread
in the caller and one thread in the callee. Rank your defenses by this rule.

**Fast failure and slow failure are the two shapes.** A fast network failure raises an
exception in milliseconds. A slow network failure, such as a dropped ACK, blocks threads for
minutes. A blocked thread serves no other transaction, so capacity falls. When every thread
blocks, the server is down for practical purposes. (Sec. 4.1, p. 50.)

---

## 3. The socket is an abstraction over packets

An arrow on an architecture diagram is an abstraction of a network connection. A connection
is itself an abstraction. The network carries only packets. (Sec. 4.1, p. 48.)

TCP builds the connection with a three-way handshake.

1. The caller sends a SYN packet to a port on the remote host.
2. The remote host returns a SYN/ACK packet to accept the connection.
3. The caller returns its own ACK packet.

After step 3 the two applications can exchange data. (Sec. 4.1, pp. 48–49.)

**Refusal is the one failure the API makes obvious.** When no application listens on the
port, the remote host returns a TCP reset packet at once. C returns `-1`. Java, C#, and Ruby
raise an exception. Programmers handle this case, because the interface forces them to.
(Sec. 4.1, pp. 46, 48.)

**Every other failure hides behind the same call.** The defenses in
`03-stability-patterns.md` exist because the other five classes produce no signal that the
language forces you to read.

---

## 4. The listen queue

Each port holds a listen queue. The queue counts the connections that are pending. A pending
connection has sent SYN and has received no SYN/ACK. (Sec. 4.1, p. 49.)

Two states follow, and they behave in opposite ways.

| State of the listen queue | What the remote host does | What the caller experiences |
|---|---|---|
| The queue is full | It refuses further connection attempts quickly | A fast, visible failure |
| The queue holds the connection | It leaves the connection partly formed | The calling thread blocks inside the operating system kernel |

**The second state is the dangerous one.** "The listen queue is the worst place to be."
(Sec. 4.1, p. 49.) The thread that called `open()` waits inside the kernel. It waits until
the remote application accepts the connection, or until the connect attempt expires.

**The operating system owns that limit unless you set your own.** Connect timeouts differ
between operating systems. Nygard states that they are usually measured in minutes. One
calling thread can wait ten minutes for the remote server. (Sec. 4.1, p. 49.)

**Rule: set an explicit connect timeout on every client.** Catalog entry `I-01` names the
defect. A client that you construct with no connect timeout inherits a limit that nobody on
your team chose.

---

## 5. The read with no end

The same trap appears after the handshake succeeds. The caller connects. The caller sends
its request. The remote application then reads the request slowly, or never answers. The
`read()` call blocks until the answer arrives. (Sec. 4.1, p. 49.)

**In Java the default is to block forever.** You must call `Socket.setSoTimeout()` to leave
the blocking call, and you must handle the resulting `IOException`. (Sec. 4.1, p. 49.)

**A dribble of bytes defeats a naive design.** Nygard's example is exact. The remote system
can return one byte per second for ten years, and your thread stays on that one call.
(Sec. 4.1, p. 57.)

**Rule: set an explicit read timeout on every call.** Catalog entry `I-02` names the defect.
A connect timeout alone protects only the handshake.

---

## 6. Why a dropped idle connection produces a hang and not an error

This mechanism produces silence and not an error. A firewall or a NAT device
keeps a finite table of established connections. The table cannot hold connections of
infinite duration, although TCP itself permits them. The device also records a "last packet"
time for each entry. After enough idle time, the device assumes the endpoints are dead and
removes the entry. (Sec. 4.1, p. 54.)

**TCP has no way to tell the endpoints.** TCP was not designed for an intelligent device in
the middle of a connection. A third party cannot signal either endpoint that the connection
is gone. Both endpoints continue to believe that the connection is valid. (Sec. 4.1, p. 54.)

What follows is the whole answer to the question.

| Party | What it believes | What it does |
|---|---|---|
| The caller's TCP stack | The connection is valid | It sends the packet, waits for an ACK, receives none, and retransmits |
| The middlebox | The connection no longer exists | It drops each packet. It sends no reset packet and no ICMP message. |
| The remote host | The connection is valid | It waits for traffic that never arrives |
| The application thread | The call is in progress | It waits inside `read` or `write` |

**No reset packet and no ICMP message arrive.** The device stays silent on purpose. An ICMP
"destination unreachable" reply would let an attacker probe for active connections with
spoofed source addresses. (Sec. 4.1, p. 54.)

**So the caller does not receive an error. The caller receives silence.** Nygard states the
contrast twice on the same page. A read or a write on either end does not produce a TCP
reset. It does not produce the error of a half open socket either. The stack retransmits
into a black hole instead. (Sec. 4.1, p. 54.)

**The wait then comes from the retransmit policy of the operating system.**

| Platform in the story | Setting | A write blocks for | A read blocks for |
|---|---|---|---|
| Linux, 2.6 series kernel | `tcp_retries2 = 15`, the default | About twenty minutes before the stack reports a broken connection (p. 54) | Longer than the write case |
| HP-UX, the servers in use | Vendor default | Thirty minutes (p. 55) | It can block forever (p. 55) |

Do not copy these two numbers into a design. They are the numbers of the machines in
Nygard's story. Read the setting on your own hosts, and set an application timeout that is
much shorter than it.

**Two design consequences follow.**

1. Give every pooled connection keepalive traffic, so the middlebox resets its "last packet"
   time. Catalog entry `I-03` names the defect.
2. Learn the idle timeout of every device in the route. The number is a local decision. Read
   the configuration, do not assume a value.

**Modern.** A cloud NAT gateway, a managed load balancer, and a service mesh sidecar each
apply their own idle timeout to a pooled connection. The mechanism is the one Nygard
describes. Read the provider's documented idle timeout, then set the pool keepalive interval
below it. No book supplies that number.

---

## 7. HTTP does not remove any socket failure

Higher-level protocols run over sockets. They add their own failure modes. They remain open
to every failure at the socket layer. (Sec. 4.1, p. 46.)

Nygard's example is `java.net.HttpURLConnection`. One call opens the socket, sends the
request, waits for the response, parses it, and returns a stream. It is one large blocking
call with no parameters. `getInputStream()` carries no timeout. Setting
`Socket.setSoTimeout()` requires a `SocketImplFactory` that would change every socket in the
process, not only the sockets of this interaction. (Sec. 4.1, p. 57.)

**Rule: choose an HTTP client that exposes the connect timeout and the read timeout
separately.** Nygard names the Apache Jakarta Commons `HttpClient` package as an example
with that control. (Sec. 4.1, p. 57.)

**Write cynical software.** A cynical system refuses an unprotected call. It handles
violations of form and of function, such as a badly formed header or a connection that
closes without warning. (Sec. 4.1, pp. 57, 59.)

---

## 8. The vendor client library is the weak point

Vendors harden the server software that they sell. They rarely harden the client library.
The client library is ordinary code, with the same variation in quality as any other sample
of code. You control very little of it. (Sec. 4.1, p. 57.)

**Blocking is the main risk.** Nygard states it directly. The prime stability killer in
vendor API libraries is blocking, in an internal resource pool, in socket reads, in HTTP
connections, or in Java serialization. (Sec. 4.1, p. 58.)

**A callback interface is a deadlock risk.** A library that offers `registerCallback` and
`messageReceived` does not tell you which thread invokes your method. You cannot know which
monitors that thread already holds. A slow or synchronized callback blocks threads inside
the library. Those blocked threads then block your calls to `send()`. Those blocked calls
then hold your request-handling threads. (Sec. 4.1, p. 58.)

**The remedies Nygard names, in order of what you control.**

1. Decompile the library, find the defect, and report it as a bug. (Sec. 4.1, p. 57.)
2. Apply commercial pressure to the vendor for a fix. (Sec. 4.1, p. 57.)
3. Acquire locks in one consistent order, and release them in the reverse order.
   (Sec. 4.1, p. 58.)
4. Make the callback place work on your own queue and return at once. Catalog entry `I-32`
   names the defect.
5. Keep the dangerous call away from the request-handling thread. Catalog entry `I-20` names
   the defect. Read `03-stability-patterns.md` for the pattern.

---

## 9. Integration point latency

Sec. 9.9 supplies the cost model. Every communication with another system carries latency.
"A remote call takes at least 1,000 times as long as a local call." (Sec. 9.9, p. 199.)

**The caller inherits the latency of the callee.** The caller is processing a transaction,
or it would not be calling. So the caller takes at least as long to answer as the remote
system takes. (Sec. 9.9, p. 199.)

**Location transparency is discredited for two named reasons.** (Sec. 9.9, p. 199.)

| Reason | What it means for your design |
|---|---|
| A remote call has different failure modes than a local call | It is open to network failure, to failure in the remote process, and to a version mismatch between caller and server |
| It leads developers to design a remote interface like a local one | The result is a chatty interface. Each method call adds its own latency. The total response time becomes very slow. |

**The individual becomes the collective.** "Performance problems for individual users become
capacity problems for the entire system." (Sec. 9.9, p. 199.) This is the sentence that
explains why a vendor latency problem becomes your outage.

**A thread that waits is not a thread that is free.** Nygard lists what the idle thread still
holds. (Sec. 9.9, p. 199.)

| The blocked thread holds | The consequence |
|---|---|
| Memory | Less memory remains for the requests that arrive next |
| CPU time slices | The thread consumes them while it does nothing |
| Database connections that other threads need | Contention at the pool. Read `06-cascading-failure.md`. |
| Row locks or page locks in the database | Contention inside the database itself |
| Its own capacity to do queued work | The opportunity cost. Queued work waits with a thread available in name only. |

**Rule: expose yourself to latency as seldom as possible.** (Sec. 9.9, p. 199.) Nygard
compares integration point latency to the house advantage in blackjack. The more often you
play, the more often it works against you.

**Rule: avoid a chatty remote protocol.** A chatty protocol takes longer to execute, and it
holds request-handling threads. (Sec. 9.9, p. 199.) The named remedy is the **Summary
Object**. One coarse call returns a collection of small objects. Each object carries exactly
the fields that the caller needs. (Sec. 9.9, pp. 199–200.)

**The 1+N shape is the signature to search for.** One call retrieves a list. Then the code
calls once, or more than once, for each element of the list. (Sec. 9.9, p. 200.)

---

## 10. Three war stories

**How Many Feeds? (Sec. 4.1, p. 47.)** A large retailer replatformed its site. Opening the
production firewall rules required a list of every feed in and out of the environment. The
usual connections were easy. Nobody could name the feeds. The project had a dedicated
project manager for enterprise integration. He held a populated database of integrations.
It recorded the transport, the source system, the frequency, the volume, and the business
stakeholder of each feed. He ran a report and supplied the list. Nygard was impressed that
the database existed and dismayed that it was necessary. The site launched with stability
problems that woke him at 3 a.m. He never kept a tally. He is sure that every synchronous
integration point caused at least one outage. **What it proves:** an integration inventory
is the first artifact, not a document you write later.

**The 5 a.m. Problem (Sec. 4.1, pp. 50–56.)** About thirty application server instances hung
within a five-minute window at almost exactly 5 a.m. every day. East Coast traffic began to
rise at that hour from roughly 100 transactions per hour. A restart always cleared it.
Thread dumps showed every request-handling thread inside the Oracle JDBC library, in
low-level socket reads and writes. Packet capture showed almost no traffic in either
direction. The database was healthy, with no blocking locks, an empty run queue, and trivial
I/O. The chain was this. The connection pool returned the most recent connection first.
So one connection served the light overnight traffic, and thirty-nine sat idle past the
one-hour idle timeout of the firewall. The firewall removed those entries in silence. The
TCP stack then retransmitted into a black hole. Two candidate fixes were rejected because
each one would itself hang, a validity query and an idle-age discard. The fix that worked
was Oracle dead connection detection, whose periodic ping resets the "last packet" time of
the firewall. **What it proves:** a middlebox can invalidate the connection abstraction
without either endpoint learning of it. The absence of packets is a positive diagnostic
finding.

**RMI Across the Seas (Sec. 9.9, p. 200.)** A fat-client Java system used RMI to reach its
server. The server presented an object view of the client's enterprise data warehouse.
Expanding one node of a tree control took less
than a second for users in the United States. The same action took twenty minutes for users
in the United Kingdom. Those users asked for a local server and a local warehouse, a
multimillion-dollar investment. The client made one remote call for the collection of
children. It then called each child three times, for the name, the type, and the number of
children. The team added one method that returned Summary Objects carrying those three
fields. The United Kingdom users then saw the same subsecond response as the United States
users, and nobody replicated the warehouse. **What it proves:** protocol shape, not
bandwidth, decided the response time. The same code was correct and unusable at different
distances.

---

## 11. What to do about it

Nygard names the countermeasures in Sec. 4.1, pp. 59–60. Each one has an owner file in this
skill.

| Countermeasure | Signal that you need it now | Where to read it |
|---|---|---|
| Circuit Breaker | You call a dependency that can be sick for minutes | `03-stability-patterns.md` |
| Decoupling Middleware | The call does not need an answer in the same request | `03-stability-patterns.md` |
| Use Timeouts | Any outbound call at all | `03-stability-patterns.md` |
| Handshaking | The callee can tell you that it is saturated | `03-stability-patterns.md` |
| A test harness for each integration point | You cannot produce these six failures on demand | `08-fault-injection-and-test-harness.md` |
| Open the abstraction with a packet capture | The application layer shows no error and no answer | `04-fault-localization.md` |

**Build one test harness for each integration point.** The harness is a simulator with
controllable behavior. Canned responses give you functional testing and isolation from the
target. Switches let you simulate several kinds of system failure and network failure. For a
stability test, operate every switch while the system carries a large load.
(Sec. 4.1, p. 59.) A mock cannot produce a hang, a dribble, or a middlebox that discards
packets.

**Know when to open an abstraction.** Debugging an integration point failure usually
requires the removal of one layer of abstraction. Most of these failures violate the
high-level protocol, so the application layer cannot describe them. Packet sniffers and
other network diagnostics help. (Sec. 4.1, p. 59.)

**Expect the failure to propagate.** "Failure in a remote system quickly becomes your
problem, usually as a cascading failure when your code isn't defensive enough."
(Sec. 4.1, p. 60.) Read `06-cascading-failure.md` for the transmission mechanisms.

---

## 12. Review checklist for one integration point

Apply this list to each outbound call in the change. Report only the rows that apply. "Not
applicable" is a valid result.

| Question | Failure it prevents | Catalog code |
|---|---|---|
| Does the client carry an explicit connect timeout? | The thread blocks in the listen queue for minutes | `I-01` |
| Does the client carry an explicit read timeout? | The thread waits with no end for a dribble of bytes | `I-02` |
| Does the pool send keepalive traffic across every middlebox? | The pool locks at the next traffic rise | `I-03` |
| Does every blocking wait carry a time bound, including pool checkout? | Every request thread blocks on checkout | `I-08` |
| Does the call have its own circuit breaker? | The caller keeps calling a sick dependency | `I-15` |
| Does this integration point have its own resource pool? | One sick vendor drains the pool for every vendor | `I-19` |
| Does a worker thread make the call, and not the request thread? | A remote problem becomes local downtime | `I-20` |
| Does a vendor callback place work on your own queue and return at once? | A plugged library thread stops your senders | `I-32` |
| Does the code check the status, then the content type, then the schema? | A proxy error page reaches the parser as data | `I-30` |
| Does the request state the maximum response that you accept? | An unbounded response body exhausts memory | `I-29` |
| Does the integration point appear in the dependency inventory? | Nobody can enumerate the feeds | `I-41` |
| Does the protocol avoid the 1+N shape across the network boundary? | Latency accumulates for each element | No code. Sec. 9.9 owns it. |

Every code above resolves in `integration-fault-catalog.md`.

**Two boundaries this file does not own.** For system-call correctness at the socket, such
as `EINTR`, a short read, or a descriptor limit, read
`systems-programming/references/07-ipc-and-sockets.md`. For the safety of repeating a write
after a failed call, read `data-systems-design/references/hazard-catalog.md`. This file
decides that the call can fail. That file decides whether the retry is safe.
