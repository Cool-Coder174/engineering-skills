# The Troubleshooting Model

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Chapter 12, "Effective Troubleshooting", written by Chris Jones. Theory and pitfalls at
p. 169–172. Conclusion at p. 187. Practice steps and case studies at p. 173–186.

This file gives the whole model and the two failure modes of a new troubleshooter. Each
later reference file gives one step of the model in detail. Read this file first.

---

## 1. Troubleshooting is a skill, not a talent

**The book states that troubleshooting is both learnable and teachable**
(`SRE Ch. 12, p. 169`). Many teams treat it as an innate skill that some engineers hold and
others do not. The book rejects that view. It names one reason for the wrong view. An
engineer who troubleshoots often runs the process without conscious thought. The book
compares the difficulty of the explanation to the difficulty of explaining how to ride a
bike (`p. 169`).

**A systematic approach can bound the time to recovery** (`SRE Ch. 12, p. 187`). The stated
alternative is luck, or the experience of one person. The book states the benefit as a bound
on recovery time and a better experience for the user. Keep the hedge. The chapter says that
the approach *can* help, not that it always helps.

**Keep a method that already works for you.** The book invites a reader with expertise to
disagree with its definitions and its process (`SRE Ch. 12, p. 169`). This model is a
default, not a law.

The chapter opens with two epigraphs. Brian Redman states that expertise is more than an
understanding of how a system is supposed to work. He states that a person gains expertise
by investigating why a system does not work. John Allspaw states that the ways a system goes
right are special cases of the ways it goes wrong (`SRE Ch. 12, p. 169`). Both epigraphs
send the reader to the running system.

---

## 2. The method needs two things, and a new responder usually lacks one

The book states that the exercise depends on two factors (`SRE Ch. 12, p. 169`).

| Factor | What it is | What the book says about its absence |
|---|---|---|
| The generic method | How to troubleshoot with no knowledge of the particular system | You can still investigate from first principles, but usually less efficiently (`p. 169`) |
| Knowledge of the system | How the system is built, how it should operate, and its failure modes | This factor typically limits an engineer who is new to a system (`p. 169`) |

**You can solve a fault from first principles alone, but the book calls that path less
efficient.** The book compares two approaches. The first uses the generic process and
derivation from first principles. The second adds an understanding of how the parts are
supposed to work. The book states that the first approach is usually less efficient and less
effective (`SRE Ch. 12, p. 169`). Keep the two hedges. The book writes "usually", and it
writes "typically".

**The book states that there is little substitute for learning the design and the build of
the system** (`SRE Ch. 12, p. 169`). Footnote 58 at `p. 187` adds the counterweight. Work
from first principles is often an effective way to learn how a system works. So the two
factors feed each other. A responder who troubleshoots a system learns that system.

---

## 3. The formal frame

**The chapter frames troubleshooting as the hypothetico-deductive method**
(`SRE Ch. 12, p. 170`). The frame has four parts.

1. You hold a set of observations about the system.
2. You hold a theoretical basis for the behavior of the system.
3. You state possible causes as hypotheses.
4. You test each hypothesis.

The chapter then combines the two inputs. Telemetry and logs give the current state
(`p. 170`). Your knowledge of the build, the intended operation, and the failure modes turns
that state into a list of possible causes (`p. 170`).

**A hypothesis with no test is an opinion.** The frame requires the test. Read
`04-test-and-treat.md` for the four rules that a test must satisfy.

---

## 4. The loop

Figure 12-1 at `SRE Ch. 12, p. 170` names the steps. The chapter names them again at
`p. 171`. The signal column below gives the condition that starts each step.

| Step | Signal that starts it | What you do | Output | Detail |
|---|---|---|---|---|
| 1. Problem report | An alert fires, or a person reports a fault | Record expected behavior, actual behavior, and how to reproduce it (`p. 172`) | A ticket in a searchable queue | Section 5 below |
| 2. Triage | The report states the user impact | Make the system serve as well as it can. Preserve the evidence. (`p. 173`) | A mitigation and a saved evidence set | `02-triage-first-diagnose-second.md` |
| 3. Examine | The system serves again, or the impact is bounded | Read metrics, logs, traces, and exposed state (`p. 174–175`) | A list of components that behave wrongly | `03-examine-and-diagnose.md` |
| 4. Diagnose | You hold observations and no cause | Apply one technique. Divide and conquer, bisection, simplify and reduce, ask what and where and why, or what touched it last. (`p. 176–178`) | A short list of hypotheses | `03-examine-and-diagnose.md` |
| 5. Test and treat | You hold two or more hypotheses | Run the test that separates them (`p. 179–180`) | One hypothesis kept. The others removed. | `04-test-and-treat.md` |
| 6. Cure | One cause explains every observation | Repair the cause. Prevent the recurrence. Write the record. (`p. 171, p. 182`) | A Fault Record | `05-negative-results-and-bias.md`. `incident-response` owns the postmortem. |

