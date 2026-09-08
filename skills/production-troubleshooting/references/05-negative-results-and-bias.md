# Negative Results and the Traps

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 12, "Test and Treat", "Negative Results Are Magic" and "Cure", p. 179–186, with the
reasoning pitfalls at p. 171 and the footnotes at p. 187. Ch. 11, "Feeling Safe",
p. 164–166. Ch. 6, p. 83 and p. 88. *Release It!* — Michael T. Nygard (Pragmatic Bookshelf,
2007), Sec. 17.4, p. 281–283. Randall Bosetti wrote the sidebar "Negative Results Are Magic"
(`SRE Ch. 12, p. 180`). `incident-response` owns `SRE Ch. 15`, the postmortem chapter.

**Read this file when the diagnosis feels certain.** Read it when the newest alert reuses the
cause of the last alert. Read it when a metric moves with the fault and a fix is already in
progress. Certainty during an outage is the signal. An understanding of the failures in our
reasoning is the first step to avoiding them (`SRE Ch. 12, p. 172`). This file names those
failures. It also states the value of the result that you want least, which is the test that
removed your idea.

---

## 1. A negative result is a result

**Definition.** A negative result is an outcome in which the expected effect is absent
(`SRE Ch. 12, p. 180`). Any test that does not work as planned produces one. The book
includes a new design, a heuristic, and a human process that fails to improve on the thing
it replaces.

**Do not ignore a negative result and do not discount it** (`SRE Ch. 12, p. 180`). A clear
negative result can resolve some of the hardest design questions. A team often holds two
reasonable designs, and progress on one of them stalls against vague questions about the
other one.

| Claim from the sidebar | What the result gives the next engineer | Page |
|---|---|---|
| The result is conclusive | Something certain about production, about the design space, or about a performance limit of a system that exists | `p. 180` |
| The supplementary data helps others | A microbenchmark, a documented antipattern, or a project postmortem | `p. 181` |
| The tools and the methods outlive the experiment | A load generator or a benchmark tool that the next team runs | `p. 181` |
| Publication reduces the bias in our metrics | An example of how to accept uncertainty | `p. 181` |
| Publication saves a repeat | The next team does not design and run the same test | `p. 181` |

**Scope the result when you design the test** (`SRE Ch. 12, p. 181`). A broad negative result
helps more peers than a narrow one. Consider the scope before you run the test.

---

## 2. Two war stories from the sidebar

**The web server that held 800 connections** (`SRE Ch. 12, p. 180`). A development team
decided against a web server. It handled only about 800 connections out of the 8,000 that
the team needed, and it then failed because of lock contention. The team recorded that number
and that cause. A later team evaluated web servers and did not start from nothing. It read
the documented negative result and decided quickly. It needed fewer than 800 connections, or
the lock contention was repaired. One recorded number removed a week of work for a team that
the first team never met.

**The tool that outlived the experiment** (`SRE Ch. 12, p. 181`). Many webmasters have gained
from Apache Bench, a web server load test. The book states that its first results were likely
disappointing. The experiment that produced the tool did not confirm what its author
expected, and the tool stayed. Write a script that applies a configuration change. The
application that you build now may not gain from a database on solid-state disks. The next
one may gain. The script stops you from missing the same optimization then.

---

## 3. Publish the negative result

**The publication rule.** Tell everybody about the designs, the algorithms, and the team
workflows that you removed (`SRE Ch. 12, p. 182`). Publish the result that surprised you
first, so that other people are not surprised. The book includes your future self.

**The skepticism rule.** Distrust a design document, a performance review, or an essay that
does not mention failure (`SRE Ch. 12, p. 182`). The book gives two readings of such a
document. It is filtered too heavily, or the author was not rigorous.

**Why publication is rare.** It is tempting to omit a negative result, because it is easy to
perceive that the experiment failed (`SRE Ch. 12, p. 181`). Many results go unreported
because people believe wrongly that a negative result is not progress.

| Signal | Where the negative result goes | Who reads it |
|---|---|---|
| A test removed a hypothesis during an incident | Section 6 of the Fault Record in `../SKILL.md` | The next responder on this service |
| A mitigation did not change the symptom | The Fault Record, and the live incident document | Every responder on the incident |
| A design was considered and dropped | The design document, in a named section | The team that revisits the design |
| A reverted change did not cause the fault | The change window table, Section 4 of the Fault Record | `reverse-branching`, and the change author |
| A performance idea did not improve the number | A microbenchmark with the numbers and the method | Any team that evaluates the same component |
| A whole incident closed with no cause | The postmortem, with the hypotheses that you removed | The organization repository. `incident-response` owns it. |

