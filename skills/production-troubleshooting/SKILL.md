---
name: production-troubleshooting
description: A method that finds the cause of a production fault instead of guessing at it. The method runs six steps: problem report, triage, examine, diagnose, test and treat, and cure. Use this skill when a service is down, slow, or wrong in production. Trigger words include outage, incident, on-call, page, alert, root cause, postmortem, and blameless review. Also use it for a latency spike, an error rate rise, a timeout, or a hang. Also use it for a memory leak, a thread dump, a blocked thread, or a cascading failure. Also use it when a deploy causes a regression. The content comes from Site Reliability Engineering (Beyer, Jones, Petoff, Murphy) and Release It! (Nygard).
---

# PRODUCTION TROUBLESHOOTING

**ROLE:** You are the responder on a production fault. You find the cause.

**CORE FUNCTION:** You receive a fault report. You make the system serve as well as it can.
Then you find the cause with a test that can fail. Then you write the record.

Troubleshooting is a skill that a person learns (`SRE Ch. 12, p. 169`). A responder who
follows the method finds the cause. A responder who guesses removes the wrong change.

This skill is a **knowledge and gate skill**. It does not run a workflow. `engineer-workflow`
owns the phase order. This skill is the unscheduled entry point that a page creates, and it
hands a proven cause to `planner` as a new task.

## Table 1 — Which skill answers which question

| Skill | Question it answers | Time |
|---|---|---|
| `production-troubleshooting` (this skill) | Why is production broken now? | During the fault |
| `incident-response` | Who commands, who communicates, and what does the record hold? | During the fault |
| `reverse-branching` | Which change caused it, and how do we reverse it? | During the fault |
| `observability` | What must the system expose, and what must page a person? | Before the fault |
| `self-healing-apis` | How does the system recover without a person? | Before the fault |
| `slo-engineering` | How much unreliability may this service spend? | Before the fault |
| `capacity-engineering` | Which resource reaches its limit first? | Before the fault |
| `system-design` | What must we build? | Before the code |
| `data-systems-design` | Does the design stay correct across machines? | Before the code |
| `systems-programming` | Does the code stay correct against the kernel? | Before the code |
| `security-engineering` | Can an attacker cause this? | Any time |

- Use `incident-response` for the command structure, the roles, and the postmortem. Use this skill for the reasoning that locates the cause.
- Use `reverse-branching` when you already know which change to reverse. Use this skill when you do not.
- Use `self-healing-apis` to design the recovery that needs no person. Use this skill when a person must find the cause.
- Use `observability` to decide what the system exposes. Use this skill to read what it exposes.

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these faults:

- A user-facing service is down, in one region or everywhere.
- Latency rises. The error rate rises. Requests time out.
- A component hangs. Threads block. The processor sits idle while users wait.
- Memory use grows until the process dies. A pool never returns a resource.
- A deploy or a configuration change starts a regression.
- One fault causes a second fault, and the failure grows over time.
- An alert fires and you must decide what to do about it.
- The same alert fires again, and the previous cause is the tempting answer.

Do not use this skill for these tasks:

- A question about one log line, with no user impact.
- A design that no user runs yet. Use `system-design`.
- A change that you already know how to reverse. Use `reverse-branching`.

**A fault report without user impact is a ticket, not an incident.** The response must be
proportionate to the impact (`SRE Ch. 12, p. 173`). Section 10 gives the sizes.

---

# 2. THE TROUBLESHOOTING LOOP

The method has six named steps. Figure 12-1 names them (`SRE Ch. 12, p. 170`). Each step has
a signal that starts it. Do not start a step before its signal.

## Table 2 — The six steps and the signal for each

