# Routing to the Specialist Skills

**Sources:** This repository and the seventeen sibling skills in it, read on 2026-09-07.
Method framing from *The Minto Pyramid Principle* — Minto, Ch. 8 "Defining the Problem". The
Next Steps rule from Minto, Ch. 10 "Putting It Down on Paper". The news rule from Minto, Ch. 5
"Deduction and Induction: The Difference". One diagnosis from *The Art of Explanation* —
LeFever, Ch. 4 "Planning Your Explanations". The skill list is **Modern**. Neither book knows
a skill suite.

The consultant diagnoses. It refers. It does not perform the specialist work, and it does not
restate a specialist catalog. This file states which skill takes each question, what that
skill returns, and what the consultant does with the result.

---

## 1. The hard rule

**The consultant never writes code.** It writes a record. It names the file that must change
and the change that file needs. It routes the build work to `implement`.

A file edited in the user's project is defect X-01 in `advice-defect-catalog.md`. A
recommendation delivered as a patch is X-02. Both are 🔴. The reader asked for advice and
received a change nobody reviewed.

**The consultant refers. It does not duplicate.** Each specialist skill holds its own method,
its own decision tables, and its own defect catalog. The consultant names the skill and the
one question that skill inherits.

**A record that names no next owner is defect X-03.** The work then converts into no action.
The reader must derive the next step alone, which is the step the consultant was asked to
remove.

---

## 2. What Minto Ch. 8 supplies, and what it does not

**Minto Ch. 8 does not discuss routing.** It defines a problem. The chapter supplies one
sentence that decides where a question belongs, and this file translates that sentence into
software. Sections 2 and 3 are translation, not report. Section 4 onward is this repository.

Minto names four elements. A problem is not defined until all four exist. Ch. 8, "Lay Out the
Problem," p. 127.

1. The Starting Point / Opening Scene. The structure or the process that works today.
2. The Disturbing Event. The thing that threatened it.
3. R1. The undesired result.
4. R2. The desired result, with a number or a specific end state.

The routing sentence is this one. "The Solution generally comes from changing what is going on
in the structure or process identified as the original Starting Point/Opening Scene."
Ch. 8, "Laying out the Elements," p. 123.

**Translation.** The Opening Scene names the owner. Draw the structure or the process that the
change touches. The skill that owns that structure owns the work. A replication topology
routes to `data-systems-design`. A signal handler routes to `systems-programming`. A token
routes to `security-engineering`.

**The R2 row also names an owner.** Minto states R2 in end-product terms with a number or a
specific end state, Ch. 8, "R2 (Desired Result)," p. 130. **Modern translation.** Some numbers
in software are not the consultant's to set. A reliability number belongs to `slo-engineering`.
A saturation number belongs to `capacity-engineering`. The consultant states the R2 row and
routes the threshold.

### Minto's worked example — the industrial real estate seller

Ch. 8, "Laying out the Elements," pp. 122–124, Exhibit 32. A company sold industrial real
estate for thirty years with one method. Salesmen list the prospects, write a script, and
deliver the message. Those three boxes are the Opening Scene, and they yielded ten percent
growth a year. The Disturbing Event was a projection that showed quarterly sales down ten
percent instead of up ten percent. Because the solution changes the Opening Scene, the
candidate causes are exactly the three boxes. The list is stale, or the script is weak, or the
delivery fails. **Engineering form.** Draw the pipeline, the request path, or the deploy path
before you name a candidate. The drawing supplies the decomposition, and each box carries an
owner in section 5.

### Minto's worked example — the supermarket test-marketing fee

Ch. 8, "Converting to an Introduction," pp. 125–126, Exhibit 33. A packaged foods manufacturer
wanted a week of shelf space to test new products. The supermarkets refused, so the
manufacturer paid a fee. The supermarkets formed chains and raised the fee to twenty thousand
dollars a week. A committee refused to pay. The supermarkets then refused all week-long
testing. Minto calls this a triple-layer problem. Every earlier attempt collapses into the
Situation, and only the newest undesired result becomes the Complication. **Engineering form.**
A repository that already tried two fixes routes on its newest R1, not on its oldest. Read the
prior attempts as history. Route the live failure, and give the specialist that history.

### Minto's worked example — the household goods distributor

Ch. 8, "Real-Life Example," pp. 137–138, Exhibits 35 and 36. Three distribution centers plus
rented space could serve 438 stores, although the design assumed 490. Volume grew four to five
percent a year and the company planned 198 new stores. R1 was a capacity limit inside two
years. R2 was sufficient capacity, with four stated criteria. Lowest capital outlay, lowest
operating cost, the same processing speeds, and the same full-line strategy. The company had
already named its alternatives. The recommendation was an action, not a winner. Add capacity
step by step, and delay the fourth warehouse. **Engineering form.** This is the shape of a
consultant question with a routed threshold. The consultant compares the options against R2.
`capacity-engineering` measures which resource reaches its limit first.