**Read the signal, not only the step.** A responder who runs step 3 while the signal for
step 2 still holds runs the loop in the wrong order. That is trap D-01 in
`diagnostic-trap-catalog.md`.

### The four golden signals belong to step 3

Step 3 asks what each component does. The four golden signals from `SRE Ch. 6, p. 87–89`
give the answer. They are latency, traffic, errors, and saturation.
`03-examine-and-diagnose.md` maps each signal to the question it answers.

---

## 5. Step 1 sets the quality of every later step

**An effective problem report states three things** (`SRE Ch. 12, p. 172`). It states the
expected behavior. It states the actual behavior. Where possible, it states how to reproduce
the behavior. A report that omits the expected behavior is trap D-39 in
`diagnostic-trap-catalog.md`.

**Reports need a consistent form and a searchable location, such as a bug tracker**
(`SRE Ch. 12, p. 172`). Google teams use custom forms and small web applications. Each form
asks for the information that the particular system needs. The form then generates the bug
and routes it (`p. 172`).

**Open a bug for every issue, including one that arrives by email or by chat**
(`SRE Ch. 12, p. 172`). The bug creates a log of the investigation and of the remedy. A
later responder can read that log.

**Do not accept a report that names one engineer as its destination.** The book gives three
harms (`SRE Ch. 12, p. 172`). A person must transcribe the report into a bug. The report has
lower quality and stays invisible to the rest of the team. The load concentrates on the few
engineers that reporters happen to know, and not on the engineer currently on duty. This is
trap D-38 in `diagnostic-trap-catalog.md`.

**Step 1 is also the place to offer self-repair.** The book states that this point is a good
one at which to give reporters tools for self-diagnosis and self-repair of common issues
(`SRE Ch. 12, p. 172`).

---

## 6. Why the model is a loop and not a line

**The chapter states that you test repeatedly until you identify a root cause**
(`SRE Ch. 12, p. 171`). Figure 12-1 draws the return path. Step 5 returns to step 3 or to
step 4.

Three properties force the loop.

1. **A test produces a new observation.** The chapter gives two ways to test a hypothesis.
   Each way adds to the state that step 3 reads (`SRE Ch. 12, p. 171`).
2. **A treatment changes the system.** The chapter states that active treatment refines your
   understanding of the state of the system and of the possible causes (`p. 171`).
3. **A removed hypothesis rewrites the list.** Step 4 must run again on the reduced list.

| Way to test | What you do | What it returns to |
|---|---|---|
| Compare | Compare the observed state against your theory. Find confirming or disconfirming evidence. (`p. 171`) | Step 3, with a sharper question |
| Treat | Change the system in a controlled way. Observe the result. (`p. 171`) | Step 3, with a changed system |

**The parallel rule.** A repair of the proximate cause does not always have to wait for the
root cause or for the postmortem (`SRE Ch. 12, p. 171`). Run the repair and the loop at the
same time.

**The exit condition is weaker than proof.** Step 6 asks you to prove that the narrowed
cause is the actual cause. The chapter states that proof is often unavailable
(`SRE Ch. 12, p. 182`). Real systems are path dependent. A system must reach a specific
state before the fault appears. Reproduction in live production can be impossible, or it can
cost more downtime than you accept. So the loop can end at a probable cause. Name the doubt
when it does.

**One incident can have several root causes.** Repair each one (`SRE Ch. 6, p. 83`).
Footnote 72 at `SRE Ch. 12, p. 188` cites the limits of a search for a single root cause.
That is trap D-09.

---

## 7. Failure mode 1 — the system as it should be, not as it runs

**Signature.** The responder cites an architecture diagram, a design document, or a code
comment. The responder cites no value from the running process.

**Where it sits.** The book locates the pitfalls at the Triage, Examine, and Diagnose steps.
It states that the cause is often a lack of deep system understanding
(`SRE Ch. 12, p. 171`).