| Step | Signal that starts it | Action | Output |
|---|---|---|---|
| 1. Problem report | An alert fires, or a person reports a fault | Record expected behavior, actual behavior, and how to reproduce it (`p. 172`) | A ticket in a searchable queue |
| 2. Triage | The report states the user impact | Make the system serve as well as it can. Capture evidence first. (`p. 173`) | A mitigation and a saved evidence set |
| 3. Examine | The system serves again, or the impact is bounded | Read metrics, logs, traces, and exposed state (`p. 174–175`) | A list of components that behave wrongly |
| 4. Diagnose | You hold observations and no cause | Apply one technique from Table 6 (`p. 176–178`) | A short list of hypotheses |
| 5. Test and treat | You hold two or more hypotheses | Run the test that separates them (`p. 179–180`) | One hypothesis kept. The others removed. |
| 6. Cure | One cause explains every observation | Fix the cause. Add the test that catches it. Write the record. (`p. 182`) | A Fault Record |

- **The loop rule.** Step 5 returns to step 3 or to step 4. Repeat until one cause holds (`SRE Ch. 12, p. 171`). The model is a loop, not a line.
- **The parallel rule.** A fix for the proximate cause does not wait for the root cause (`SRE Ch. 12, p. 171`).

## The eight gates

These gates are mandatory. Check each one before you call the record complete.

| Gate | Rule | Source |
|---|---|---|
| G1 Triage gate | Mitigate before you diagnose. Capture the evidence that the mitigation destroys, first. | `SRE Ch. 12, p. 173` |
| G2 Evidence gate | Every hypothesis names an observation from the running system, not a design document. | `SRE Ch. 12, p. 175` |
| G3 Change window gate | List every change in the window, across code, configuration, infrastructure, dependency, and demand. State when no change is in the window. | `SRE Ch. 22, p. 335`. `SRE Ch. 12, p. 184` |
| G4 Test gate | Every hypothesis gets a test that can fail. Run it from the caller's vantage point. Order the tests by likelihood and by risk. | `SRE Ch. 12, p. 179` |
| G5 Negative result gate | Record and publish every test that removed a hypothesis. | `SRE Ch. 12, p. 180–182` |
| G6 Trap scan gate | Run the D- catalog before you accept a cause. Report only the traps that apply. | This skill |
| G7 Cure gate | The fix ships with the test or the alert that catches the fault. Every other root cause gets its own repair. | `SRE Ch. 6, p. 83`. `SRE Ch. 12, p. 182` |
| G8 Proportionality gate | Match the output to the impact. "Not applicable" is a valid result. | `SRE Ch. 12, p. 173` |

Deep dive: `references/01-the-troubleshooting-model.md`.

---

# 3. TRIAGE COMES BEFORE DIAGNOSIS

**Stop the bleeding first.** Your first instinct in a major outage is to search for the root
cause. Ignore that instinct (`SRE Ch. 12, p. 173`). A correct diagnosis helps no user while
the system stays down. **Capture the evidence that the mitigation will destroy, first.**
Rapid triage does not remove the duty to preserve the logs (`SRE Ch. 12, p. 173`).

## Table 3 — Impact to response

`incident-response` owns the declaration gate, the response classes, and the four roles. This
table is the short form that a lone responder needs before that skill starts. Read
`incident-response` Section 2 for the full gate.

| Impact | Who responds | Page a person? | Protocol |
|---|---|---|---|
| One user. A workaround exists. | One engineer, in business hours | No | Ticket only |
| One feature is degraded for many users | The on-call engineer | Yes | Incident ticket |
| A user-facing service is down in one region | The on-call engineer and the service owner | Yes | Incident ticket, plus a status update |
| A global outage | The formal incident protocol. Call more people. | Yes | Formal incident management |
| Data loss or data corruption | The formal incident protocol. Freeze first. | Yes | Formal incident management |

- **The proportion rule.** The response must be proportionate to the impact (`SRE Ch. 12, p. 173`).
- **The protocol rule.** Adopt the formal protocol when the fault crosses teams, or when you cannot state an upper bound for its duration (`SRE Ch. 11, p. 165`).
- **The staffing rule.** If you feel overwhelmed, call more people. It can be correct to page the whole company (`SRE Ch. 13, p. 189`).
- **The state rule.** The person who triggered the event holds the most state. Use that person (`SRE Ch. 13, p. 197`).
- **The budget rule.** A network outage or a datacenter failure eats into the error budget of the service (`SRE Ch. 3, p. 60`). State the minutes in the record. `slo-engineering` owns the budget and its policy.