**Gate G5 in `../SKILL.md` is this rule.** Record and publish every test that removed a
hypothesis. Trap `D-40` names the absent record. A reverted change that did not help is a
negative result, not an embarrassment. Record which revert you attempted, and state that it
did not remove the symptom.

---

## 4. Confirmation bias in an outage

**Two ways of thinking exist** (`SRE Ch. 11, p. 164`). The first is intuitive, automatic and
rapid. The second is rational, focused and deliberate. For an outage in a complex system,
the book states that the second one is more likely to produce better results.

**Stress moves a responder to the first mode.** Cortisol and corticotropin-releasing hormone
cause behavioral consequences, including fear, that impair cognitive function and cause
suboptimal decisions (`SRE Ch. 11, p. 164`). The deliberate approach is then subsumed by
immediate and unconsidered action. That action is the abuse of a heuristic.

**War story — the fourth page in one week.** The same alert pages for the fourth time in a
week (`SRE Ch. 11, p. 165`). An external infrastructure system caused the previous three
pages. The book states that it is extremely tempting to exercise confirmation bias and to
associate this fourth occurrence with the previous cause automatically. The responder then
spends the incident on a line of reasoning that was wrong from the start. Nothing in the
story says the fourth cause is different. The story says only that you did not test it.

**The two named costs of intuition** (`SRE Ch. 11, p. 165`). Intuition can be wrong, and it
is often less supportable by obvious data. A quick reaction is deep-rooted in habit, and a
habitual response is unconsidered, which the book calls disastrous. So take a step at the
desired pace when enough data is available for a reasonable decision. Examine your
assumptions critically at the same time. The book asks for both, not for one.

| Signal that bias is active | Counter-move | Trap |
|---|---|---|
| The same alert name fired three or more times this week | Treat the previous cause as one hypothesis. Give it a test. | `D-07` |
| The first hypothesis arrived before the first observation | Return to Examine. Read the running system first. | `D-11` |
| The responder cites a past incident and no telemetry | Ask for the observation in this incident that supports the claim. | `D-11` |
| The responder feels overwhelmed | Call more people (`SRE Ch. 13, p. 189`). Escalate. | — |
| No test in the record can fail | Rewrite the hypothesis until a result can remove it. | `D-28` |
| A coding agent proposed a cause | Treat the output as a hypothesis. Gate G4 applies to it. **Modern** | — |

**The three most important on-call resources** are clear escalation paths, well-defined
incident-management procedures, and a blameless postmortem culture (`SRE Ch. 11, p. 165`).
All three reduce the stress that produces the bias. Escalation for an outage with significant
unknown dimensions is a principled reaction, not a defeat. `incident-response` owns all three.
This file keeps only the effect that stress has on a diagnosis.

---

## 5. Correlation is not causation

**The rule.** Remember that correlation is not causation (`SRE Ch. 12, p. 171`). The chapter
gives one example of a shared cause. Packet loss inside a cluster correlates with failed hard
drives in that cluster. A power outage caused both. Network failure does not cause a disk
failure, and a disk failure does not cause packet loss.

**Coincidence grows with the metric count.** Systems grow in size and complexity, and teams
monitor more metrics. Events that correlate well by pure coincidence are then inevitable
(`SRE Ch. 12, p. 171`). Footnote 64 at `p. 187` gives the book's example. The count of
computer science doctorates awarded in the United States correlates with cheese consumption
per person at r² = 0.9416 between 2000 and 2009. No plausible theory explains it.

| Reading of a correlation | What it claims | The test that separates it |
|---|---|---|
| A causes B | A mechanism connects them in one direction | Remove A. B must stop. |
| A and B share one cause | A third factor drives both | Find a case where A moves and B does not |
| Coincidence | Nothing connects them | Read a path that touches B and never touches A |

**The gate.** State the mechanism that connects the two metrics before you build a fix. Then
run a test that a coincidence would fail. Trap `D-06` names this failure.

**Separate the term that starts a fault from the term that grows it.** A retry graph can
indicate bad retry behavior, and retries can also be a compounding cause rather than the
origin (`SRE Ch. 22, p. 327`). Trap `D-10` names the amplifier reported as the origin.

---

## 6. Two war stories about a false cause

**The correlation that was fatally flawed** (`SRE Ch. 12, p. 184–185`). App Engine developers
found a correlation between a latency rise and a rise in `merge_join` datastore calls. That
call often indicates poor indexing when a service reads from the datastore. The mechanism was
plausible, so the team started to design composite indices on the properties that the
application used. Dapper tracing then showed that requests for static content were also much
slower, and static content never touches the datastore. That single observation exposed the
correlation as spurious and the indexing theory as fatally flawed. The lesson is not that the
team was careless. A plausible mechanism is not a test, and one path that excludes the
suspected component removes the theory in one step.