**Why it is fatal.** Every deduction after a wrong premise is unsound. The responder tests a
hypothesis about a system that does not exist. The tests can all pass, and the fault stays.
This is trap D-11 in `diagnostic-trap-catalog.md`.

**The remedy that the book gives.** Read the state that the running server exports
(`SRE Ch. 12, p. 175`). A Google server carries endpoints that show a sample of the calls it
recently sent and received. The chapter states the point of those endpoints. You can
understand how one server communicates with others without an architecture diagram
(`p. 175`). Some of those endpoints carry error-rate and latency histograms for each type of
call. The chapter states that they let you tell quickly what is unhealthy (`p. 175`).

**The gap is also a cause, not only a reading error.** A code change or an environment
change can represent the state of reality wrongly. The chapter states that such problems
often create the need to troubleshoot (`SRE Ch. 12, p. 186`). The remedy it gives is to
simplify, to control, and to log such changes.

**The rule.** Every hypothesis must name an observation from the running system. This is
gate G2 in `../SKILL.md`.

---

## 8. Failure mode 2 — the irrelevant symptom

**Signature.** The responder investigates a metric that moved, and no test connects that
metric to the reported fault. Or the responder reads a metric that does not mean what the
responder thinks it means.

**What the book calls it.** Pitfall 1 at `SRE Ch. 12, p. 171` names two forms. The responder
examines symptoms that are not relevant. The responder misunderstands the meaning of a
system metric. The book states the result. Wild goose chases often follow.

**Why a large system produces this trap by itself.** As a system grows, and as the number of
monitored metrics grows, events that correlate well by pure coincidence become inevitable
(`SRE Ch. 12, p. 171`). Footnote 64 at `p. 187` gives an example with no plausible mechanism.
The count of computer-science doctorates in the United States correlates with cheese
consumption per person at r² = 0.9416.

**The remedy that the book gives.** Learn the system, and learn the common patterns that
distributed systems use (`SRE Ch. 12, p. 171`). Then separate a shared cause from a real
cause. The chapter gives the physical example. Packet loss in a cluster and failed hard
drives in the same cluster share one cause, which is a power outage. Neither one causes the
other (`p. 171`).

**Two catalog entries carry the misread-metric form.** D-12 covers a mean over a bimodal
latency distribution. D-13 covers failed requests that never enter the latency metric. Read
`diagnostic-trap-catalog.md`, group C.

**The rule.** State the mechanism that connects the metric to the fault. Then run a test
that a coincidence would fail.

---

## 9. The other two pitfalls, and where all four sit

The chapter lists four pitfalls at `SRE Ch. 12, p. 171`. Section 8 covered the first one.
Section 7 covered the premise that all four share. This table gives all four.

| Pitfall (`p. 171`) | Step it damages | Remedy that the book gives | Catalog |
|---|---|---|---|
| Irrelevant symptoms, or a misread metric | Examine | Learn the system. Learn common distributed-systems patterns. | D-06, D-12, D-13 |
| A wrong idea of how to change the system, its inputs, or its environment, so a test is neither safe nor effective | Test and treat | Learn the system. Design a test with mutually exclusive alternatives. | D-26, D-27, D-29 |
| A wildly improbable theory, or the cause of a past problem reused because it happened once | Diagnose | Not all failures are equally probable. Prefer the simpler explanation. | D-07, D-08 |
| A spurious correlation that is a coincidence, or that shares a common cause | Diagnose | Correlation is not causation. Expect coincidence at scale. | D-06 |

**Horses, not zebras.** The book quotes the rule that doctors learn: "when you hear
hoofbeats, think of horses not zebras" (`SRE Ch. 12, p. 171`). Footnote 61 at `p. 187`
attributes it to Theodore Woodward at the University of Maryland School of Medicine in the
1940s.

**The stated limit of that rule.** Footnote 61 adds that some systems eliminate entire
classes of failure. It gives one example. With a well-designed cluster filesystem, a single
dead disk is an unlikely cause of a latency problem (`SRE Ch. 12, p. 187`). Your prior
depends on your design.

**Occam's Razor, and its counterweight.** Prefer the simpler explanation, all other things
being equal (`SRE Ch. 12, p. 171`). Footnote 62 at `p. 187` names Hickam's dictum as the
counterweight. A set of common low-grade problems that together explain every symptom can be
more likely than one rare problem that causes them all.

---

## 10. War stories