---

## 3. Where the reader stands decides the route

**Minto's own warning is the routing gate.** "A big error some writers make is in not
specifying to themselves whether some action has already been taken by the reader."
Ch. 8, "Look for the Question," p. 131.

Ask one question before you route. Does the thing exist yet, and did the reader already act on
it? The seven problem situations of Exhibit 34, p. 132, answer it. The route column is
**Modern**. Minto names the situation. This repository names the owner.

| The reader's position (Minto, Exhibit 34) | The question it generates | Route |
|---|---|---|
| 1. They do not know how to get from R1 to R2 | How do we get from R1 to R2? | `planner`, or `system-design` when the system does not exist |
| 2. They think they know, and are not certain | Is it the right solution? | **Stay.** Duty 1, the comparison |
| 3. They know the solution, not the implementation | How do we implement it? | `detail-planning`, then `implement` |
| 4. They implemented, and it did not work | What should we do? | `production-troubleshooting`. `incident-response` when it fails now |
| 5. They hold several options and cannot pick one | Which alternative? | **Stay.** Duty 1, the comparison |
| 6. They know R1 and cannot state R2 | What are our objectives? | **Stay.** The R2 table is the deliverable |
| 7. They know R2 and doubt they are at R1 | Do we have a problem? | An audit against R2, then the owning skill measures it |

**Situation 5 is the consultant's home case.** The reader holds candidates and cannot choose.
Read `09-the-tradeoff-evaluation.md`.

**Situation 6 stops the comparison.** With no threshold and no end state, no option can be
shown to meet the requirement. Read `08-defining-and-structuring-the-problem.md`.

**Do not manufacture a route.** Minto refuses to invent a Disturbing Event when the
information is missing. "In that case do not trouble yourself with trying to manufacture a
Disturbing Event." Ch. 8, "The Disturbing Event," p. 129. **Translation.** When no specialist
owns the question, say so and answer it here. An invented routing line costs the reader a read
and returns nothing.

---

## 4. The three results of a routing decision

**A routing decision has three results, not two.** Name the result before you write the record.
Each result has a signal that triggers it.

| Result | The signal | What the consultant does |
|---|---|---|
| **Stay** | The reader must choose between named options, or must understand a concept | Runs Duty 1 or Duty 2 to the end. The routing line appears only in Next Steps |
| **Route and continue** | One criterion needs evidence that only a specialist method produces | Writes that criterion as a yes/no issue. Marks the cell **undetermined**. Routes the one question |
| **Stop and route now** | A service fails now, or a production fault has no known cause | Emits the routing line alone. Writes no comparison and no explanation |

**Stop and route now costs the reader nothing.** A reader inside an incident has no time to
read a record. Two rows of section 5 carry this result. They are `incident-response` and
`production-troubleshooting`.

---

## 5. The routing table

Order used: **time**. The rows follow the life of a change. Rows 1 to 9 run before the plan.
Rows 10 to 15 run the pipeline. Rows 16 and 17 run after the release.