## Table 4 — Mitigation moves, and the evidence each one destroys

| Signal | Move | Evidence the move destroys | Capture this first |
|---|---|---|---|
| One cluster is unhealthy. Others are healthy. | Send the traffic to the healthy clusters (`SRE Ch. 12, p. 173`) | The live state of the unhealthy cluster | Thread dump, queue depth, memory use |
| Every replica crash-loops | Admit about 1% of the traffic. Raise it slowly. (`SRE Ch. 22, p. 340`) | Nothing | Crash log, restart count, start time |
| The fault started at a deploy | Reverse the change (`SRE Ch. 22, p. 335`) | The running version and its configuration | Version id, configuration snapshot, deploy time |
| One dependency is saturated | Throttle the pool for that dependency (`Release It! Sec. 16.9, p. 262`) | The blocked-thread evidence | Thread dump on the caller and on the callee |
| Threads are busy and processor use is low | Take a thread dump. Then restart the component. (`Release It! Sec. 16.7, p. 259`) | The blocked stacks | Thread dumps from several nodes |
| Batch work runs on the serving path | Stop the batch work (`SRE Ch. 22, p. 341`) | Nothing | Job names and start times |
| One request shape crashes the process | Block that request shape (`SRE Ch. 22, p. 341`) | The crashing request | A copy of the request |
| New sessions exceed the capacity | Throttle new sessions at the edge (`Release It! Sec. 7.6, p. 158`) | The demand profile | Session count over time, source addresses |
| The fault writes wrong data | Freeze the system (`SRE Ch. 12, p. 173`) | Throughput evidence | The wrong records and their identifiers |
| Automation causes the damage | Disable all team automation (`SRE Ch. 13, p. 195`) | The automation's next action | The command it sent and its target set |

- **Name the source before you restart a server.** A restart can amplify a cold-cache fault (`SRE Ch. 22, p. 340`).
- **Test the move on one node first.** Then apply it to the fleet (`Release It! Sec. 16.10, p. 262–263`).
- **Read the verdict from the fleet, not from the one node.** One repaired node behind a load balancer receives all the traffic. It can fail for that reason alone (`Release It! Sec. 16.10, p. 263`).
- **Modern:** a container orchestrator that restarts unhealthy pods behaves like the cluster scheduler that restarts unhealthy tasks in `SRE Ch. 22, p. 339`. It can also drive the task count down during an outage.

Deep dive: `references/02-triage-first-diagnose-second.md`.

---

# 4. EXAMINE

Examine means one thing in this skill. You observe what each component does, and you decide
whether that behavior is correct (`SRE Ch. 12, p. 174`). Ask four questions of every suspect
component. The last column names the trap that applies when the telemetry is absent. The
trap codes live in `references/diagnostic-trap-catalog.md`.

## Table 5 — Four questions and the telemetry that answers each

| Question | Telemetry that answers it | Trap if it is absent |
|---|---|---|
| What does the component serve? | Traffic rate. Error rate by class. Latency histogram. Aborted request count. | D-12, D-13 |
| What does it ask of its dependencies? | Per-dependency call count, latency, timeout count, error class, and endpoint address | D-14, D-18 |
| What does it consume? | Processor, memory, threads busy over five seconds, queue depth, pool checkouts, high-water mark, file descriptors | D-17 |
| What does the user see? | An external probe from the user vantage point. The probe stores the failing body. | D-15, D-20 |

- **The four questions collect the four golden signals.** They are latency, traffic, errors, and saturation (`SRE Ch. 6, p. 88`). `observability` owns the signal definitions, the error taxonomy, and the histogram rule. Read its Section 3. This skill reads what those signals say during a fault.
- **The two-sided rule.** Instrument both sides of every boundary. Without both sides you cannot separate a slow dependency from a slow network (`SRE Ch. 6, p. 87`).
- **The component-up rule.** Every component can report healthy while the user sees a fault. This happens with blocked threads and with cascading failure (`Release It! Sec. 17.5, p. 287`).
- **The state endpoint.** A server endpoint that lists its recent calls tells you how the server communicates. You need no architecture diagram (`SRE Ch. 12, p. 175`).
- **Per-dependency telemetry.** For each integration point read the Circuit Breaker state, the timeout count, and the request count. Read also the average response time, the error counts by class, and the address of the remote endpoint (`Release It! Sec. 17.6, p. 298`). `self-healing-apis` owns the design of the Circuit Breaker. This skill reads its state.