**Voodoo Operations** (`Release It! Sec. 17.4, p. 281–283`). An administrator's pager fired,
and she immediately started a database failover on the production server. The message read
that a data channel lifetime limit was reached and that a reset was required. The author of
that message recognized it. It was a debug message about an encrypted channel to an outside
vendor. It had nothing to do with the database. The application reset the channel itself
right after it emitted the message. He traced the practice to a system failure about six
months earlier. That message was the last line logged before the Sybase server stopped. The
book states the point exactly. There was no causal connection, but there was a temporal
connection. That connection, plus an ambiguous message, produced weekly database failovers
during peak hours for six months. The sidebar also names the reason. Humans hold a natural
bias toward the detection of patterns that are not there. For early humans the cost of a
false positive was low, and the cost of a false negative was high.

---

## 7. The most recent change, and the anchor it creates

**The inertia rule.** A working computer system tends to remain in motion until an external
force acts on it (`SRE Ch. 12, p. 177`). That force is often a configuration change or a
shift in the type of load served. Recent changes are a productive place to start.

**What the heuristic needs.** A well-designed system carries extensive production logging of
new version deployments and configuration changes at all layers of the stack
(`SRE Ch. 12, p. 177`). The book names the range. It runs from server binaries that handle
user traffic down to packages installed on individual nodes. Annotate a graph of error rates
with the start time and the end time of each deployment (`SRE Ch. 12, p. 178`).

**The anchor.** The heuristic that finds most causes also holds you on the wrong change. The
change log must state that no change is in the window as confidently as it states that one
change is. App Engine did exactly that (`SRE Ch. 12, p. 184`). A traffic spike at about 20:45
returned to baseline, so it could not explain a rise that continued for several days. The
change in performance happened on a Saturday, when no change to the application and no change
to the production environment were in flight. The most recent code pushes and configuration
pushes had completed days before, and no other application on the same infrastructure showed
the same effect.

| Evidence you hold | The claim you may write | Next action |
|---|---|---|
| The fault start time falls inside the change window | The change is one hypothesis | Test it like any other hypothesis |
| The fault started before the change started | The change is removed | Record the negative result. Search the earlier window. |
| No change of any layer sits in the window | No change is in the window | Search load, stored data, and dependency state |
| A change sits in the window and the revert removed the fault | Probable cause. Name the remaining doubt. | Hand the revert record to `reverse-branching` |
| A change sits in the window and the revert changed nothing | The change is removed | Record the negative result. Return to Examine. |

**Log every layer, not only code.** Traps `D-31`, `D-32` and `D-33` cover the three failures
here. They are the code-only change log, the last change named with no test, and the error
graph with no deploy marks.

---

## 8. Rare causes, simple causes, and several causes

Not all failures are equally probable, and you should prefer the simpler explanation, all
other things being equal (`SRE Ch. 12, p. 171`). Section 9 of
`01-the-troubleshooting-model.md` holds the full pitfall table, the horses-not-zebras rule,
and its footnoted limits at `p. 187`. Two of those limits matter to a certain diagnosis.

**A simple explanation is the first test, not the verdict.** Footnote 62 at
`SRE Ch. 12, p. 187` names Hickam's dictum. A set of common low-grade problems that together
explain every symptom can be more likely than one rare problem that causes them all.

**One incident can have several root causes.** A root cause is a defect in a software system
or a human system (`SRE Ch. 6, p. 83`). Its repair must instill confidence that the event does
not happen again in the same way. The book's own example lists three at once. Insufficient
process automation, software that crashed on bogus input, and insufficient testing of the
script that generated the configuration. Each one stands alone as a root cause, and each one
must be repaired. Traps `D-08` and `D-09` name both failures.

---

## 9. Cure means the cause and the check

**Proof is often unavailable.** Definitive proof that a given factor caused a problem, by
reproduction at will, can be difficult in a production system (`SRE Ch. 12, p. 182`). The
book gives three reasons. Systems are complex, so several factors can be jointly causal while
no single factor is the cause. Real systems are path dependent, so they must reach a specific
state before the fault appears. Reproduction in a live production system may not be an
option, because of the state complexity or because more downtime is unacceptable.

A nonproduction environment reduces these limits, at the cost of another copy of the system.
Table 9 in `../SKILL.md` maps the evidence that you hold to the claim that you may write. Use
its words. Do not write "proven cause" over a suggestive result. Trap `D-30` names that
overclaim.