| Skill | The question that routes the work there | What that skill returns | What the consultant does with the result |
|---|---|---|---|
| `system-design` | What must we build, and how large must it be? | A seven-step design, a load estimate, a component choice, a scaling step | Keeps the concept explanation. Cites the design as the Opening Scene |
| `data-systems-design` | Does the design stay correct across machines? Storage, schema, replication, sharding, isolation, queues, event streams | A Design Record, decision tables, and an `H-` hazard scan | Keeps the option comparison. Copies the hazard names into the record as constraints |
| `systems-programming` | Does the code stay correct against the kernel? Files, processes, signals, threads, linking, loading | Kernel boundary rules, recipes, and an `S-` failure scan | Keeps the concept explanation. Nothing else |
| `security-engineering` | Can an attacker break it? Secrets, identity, tokens, crypto, untrusted input | A threat model, decision tables, and a `V-` vulnerability scan | Keeps nothing. A security criterion is scored there and cited here |
| `slo-engineering` | What number does the word "reliable" mean here? | An SLI, an SLO, an error budget, a release policy, an `L-` scan | Writes the returned number into one R2 row |
| `capacity-engineering` | Which resource reaches its limit first? | The bottleneck, a load-test plan, a headroom number, a `C-` scan | Writes the measurement into the performance cell, scoped to the workload |
| `observability` | What must the system expose, and what must page a person? | Metrics, alerts, logs, dashboards, an `M-` gap scan | Keeps nothing. Records the routing line only |
| `reverse-branching` | How does a person or an agent undo this safely? | A reversibility contract, a Reversal Record, an `R-` hazard scan | Writes the exit cost into one R2 row |
| `self-healing-apis` | Is the fault in us, in the network, or in the vendor? Timeouts, retries, breakers, webhooks | A localization result and an `I-` integration-fault scan | Keeps the vendor comparison. Cites the localization result |
| `planner` | Should we build it, and in what order? | A phased `plan.md` with non-functional requirements | Gives the recommendation as the first input to the plan |
| `detail-planning` | What exactly does this one phase do? | `executor.md`, a file-referenced specification | Keeps nothing |
| `implement` | Write the code | Production code that holds the robustness invariants | Keeps nothing. This is the read-only boundary |
| `verify` | Does the code match the specification? | A discrepancy list with concrete fixes | Keeps nothing |
| `code-review` | Is this difference safe to merge? | Severity-ranked findings and a merge verdict | Keeps nothing |
| `engineer-workflow` | Run the pipeline from plan to review | The pipeline state in `plan.md` and `executor.md` | Gives the recommendation as the first input |
| `incident-response` | A service breaks now. Who commands? | Named roles, a written incident state, a postmortem, an `N-` scan | Keeps nothing. Stop and route now |
| `production-troubleshooting` | What caused this production fault? | A diagnosed cause with evidence, and a `D-` trap scan | Keeps nothing. Stop and route now |

**The catalog prefixes above are real files.** They were read in this repository on 2026-09-07.
`H-` holds 48 entries, `S-` 55, `V-` 64, `L-` 47, `C-` 47, `M-` 44, `R-` 50, `I-` 48, `N-` 52,
and `D-` 40. Seven skills hold no catalog. They are `system-design`, `planner`,
`detail-planning`, `implement`, `verify`, `code-review`, and `engineer-workflow`. Date every
count you cite, because a count changes when the suite changes.

**One case routes nowhere.** The reader asks for a comparison, and the evidence shows the
incumbent meets R2 and is not understood. That is an explanation problem, not a selection
problem. Stay, and run Duty 2. LeFever Ch. 4, "Identifying Explanation Problems," pp. 34–36.
It is defect X-08 when the consultant misses it.

---

## 6. Two boundaries the consultant defends

**A criterion that depends on a specialist's subject is scored there, not here.** The
consultant states the criterion as a yes/no issue, names the skill, and leaves the cell
**undetermined** until the specialist answers.

The reason is evidence, not politeness. A specialist skill holds the method that settles the
question. A consultant answer would be a guess with a citation missing. Read
`10-research-method-and-sources.md` for the rule that silence reads undetermined, never "no".

**A concept explanation stays here, whatever its subject.** Explaining is this skill's own
duty. The consultant explains replication, or a signal, or a token, at the reader's level. The
follow-on work then routes.

**The test between the two.** Ask whether the answer changes a cell in the score matrix. A
cell change is specialist work. A reader change is consultant work.

---

## 7. How to load a skill

**Signal.** The routing decision is made. The consultant needs one fact from that skill, such
as a hazard name, a threshold rule, or the exact scope of the skill.

**Read the skill by its own index. Do not read its whole directory.**

1. Read the target skill's `SKILL.md` in full. It states the skill's scope and its rules.
2. Find the reference index near the end of that file. Each row names a file, its content, and
   the signal to read it.
3. Read only the reference files whose signal matches your question.
4. Read the skill's defect catalog only when the question asks for a scan of that kind.
5. Record every file you read in the Sources table of your record, with the date.

**The consultant loads a skill to read one fact.** It does not run the specialist method. The
method belongs to the specialist, and the reader runs it later. A consultant that runs a
specialist method produces an unreviewed second opinion.

**A skill you read whole produces news.** Minto uses that word for facts that are true and
unrelated. Ch. 5, "How It Differs," p. 72. Extra reading costs context and adds cells that no
R2 row needs.

**Never copy a specialist catalog into the record.** Cite the code and the file. `H-14` and
`data-systems-design/references/hazard-catalog.md` are enough for a reader to find the entry.

**Modern.** Both books predate an agent that reads its own instructions on demand. The load
rule is this repository's.

---

## 8. When the consultant answers alone

**The signal is the verb.** A question about choosing or about understanding stays here. A
question about building, operating, or repairing leaves.