Deep dive: `references/03-examine-and-diagnose.md`.

---

# 5. DIAGNOSE

Diagnose means one thing in this skill. You convert observations into a short list of
hypotheses. Section 6 confirms one. Every page-only citation below comes from `SRE Ch. 12`.

## Table 6 — How to choose the technique

| Signal | Technique | Cost |
|---|---|---|
| The stack has few layers and known interfaces | Divide and conquer. Walk the stack from one end to the other. (`p. 176`) | Linear in the layer count |
| The stack is large. A walk is too slow. | Bisection. Split the system. Test the boundary. Repeat. (`p. 176`) | Logarithmic in the component count |
| Components have defined inputs and outputs | Simplify and reduce. Inject known data at each hop. Check the output. (`p. 176`) | You must build a test input |
| The system does something, but not the wanted thing | Ask what, then where, then why (`p. 176–177`) | Cheap. It often needs a profiler. |
| The fault started at a point in time | What touched it last. Read the change log. (`p. 177`) | It needs a change log. See D-31. |
| This service faults often | Build a diagnostic tool for this service (`p. 178`) | High one time. Low after that. |
| Every component reports healthy | Read the user path from outside (`SRE Ch. 10, p. 156`) | It needs an external probe |

- **The inertia rule.** A working system stays working until an external force acts on it. That force is often a configuration change or a shift in the load (`SRE Ch. 12, p. 177`).
- **The counter-rule.** The change log must state "no change is in the window" as confidently as it states "this change is". The App Engine team removed change as a cause because no push was in flight (`SRE Ch. 12, p. 184`).
- **The bisection example.** A response header that lists the backends which served the request lets you discount the frontend and the load balancers in one step (`SRE Ch. 12, p. 176, p. 178`). The book states the strength of that step. The response probably would not carry the header if the request had not reached the backends.
- **Prefer the common cause.** When you hear hoofbeats, think of horses, not zebras (`SRE Ch. 12, p. 171`). Note the counterweight. A set of common low-grade faults can together explain the symptoms better than one rare fault (`SRE Ch. 12, p. 187`).

## Table 7 — Symptom to first hypothesis

Each row states a hypothesis, not a cause. Test the hypothesis with Section 6.

| Observation | First hypothesis | Source |
|---|---|---|
| Threads are busy. Processor use is low. | A caller blocks on a dependency, or on a pool with no timeout | `Release It! Sec. 16.6–16.7, p. 258–259` |
| Mean latency is normal. The frontend error rate is high. | A small share of requests hangs for the full deadline | `SRE Ch. 22, p. 330–331` |
| Latency rises after a restart or after a new cluster starts | A cold cache that the service needs for its capacity | `SRE Ch. 22, p. 331–333` |
| The error rate rises again after the load falls | Retries amplify the load | `SRE Ch. 22, p. 325–327` |
| Successful throughput falls below the rate before the fault | Overload after a failover moved the load | `SRE Ch. 22, p. 314–316` |
| Latency rises with no code change and no traffic rise | Per-request work that grows with the stored data | `SRE Ch. 12, p. 185–186` |
| Processor use is at 100% on one small dependency | That dependency is the constraint. Every tier above it queues. | `Release It! Sec. 16.8, p. 261` |
| Sessions rise faster than users | The traffic is not a user | `Release It! Sec. 7.4, p. 155–157` |
| A metric correlates with the fault and has a plausible mechanism | Test the correlation before you build the fix | `SRE Ch. 12, p. 184–185` |
| Throughput is flat and the processor waits | Contention for a pooled resource | `Release It! Sec. 9.1, p. 176–179` |
| Health checks fail and the process still runs | Overload starves the health check path | `SRE Ch. 22, p. 317, p. 339` |
| One remote call is slow only from one region | A chatty protocol. Count the calls per user action. | `Release It! Sec. 9.9, p. 200` |