**Shakespeare search, and the header that removed two tiers**
(`SRE Ch. 12, p. 173, p. 175–176, p. 178`). An on-call engineer received the alert
`Shakespeare-BlackboxProbe_SearchFailure`. The black-box prober had found no search results
for five minutes. The alerting system had already filed a bug with two links. One link gave
the recent probe results. The other gave the playbook entry. About half of the once-a-minute
probes succeeded over the previous ten minutes, with no discernible pattern. The prober did
not record what the failing responses returned. Manual reproduction gave HTTP 502 with no
payload. The response carried an `X-Request-Trace` header that listed the backend servers.
The book states the inference and its strength. The response probably would not carry that
header if the request had not at least reached the search backends and failed there
(`p. 178`). That fact discounted the API frontend and the load balancers in one step.
Response provenance can exonerate whole tiers. A probe that hides the failing response costs
a manual reproduction cycle.

**App Engine, and the correlation that was not a cause** (`SRE Ch. 12, p. 182–186`). An
internal customer reported latency up by nearly an order of magnitude. Processor time and
serving-process count were both nearly four times higher. No code change and no traffic rise
explained it. The team removed change as a cause with evidence. The shift happened on a
Saturday with no application push and no production push in flight. The most recent pushes
had completed days before. The application developers then found a correlation between the
latency rise and a rise in `merge_join` datastore calls. That call normally indicates a poor
index, so the team started to build composite indices. Dapper tracing then showed that
requests for static content were also much slower. Static content never touches the
datastore, so the index theory was fatally flawed. Tracing also showed a window of about
250 ms between the start of a request and the first remote call. Dapper can only trace
remote calls, so the work in that window was invisible. The developers added their own
instrumentation and found the cause. A long-standing access-control defect created a
whitelist object on every access to one path. A security scanner produced thousands of them
in half an hour. Every later request then had to check all of them.

**Spanner, and the chain of what, where, and why** (`SRE Ch. 12, p. 177`). A Spanner cluster
showed high latency, and calls to its servers timed out. That is the symptom. The first
question asked why. The server tasks used all their processor time. They could not make
progress on the requests that clients sent. The second question asked where in the server
that time went. A profile showed the sort of entries in logs checkpointed to disk. The third
question asked where inside the sorting code. The answer was the evaluation of a regular
expression against paths to log files. The chapter names three remedies. Rewrite the
expression to avoid backtracking. Search the codebase for the same pattern. Consider RE2,
which does not backtrack and which guarantees linear runtime growth with input size
(`SRE Ch. 12, p. 177`). A malfunctioning system usually still does something. The chain of
questions locates the work.

---

## 11. What the model gives you, and what it does not

| Claim | Status |
|---|---|
| The method is learnable and teachable | Stated at `SRE Ch. 12, p. 169` |
| A systematic approach can bound the time to recovery | Stated at `SRE Ch. 12, p. 187` |
| The idealized model matches practice | Denied. The chapter states that practice is never as clean as the model (`p. 172`) |
| The loop always reaches a proven cause | Denied. Proof is often unavailable (`p. 182`) |
| One incident has one root cause | Denied. Repair every cause (`SRE Ch. 6, p. 83`) |

**The first step against a reasoning failure is to name it.** The chapter states that an
understanding of the failures in our reasoning is the first step to avoiding them
(`SRE Ch. 12, p. 172`). It then gives the method in one line. Know what you know, know what
you do not know, and know what you need to know (`p. 172`).

---

## 12. Where to read next

| Signal | File |
|---|---|
| The service is down, and you must act now | `02-triage-first-diagnose-second.md` |
| You hold observations and no cause | `03-examine-and-diagnose.md` |
| You hold two or more hypotheses | `04-test-and-treat.md` |
| The diagnosis feels certain, or it repeats a past cause | `05-negative-results-and-bias.md` |
| You write a design, or you list the telemetry you did not have | `06-making-troubleshooting-easier.md` |
| Any fault. Any review of a diagnosis. | `diagnostic-trap-catalog.md` |
| You need the gates, the tables, and the Fault Record template | `../SKILL.md` |

**Two boundaries.** Use the `reverse-branching` skill when you already know which change to
reverse. Use this skill when you do not. Use the `self-healing-apis` skill to design the
recovery that needs no person. Use this skill when a person must find the cause. `../SKILL.md`,
Section 13 holds the full table of overlaps and their owners.