| The reader asks | The consultant | Because |
|---|---|---|
| "What is X?" or "How does X work?" | Answers alone. Duty 2 | Explaining is this skill's duty |
| "X or Y?", and no criterion needs a specialist | Answers alone. Duty 1 | The evidence is documentation, a licence, and the repository |
| "X or Y?", and one criterion needs a specialist | Answers, and routes that one criterion | Section 6, boundary 1 |
| "Should we migrate?" | Answers alone, then routes the plan to `planner` | The decision is a comparison. The sequencing is a plan |
| "How do I build X?" | Routes to `planner` | The reader is at situation 1 or 3 |
| "Write X" or "Fix X" | Routes to `implement` | Section 1, the hard rule |
| "Review this pull request" | Routes to `code-review` | The consultant checks arguments, not diffs |
| "The service is down" | Routes to `incident-response`. Stops | The consultant is not an operator |
| "Why is production slow?" | Routes to `production-troubleshooting`. Stops | A fault needs a diagnosis, not a comparison |
| "Is X good?" | Refuses the question. Rewrites it against R2 | An unscoped question has no admissible evidence |

**The consultant may answer and route in the same record.** The two are not exclusive. State
the answer in the body. State the route in Next Steps.

---

## 9. Where the routing line goes

**Put the routing line in Next Steps. Put nothing arguable there.** Minto states the admission
rule for that section. "The only rule is that what you put in this section must be things that
the reader will not question." Ch. 10, "Stating Next Steps," p. 187.

The routing line has two parts and no more.

1. The skill name, in backticks.
2. The single question that skill inherits, in one interrogative sentence.

```
Route "Is the 30-second stall in our client, in the network, or in the vendor?"
  to self-healing-apis.
Route "Which resource saturates first at 10,000 writes per second?"
  to capacity-engineering.
```

**A routing line that carries a verdict belongs in the body.** "Route the migration to
`planner`" assumes the migration. That assumption is arguable, so it belongs in the Key Line
where the reader can challenge it.

**One routing line per open question.** Two questions in one line hide which one the specialist
must answer first. Read `07-logical-order-and-mece.md` for the ordering rule.

**Give the specialist the Situation, not only the question.** The supermarket example in
section 2 is the reason. A specialist that inherits the newest R1 alone can repeat a fix the
team already tried. Name the two or three statements the reader accepts without proof.

---

## 10. Routing faults

Each fault below has a signature you can search for. The fix is required, not optional.

| Symptom | Diagnosis | Fix |
|---|---|---|
| The record ends at the verdict | No owner named. X-03 | Add one action and one routing line |
| The record repeats a hazard catalog | Duplication, not referral | Cite the code and the file. Delete the copy |
| Every question routes somewhere | The consultant abdicated | Answer the choosing questions and the understanding questions here |
| Nothing routes | The consultant scored a specialist cell | Mark the cell undetermined. Name the skill |
| The routing line names two skills | The question was not collapsed | Split it into two questions, or find the one that runs first |
| A live outage is compared, not routed | The reader is at situation 4 | Stop. Route to `incident-response` now |
| The routing line asserts the plan | An arguable item in Next Steps | Move the assertion into the body |
| The specialist asks for context the record holds | The routing line carried no Situation | Add the agreement statements to the line |
| A file changed in the user's project | The read-only charter broke. X-01 | Stop. Tell the user which file changed. State the change as a recommendation, and route the work |

---

## 11. When not to route

**"Not applicable" is a valid result.** It is better than an invented routing line.

Do not route in these four cases.

1. The question is a definition and no decision depends on it. Answer it in two sentences.
2. The reader needs a concept, not a change. Section 6, boundary 2.
3. No specialist owns the subject. Say that, and answer with the evidence you hold.
4. The specialist skill would return the answer the record already carries. Cite it once.

**Route at once, and stop, in two cases.** A live outage routes to `incident-response`. A
production fault with an unknown cause routes to `production-troubleshooting`. In both cases
the consultant produces no comparison and no explanation.

---

## Related references

- `08-defining-and-structuring-the-problem.md` — the four elements, R1 and R2, the seven
  problem situations in full.
- `09-the-tradeoff-evaluation.md` — the comparison method that situation 5 triggers.
- `07-logical-order-and-mece.md` — MECE, the plural noun, and the five-point cap.
- `10-research-method-and-sources.md` — source precedence, and the undetermined rule.
- `01-the-explanation-scale.md` — the reader's letter, which fixes how much a routing line
  must explain.
- `advice-defect-catalog.md` — X-01, X-02, X-03, and X-08 in full.
- `data-systems-design/references/hazard-catalog.md` — the `H-` codes this file cites.
- `reverse-branching/references/rollback-hazard-catalog.md` — the `R-` codes this file cites.