---

# 6. TEST AND TREAT

**A hypothesis without a test that can fail is not a hypothesis.** It is an opinion. G4
rejects it. You test in one of two ways (`SRE Ch. 12, p. 171`). You compare the observed
state against the hypothesis, or you treat the system and observe the result.

## Table 8 — Test design

| Rule | The question you must answer before you run the test |
|---|---|
| Order the tests by likelihood, then by risk to the system | Which cheap test removes the largest group of hypotheses? |
| The alternatives must be mutually exclusive | Does a pass remove one group and a failure remove the other group? |
| Run the test from the caller's vantage point | Does a firewall rule or a route change the result? |
| Watch for a side effect that changes a later test | Does this test alter the thing that I measure? |
| Record every change that you make | Can I return the system to the state before the test? |
| Accept a suggestive result when a race resists reproduction | Can this fault be reproduced on demand? |

Source for Table 8: `SRE Ch. 12, p. 179–180`.

- **The worked example.** Two hypotheses compete. The network to the database failed, or the database refuses connections. Connect to the database with the application server's credentials. Send an echo request to the database host. The first test removes the second hypothesis. The second test removes the first hypothesis. Network topology and firewall rules can confound both (`SRE Ch. 12, p. 179`).
- **A test is a change.** Verbose logging, an added processor, and a cleared cache all alter the system. Record each one. Reverse each one (`SRE Ch. 12, p. 180`).

Deep dive: `references/04-test-and-treat.md`.

---

# 7. THE TRAP SCAN

**G6 is mandatory. Run the trap catalog before you accept a cause.** The catalog is
`references/diagnostic-trap-catalog.md`. It holds 40 named traps in 8 groups. Each entry
carries a severity, a signature, a consequence, and a fix. Each signature names something
that you can search for in a runbook, a dashboard, a log, a ticket, or an incident timeline.

Report the scan in this form:

```md
### Trap Scan
| Trap | Present | Evidence | Required fix |
|---|---|---|---|
| D-12 Mean latency over a bimodal distribution | Yes | dashboard `svc-api` has no p99 panel | add latency buckets |
| D-31 The change log covers code only | No | the change log holds config and vendor windows | — |
```

**List only the traps that apply.** "Not applicable" is a valid scan result, and it is the
preferred result. An invented finding costs the team more than a short scan.

The catalog serves two other skills. `code-review` runs it as a diagnosability pass.
`verify` runs it against the observability that a phase specified. Each owns its own format.

---

# 8. CURE AND THE RECORD

Cure means one thing in this skill. You prove the cause as far as production permits, you
repair it, and you write the record (`SRE Ch. 12, p. 182`).

- **The cure includes the test or the alert that would have caught the fault.** A fix without that check invites the same incident again (G7).
- **Why proof is often unavailable.** Real systems are path dependent. They must reach a specific state before the fault appears. Reproduction in live production can be impossible or unacceptable (`SRE Ch. 12, p. 182`).
- **The multiplicity rule.** One incident can have several root causes. Repair each one (`SRE Ch. 6, p. 83`).

## Table 9 — Proof strength at the cure step

| Evidence you hold | The claim you may write |
|---|---|
| The fault reproduces, and the fix removes it | Proven cause |
| The fix removes the fault. No reproduction exists. | Probable cause. Name the remaining doubt. |
| Several factors, and no single factor is sufficient | Causal factors. List all of them. |
| The fault needs a specific prior state that you cannot rebuild | Probable cause, plus the state that the fault needed |
| The fault stopped and nobody changed anything | No cause. Say so. Keep the ticket open. |

Source for Table 9: `SRE Ch. 12, p. 182`.

## The Fault Record

The Fault Record is the artifact that this skill produces. It is a working document during
the incident, and it becomes the input to the postmortem. It is not the postmortem.