**The written record.** Record four things once you find the factors that caused the problem
(`SRE Ch. 12, p. 182`). What went wrong with the system. How you located the problem. How you
fixed it. How to prevent it from happening again. The fourth item is the check. A cure
without it is a repair, not a cure.

| Part of the cure | The question it answers | Source |
|---|---|---|
| The fix for the cause | What defect did we repair? | `SRE Ch. 12, p. 182` |
| A repair for each other root cause | Which other defects produced this event? | `SRE Ch. 6, p. 83` |
| The test that fails on the old code | Would this have reached production again? | `SRE Ch. 12, p. 182` |
| The alert that fires next time | Would we have detected it, and how fast? | `SRE Ch. 6, p. 88` |
| The removal of the temporary mitigation | When does the workaround expire? | `Release It! Sec. 7.6, p. 160` |
| The telemetry that was missing | What could we not see during this incident? | `SRE Ch. 12, p. 186` |

**The cure names the alert. It does not design the alert.** `observability` owns the output
decision, the rule shape, and the threshold. Hand it the signal that would have caught this
fault. A fix for the proximate cause need not wait for the root-cause work or for the
postmortem (`SRE Ch. 12, p. 171`).

---

## 10. Where the record lands

**This skill does not own the postmortem.** `incident-response` owns the definition, the
triggers, the blameless rule, the review questions, and the repository. Read
`incident-response/references/04-postmortem-culture.md`. Its source is `SRE Ch. 15`.

The Fault Record is the input to that postmortem, not the postmortem itself. Two of its
sections carry the content of this file.

| Fault Record section | What the postmortem takes from it |
|---|---|
| Section 6, Hypotheses and tests | Every removed hypothesis, with the test that removed it |
| Section 10, What we could not see | The telemetry gap, as an action item for `observability` |

**The one rule this file keeps.** Distrust a document that names no failed test
(`SRE Ch. 12, p. 182`). A postmortem that lists only the surviving hypothesis is filtered.
Gate G5 in `../SKILL.md` requires the removed ones.

---

## 11. Trap scan for this file

| Trap | Short name | Where this file covers it |
|---|---|---|
| `D-06` | A correlation adopted as the cause | Sections 5 and 6 |
| `D-07` | Confirmation bias on a repeat alert | Section 4 |
| `D-08` | The rare cause chosen before the common one | Section 8 |
| `D-09` | Exactly one root cause recorded | Section 8 |
| `D-10` | An amplifier named as the origin | Section 5 |
| `D-11` | The system as designed, not as it runs | Section 4 |
| `D-30` | A suggestive result reported as proof | Section 9 |
| `D-31` | The change log covers code only | Section 7 |
| `D-32` | The last change named without a test | Section 7 |
| `D-33` | An error graph with no deploy marks | Section 7 |
| `D-40` | A negative result that nobody published | Sections 1 and 3 |

Report only the traps that apply. "Not applicable" is a valid result.

---

## 12. When this file says nothing

This file adds no output for a fault that already holds four things. A proven cause. A
reproduction. A test that fails on the old code. A published record of every removed
hypothesis. The record is complete. Say so and stop. It also adds nothing for a question about
a log line that no user can see. Match the output to the impact (`SRE Ch. 12, p. 173`).
Table 10 in `../SKILL.md` gives the sizes.

---

## 13. Cross-references

| Subject | File |
|---|---|
| The six steps, the loop, and the four pitfalls | `01-the-troubleshooting-model.md` |
| Mitigation before diagnosis, and evidence capture | `02-triage-first-diagnose-second.md` |
| How to produce a hypothesis from telemetry | `03-examine-and-diagnose.md` |
| How to design a test that can fail | `04-test-and-treat.md` |
| Telemetry that makes the next negative result cheap | `06-making-troubleshooting-easier.md` |
| Trap codes `D-06` to `D-11`, and `D-30` to `D-40` | `diagnostic-trap-catalog.md` |
| Gate G5, Table 9, Table 10, and the Fault Record | `../SKILL.md` |

**Hand-offs to other skills.**

- You hold a change to reverse. `reverse-branching/SKILL.md` owns the revert and its
  safety checks. This file owns the record that the revert did or did not remove the fault.
- The incident named missing telemetry. `detail-planning/SKILL.md` owns the
  observability specification that follows from it.
- The cure needs a test in the pipeline. `implement/SKILL.md` owns the code, and
  `verify/SKILL.md` owns the check that the test exists.
- The cause is a concurrency or a replication anomaly. Continue in
  `data-systems-design/references/hazard-catalog.md`.
- The incident needs a postmortem. `incident-response/references/04-postmortem-culture.md`
  owns the triggers, the blameless rule, the review, and the repository.