```md
## Fault Record: [service or symptom]

### 1. Problem report
- Alert name or reporter:
- Expected behavior:
- Actual behavior:
- How to reproduce (or "no reproduction"):
- First seen at:      Detected at:      Ticket:
> Source: SRE Ch. 12, p. 172

### 2. Impact
- Who is affected, and how many:
- Which user journey fails:
- Severity, from Table 3:
- Data at risk: [none / wrong values / lost records]

### 3. Triage
| Time | Move | Effect | Evidence captured before it |
|---|---|---|---|
|  |  |  |  |
- The system now serves at: [normal / degraded, describe / down]
> Rule: stop the bleeding first. SRE Ch. 12, p. 173

### 4. Change window
List every change between the last known good time and the fault start time.
| Change | Layer | Time | In the window? | Tested out? |
|---|---|---|---|---|
|  | code / config / infrastructure / dependency / demand |  |  |  |
- If no change is in the window, write that sentence. SRE Ch. 12, p. 184
> Hand a revert decision to `reverse-branching`. Do not draft the revert here.

### 5. Observations
| Question | What the telemetry says | Where you read it |
|---|---|---|
| What does it serve? |  |  |
| What does it ask of its dependencies? |  |  |
| What does it consume? |  |  |
| What does the user see? |  |  |
> Source: SRE Ch. 12, p. 174–175

### 6. Hypotheses and tests
| # | Hypothesis | Test | Vantage point | Risk | Result |
|---|---|---|---|---|---|
| H1 |  |  |  | low / medium / high | kept / removed |
- Record every removed hypothesis. A negative result is a result. SRE Ch. 12, p. 180

### 7. Trap scan
| Trap | Present | Evidence | Required fix |
|---|---|---|---|
| D-12 Mean latency over a bimodal distribution | Yes | dashboard `svc-api` has no p99 panel | add latency buckets |
- List only the traps that apply. "Not applicable" is a valid result.

### 8. Cause
- Claim strength, from Table 9: [proven / probable / causal factors / none]
- Cause or causal factors:
- The state that the fault needed:
- What still does not fit:

### 9. Cure
| Action | Owner | The check that catches it next time |
|---|---|---|
| Fix the cause |  | test or alert |
| Repair each other root cause |  |  |
| Remove the temporary mitigation |  | expiry date |
> A temporary fix with no expiry date becomes permanent architecture.
> Release It! Sec. 7.6, p. 160

### 10. What we could not see
- Telemetry that was missing, and the trap code for it:
- The one signal that would have shortened this incident most:
> Hand these items to `detail-planning` as observability requirements.
```

- **Write the record during the incident, not after it.** A shared document gives the timestamps that the postmortem needs, and it keeps others informed without an interruption (`SRE Ch. 12, p. 180`).
- **Every action in Section 9 needs an owner.** Hold yourself and others accountable to the specific actions in the record (`SRE Ch. 13, p. 198`).
- **Short form.** For a fault with one affected user, write Sections 1, 5, 6, and 8 only.

---

# 9. NEGATIVE RESULTS

**A test that removed a hypothesis is a result. Record it.** A negative result is conclusive.
It states something certain about production or about the limits of the system
(`SRE Ch. 12, p. 180–181`).

- **Keep the tool.** The tool that produced the negative result outlives the experiment. Keep the load generator and the probe (`SRE Ch. 12, p. 181`).
- **Publish the result.** A rejected approach saves the next team the same days (`SRE Ch. 12, p. 182`).
- **Distrust an incident document that names no failed test.** Such a document is either filtered, or the author was not rigorous (`SRE Ch. 12, p. 182`).
- **A negative change window counts.** "No change is in the window" is a negative result, and it is as strong as a named change (`SRE Ch. 12, p. 184`).

Deep dive: `references/05-negative-results-and-bias.md`.

---

# 10. HOW MUCH OUTPUT EACH FAULT NEEDS

Rigor must scale with the user impact. A full record for a log-line question is a fault.

## Table 10 — Fault class to required output

| Fault class | Required output |
|---|---|
| A question about a log line. No user impact. | Nothing from this skill |
| One user. A workaround exists. | Steps 1, 3, and 4. A short Fault Record. |
| One degraded feature | The full loop. A Fault Record. A trap scan of groups A, C, and E. |
| A user-facing outage | The full loop. The full trap scan. A postmortem. |
| Data loss or corruption | Freeze first. The full loop. The full trap scan. A postmortem. Preserve the evidence. |
| A repeat of a fault that has a Fault Record | Read the record first. Then run steps 5 and 6 only. |

**The silence rule. This skill produces nothing for a change that has not reached production
and that no user can see.** "Not applicable" is a valid result. Say it plainly, and continue
with the task.

**Toil.** A repeat fault that needs a manual repair each time is toil (`SRE Ch. 5, p. 75–76`).
SRE caps purely operational work at 50% of an engineer's time (`SRE Ch. 11, p. 160`). When you
cannot remove the cause of a frequent alert, the response deserves full automation
(`SRE Ch. 6, p. 93`). `slo-engineering` owns the toil budget. `incident-response` owns the
pager load.

---

# 11. LANGUAGE DISCIPLINE

These phrases are forbidden in a Fault Record and in an incident channel. Each replacement
names an observation, not an opinion.

## Table 11 — Forbidden phrase to required replacement

| Forbidden phrase | Required replacement |
|---|---|
| "it looks like a network problem" | The observation on each side of the boundary, and the test that separates them |
| "the deploy broke it" | The change identifier, the deploy time, the fault start time, and the test that keeps or removes the change |
| "we restarted it and it went away" | What you captured before the restart, and the cause that the restart is consistent with |
| "the metrics correlate" | The mechanism that connects them, and the test that removes coincidence |
| "the root cause is the retry storm" | Whether retries started the fault or amplified it (`SRE Ch. 22, p. 327`) |
| "it should not do that" | What the running system does, read from telemetry |
| "it is intermittent" | The share of requests that fail, over which window, from which vantage point |
| "we fixed it" | The cause, the fix, and the test or alert that catches the next occurrence |
| "no root cause found" | The hypotheses that you removed, and the telemetry that you did not have |

**Modern.** An LLM coding agent can propose a cause from a log and a difference. Neither book
covers such an agent. Treat its output as a hypothesis. G4 applies to it without change. The
agent must name the test that can fail.

---

# 12. GLOSSARY

This skill uses one word for one meaning. Use these words in your output.

| Word | Meaning in this skill |
|---|---|
| **examine** | To observe what a component does, and to decide whether that behavior is correct |
| **diagnose** | To convert observations into a short list of hypotheses |
| **test** | To run a check whose result keeps one hypothesis and removes another |
| **treat** | To change the system in a controlled way, in order to test a hypothesis |
| **mitigate** | To make the system serve as well as it can, before you know the cause |
| **cure** | To repair the cause, to add the check that catches it, and to write the record |
| **cause** | A defect that, when repaired, gives confidence that the event does not recur in the same way (`SRE Ch. 6, p. 83`) |
| **factor** | A condition that the fault needed, and that alone is not sufficient |
| **trap** | A named defect in the diagnosis, in the telemetry, or in the record |

---

# 13. REFERENCE INDEX

| Reference | Content | Read it when |
|---|---|---|
| `references/01-the-troubleshooting-model.md` | The six steps, the two ways to test a hypothesis, the four common pitfalls, the problem report | You start any fault, or a responder guesses |
| `references/02-triage-first-diagnose-second.md` | Mitigation moves, evidence capture, the three emergency classes, the incident protocol | The service is down and you must act now |
| `references/03-examine-and-diagnose.md` | Logging, exposed state, tracing, the four diagnosis techniques, the vital signs of a hang | You hold observations and no cause |
| `references/04-test-and-treat.md` | Test design, confounding factors, side effects, notes, the single-node caution | You hold two or more hypotheses |
| `references/05-negative-results-and-bias.md` | Correlation, confirmation bias, base rates, several root causes, publication | The diagnosis feels certain, or it repeats a past cause |
| `references/06-making-troubleshooting-easier.md` | Observability at design time, log format, message codes, the operations database | You write a design, or you list the telemetry that was missing |
| `references/diagnostic-trap-catalog.md` | 40 named traps with a signature, a consequence, and a fix. It also has an index by symptom. | Every fault. Every review of a diagnosis. |

## Overlap ownership

One skill owns each overlap. This skill does not restate what another skill owns.

| Skill | The overlap | Owner | What this skill does instead |
|---|---|---|---|
| `engineer-workflow` | Phase order and state in `plan.md` | `engineer-workflow` | This skill is an unscheduled entry point. It hands a proven cause to `planner` as a new task. |
| `planner` | Root-cause analysis before a plan | Split. This skill owns finding the cause in production. `planner` owns turning the cause into a phased plan. | Produce the Fault Record. `planner` reads Section 8 and Section 9 of it. |
| `detail-planning` | Observability specification and rollback specification | `detail-planning` | Produce Section 10 of the Fault Record as a requirement list. Do not write the spec. |
| `implement` | Timeouts, jittered retries, idempotency keys, bounded queues | `implement` | Name their absence as a trap. Read `implement` Section 3 for the fix. |
| `verify` | Checking that specified observability exists | `verify` | Supply the D- catalog as the source for the missing-observability check. |
| `code-review` | Severity-ranked findings on a difference | `code-review` | Supply the D- catalog for a diagnosability pass. `code-review` owns the report format and the verdict. |
| `system-design` | Capacity estimation and the constraint | `system-design` | Use the constraint method to localize the fault. Read `system-design/references/02-estimation.md`. |
| `data-systems-design` | Concurrency and replication anomalies | `data-systems-design` | Hand over when the cause is a data anomaly. Its `data-systems-design/references/hazard-catalog.md` continues the diagnosis. |
| `systems-programming` | Faults at the kernel boundary on one machine | `systems-programming` | Hand over when the cause is a blocked call, a short write, or a lost durable write. Use its index by symptom. |
| `security-engineering` | An attack that presents as a capacity fault | `security-engineering` | Hand over when the trigger is untrusted input or a privilege change. Do not diagnose an attack as load. |
| `reverse-branching` | Reversing a change | `reverse-branching` | Produce the change window in Fault Record Section 4, and the decision to reverse. It owns the revert mechanism, the canary, and the safety checks. |
| `self-healing-apis` | Per-dependency telemetry, timeouts, breakers, bulkheads, shedding, and automatic remediation | `self-healing-apis` | Read the telemetry that it requires. This skill owns the human loop that localizes the fault. |
| `incident-response` | Declaration, roles, the command post, handoff, the postmortem, pager load | `incident-response` | Supply the Fault Record as the diagnosis section of its Incident Record. Do not restate the roles or the postmortem template. |
| `observability` | The four golden signals, the log contract, what to expose, alert shape, the operations database | `observability` | Read the exposed signals during a fault. Hand every gap to it as a requirement. Do not restate its tables. |
| `slo-engineering` | The error budget, the burn rate, the toil budget | `slo-engineering` | State the downtime minutes in the record. It decides what those minutes cost. |
| `capacity-engineering` | The constraint, load tests, pool sizing, headroom | `capacity-engineering` | Name the saturated resource. It sizes the fix. |

**Where the telemetry gap goes.** `observability` owns the signal design and the alert rule.
`detail-planning` owns the observability specification for one phase. Section 10 of the Fault
Record is the requirement list that both of them read. Section 4 of this file holds the four
examine questions.

---

**Attribution:** the method, the terminology, the thresholds, and the page references in this
skill and in its references come from two books. The first is *Site Reliability Engineering:
How Google Runs Production Systems* by Betsy Beyer, Chris Jones, Jennifer Petoff, and Niall
Richard Murphy (O'Reilly Media, 2016). The second is *Release It! Design and Deploy
Production-Ready Software* by Michael T. Nygard (Pragmatic Bookshelf, 2007). Page numbers refer
to those editions. Content that either book predates carries the tag **Modern**, and this skill
never attributes such content to a book.
