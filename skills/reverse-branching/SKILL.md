---
name: reverse-branching
description: Reversibility engine for changes to code, configuration, schema, and data. It answers one question. How does a person or an agent undo this change safely? Use it for a revert, a rollback, a checkpoint, a git reset, or a force push. Use it for a cherry pick, a release branch, a canary, a staged rollout, or a feature flag. Use it for a kill switch, a migration, a backup, a restore, or point-in-time recovery. Use it for a bad deploy, a change-induced outage, a bisect, or the question "what changed last". It gives a reversibility contract, decision tables, a rollback hazard catalog, and agent checkpoint commands.
---

# REVERSE BRANCHING

**ROLE:** You are the reversibility engineer for a change. The change comes from a human
operator or from an AI coding agent.

**CORE FUNCTION:** You receive a change. You produce the exact path that undoes it, the
actor who may run that path, and the measured time that the path takes.

A change is easy to make. A change is hard to undo. The team learns the difference during
an outage, at speed, with no rehearsal. This skill moves that work before the merge.

**This skill exists to make the reverse path explicit before the change ships.**

## How this skill relates to the other skills

Four skills answer different questions about a production change before it ships. Read this
table before you choose one. Section 11 adds the runtime siblings and names the owner of
each overlap.

| Skill | Question it answers | Level |
|---|---|---|
| `system-design` | What must we build? | Architecture |
| `data-systems-design` | Does the design stay correct across machines? | Distributed data |
| `reverse-branching` (this skill) | How do we undo this change? | Change and release |
| `security-engineering` | Can an attacker break it? | Threat |

This skill is a **knowledge and gate skill**. It runs no workflow. `planner`,
`detail-planning`, `implement`, `verify`, and `code-review` consult it. Section 11 names
the owner of each overlap.

---

# 1. WHEN TO USE THIS SKILL

Use this skill for these tasks:

- You plan a change that reaches production.
- You write a migration, a backfill, or a delete path.
- You change a configuration value, a flag, a route, or an allowlist.
- You design a rollout, a canary, a staged release, or a feature flag.
- You build a kill switch or a stop control.
- You revert a commit, a deploy, or a deployed artifact.
- You run `git reset`, `git rebase`, or a force push.
- You cherry pick a fix onto a release branch.
- You restore data from a backup or from a point in time.
- You answer the question "what changed last".
- You search for the change that caused a fault.
- You write the rollback section of a plan or of an `executor.md` phase.
- An AI coding agent edits files that no person reviewed yet.
- A change caused an outage.

Do not use this skill for these tasks:

- A change to one file with no persisted state and no deploy.
- A comment change or a rename inside one function.
- A question about which service owns which data. Use `system-design`.
- A question about isolation levels or replication. Use `data-systems-design`.

**A configuration change activates this skill as much as a code change.** SRE defines a
push as any change to a service's running software or its configuration. SRE Ch. 6, p. 83.
A data change activates it too. Code reverts. Data does not.

---

# 2. THE REVERSIBILITY CONTRACT

Every change answers four questions before it merges. The answers form the Reversal Record
in Section 8.

| # | Question | A valid answer names |
|---|---|---|
| 1 | What artifact reverts? | The artifact, its version scheme, and the exact reverse action |
| 2 | What data reverts? | The tables, the restore layer, the RPO, and the RTO (**Modern** terms) |
| 3 | What side effect cannot revert? | The effect, and the compensating action, or "none exists" |
| 4 | Who may press the button? | The actor, and the approver when one is required |

A change has one of six reversibility classes. Table 1 in Section 5 gives them.

**RPO and RTO are Modern.** The industry uses them for "how much recent data may we lose"
and "how fast must users return to service". Neither book uses the two terms. SRE Ch. 26,
p. 427 asks both questions in plain words.

**A change inherits the class of its least reversible part.** A pull request that adds a
function and drops a column is a destructive schema change. The function does not soften
the drop.

Two rules follow from the contract.

1. Name the step that makes the change irreversible. Release It! calls it the Cleanup
   phase. Release It! Sec. 18.4, p. 334. This skill calls it the point of no return.
2. State the condition that permits that step. A bake period, a validator result, or a
   named approver.

---

# 3. THE CORE GATES

Each gate is one sentence. Each gate carries a citation. `code-review` and `verify` test a
change against these gates.

1. Every change declares its reverse path before it merges. SRE Ch. 27, p. 461.
2. Revert first. Diagnose second. SRE App. B, p. 572.
3. A rollback procedure that no test exercised does not exist. SRE Ch. 13, p. 191.
4. A backup that no restore test exercised does not exist. SRE Ch. 26, p. 434–435.
5. Replication is not recoverability. SRE Ch. 26, p. 421.
6. Canary every change, whatever its perceived risk. SRE Ch. 13, p. 194.
7. A nonemergency rollout proceeds in stages. SRE App. B, p. 572.
8. A monitoring system that you can demonstrate to be reliable supervises each stage. The book prefers it to the engineer who runs the stage. SRE App. B, p. 572.
9. Configuration is a change. Give it version control, review, and the same rollout ladder as code. SRE Ch. 8, p. 127. SRE Ch. 27, p. 459.
10. Never combine expand and contract in one change. Release It! Sec. 18.4, p. 332–334.
11. Delete dead code. Do not keep it behind a disabled flag. SRE Ch. 9, p. 132–133.
12. Treat an empty target set as an error, never as every target. SRE Ch. 13, p. 196.
13. Rate limit every automated reversal. SRE Ch. 7, p. 118. Give it a stop control. SRE Ch. 13, p. 195.
14. Make every reversal step idempotent. SRE Ch. 7, p. 110 and p. 118.
15. Automation reports a data defect. It does not repair the defect by deleting data. SRE Ch. 17, p. 242.
16. One actor writes to production during an incident. SRE Ch. 14, p. 202–203.
17. A revert is a reportable event. It triggers a postmortem. SRE Ch. 15, p. 209.
18. Log every deploy and every configuration change, at every layer of the stack. SRE Ch. 12, p. 177.
19. The revert path must not run through the system that is down. SRE Ch. 13, p. 193.
20. On implausible input, keep the previous state, raise an alert, and wait for a person. SRE App. B, p. 571.
21. Set the backup retention window from the release cadence and the detection latency. SRE Ch. 26, p. 428.
22. Rebuild an old release with the toolchain version that the source revision pins. SRE Ch. 8, p. 122.

---

# 4. THE CHANGE-INDUCED EMERGENCY

A change caused the fault. This is the order of moves. It is the shortest section and the
most important one.

1. Declare the incident. Name one write holder. Every other person proposes to that holder. SRE Ch. 14, p. 202–203.
2. Stop the rollout in flight.
3. Disable the automation first when automation causes the damage. SRE Ch. 13, p. 195.
4. Revert the monitoring change first when the change removed monitoring. SRE Ch. 13, p. 196.
5. Freeze the system when the fault can destroy data. SRE Ch. 12, p. 173.
6. Revert the change. Do not diagnose first. SRE App. B, p. 572.
7. Report at once when the revert did not fix the symptom. Do not repeat it. SRE Ch. 14, p. 204–205.
8. Restore traffic in stages. Check the SLO between each step. SRE App. D, p. 583.
9. Record the revert. Write the postmortem. SRE Ch. 15, p. 209.

**A mitigation is a change too.** One team applied a load balance change during an
incident. The change made the fault worse. The team reverted it four minutes later.
SRE App. D, p. 582–583.

**Reach for the out-of-band path when the console is down.** Google keeps command-line
tools and other access methods for exactly this case. Engineers must test them on a
routine schedule. SRE Ch. 13, p. 193.

---

# 5. DECISION TABLES

Use these tables to decide quickly. Read the linked reference before you commit to
anything that is expensive or hard to reverse.

## Table 1 — What class of change is this?

| Class | Example | Reverse action | Time to reverse | Point of no return |
|---|---|---|---|---|
| Pure code | A function body. No persisted state. | Revert the commit. Deploy the prior artifact. | Minutes | None |
| Configuration | A flag value, a timeout, a route, an allowlist | Deploy the previous configuration version | Seconds to minutes | None, while the configuration is versioned |
| Schema, additive | A nullable column, a new table, a new endpoint | Stop writing the field. Remove it later. | Minutes | The contract step |
| Schema, destructive | A dropped column, a rename, a new NOT NULL | Restore the data from a backup | Hours | The moment the statement commits |
| Data | A backfill, a bulk update, a delete | Undo the soft delete, or restore to a point in time | Minutes to days | The end of the retention window |
| External side effect | A sent email, a payment call, a partner file, a wiped disk | A compensating action, when one exists | Never, for some | The moment the effect leaves the system |

Sources. Configuration as a separate class, SRE Ch. 8, p. 127–128. Schema phases,
Release It! Sec. 18.4, p. 332–334. Data, SRE Ch. 26, p. 423–429. An effect with no undo,
SRE Ch. 13, p. 194–195. The Diskerase incident destroyed the disks. Recovery needed a
manual reinstall. Most capacity returned within three days. Stragglers took one to two
months.

## Table 2 — Which reverse action do I use?

| The change is | Reverse action | Rule and source |
|---|---|---|
| Behind a feature flag | Disable the flag | The framework reverts each change alone and at once. SRE Ch. 27, p. 463 |
| Configuration only | Deploy the previous configuration package | No new binary build is needed. SRE Ch. 8, p. 128 |
| A deployed artifact | Move the release label to the previous artifact | A label move is the cheapest reverse action. SRE Ch. 8, p. 124–125 |
| One commit on a shared branch | Add a revert commit | It rewrites no history. **Modern** |
| A rollout in progress | Stop the rollout. Drain the new pool. | SRE Ch. 17, p. 245. Release It! Sec. 18.4, p. 334 |
| A destructive migration | Restore the data. Do not revert the code first. | The old code reads the new data as incomplete. Release It! Sec. 18.4, p. 333 |
| An external side effect | Run the compensating action. Report when none exists. | SRE Ch. 13, p. 194–195 |
| Automation that still runs | Disable the automation first | SRE Ch. 13, p. 195 |
| Monitoring or alerting | Revert this first, before you assess the damage | SRE Ch. 13, p. 196 |

## Table 3 — Revert, or roll forward?

| Condition | Choose | Because |
|---|---|---|
| A change caused the fault. The prior state is known good. | Revert | Revert first and diagnose afterward. SRE App. B, p. 572 |
| The change sits in a canary stage | Revert | A rollout that triggers a variance needs a fast return to the prior configuration. SRE Ch. 17, p. 229 |
| Nobody knows which change caused the fault | Revert the whole release, then search | Revert first and diagnose afterward. SRE App. B, p. 572. A batch of 100 changes makes attribution expensive. SRE Ch. 9, p. 135 |
| The revert crosses a destructive migration | Roll forward | The old code cannot read the new data. Release It! Sec. 18.4, p. 333 |
| The prior artifact cannot be rebuilt | Roll forward | A non-hermetic build cannot reproduce it. SRE Ch. 8, p. 122 |
| The fault is bad data, not bad code | Roll forward with a corrective push, or restore | A new index resolved the query of death. SRE App. D, p. 579 and p. 581 |
| The revert procedure was never tested | Roll forward, and record the defect | An untested rollback lengthened the outage. SRE Ch. 13, p. 191 |
| The mitigation you just applied made it worse | Revert the mitigation, at once | SRE App. D, p. 582–583 |

## Table 4 — Checkpoint and revert commands (**Modern**)

Every command in this table postdates both books. The safety rule beside each one comes
from the books. Section 6 gives the four rules that govern the whole table.

| Goal | Command | Safety rule |
|---|---|---|
| Mark a safe point before a risky edit | `git commit -m "checkpoint: <name>"` | A stash is not a checkpoint. A stash has no name and is easy to lose. |
| Name a checkpoint for later | `git tag ckpt/<name>` | A tag survives a rebase of the branch. |
| List the checkpoints | `git tag -l 'ckpt/*'` and `git log --oneline` | List the configuration commits too. A push is any change to running software or its configuration. SRE Ch. 6, p. 83 |
| Undo one published commit | `git revert <sha>` | It adds a commit. Every other clone stays valid. |
| Undo one file to a checkpoint | `git restore --source=<ref> -- <path>` | It touches one path. It keeps the other work. |
| Undo the last agent turn, unpublished | `git reset --hard <checkpoint>` | Only on a branch that one actor holds. Never on a shared branch. |
| Recover a commit that you lost | `git reflog`, then `git reset --hard <sha>` | The reflog is local. It expires. It is the last resort. |
| Isolate an agent from the main tree | `git worktree add ../agent-<task> -b agent/<task>` | The agent writes in its own directory. The main tree stays clean. |
| Find the bad commit | `git bisect start`, `bad`, `good`, `run <script>` | The test must give the same answer every time. SRE Ch. 17, p. 236 |
| Revert a deployed artifact | Move the release label to the previous version | SRE Ch. 8, p. 124–125 |

## Table 5 — How do I locate the bad change?

| Signal shape | Method | Caution |
|---|---|---|
| A step change at a known time | Read the deploy log for that window | Annotate the error graph with the deploy start and end. SRE Ch. 12, p. 177–178 |
| A step change, many candidates | Binary search over the commit list | Bisection needs one working end and one failing end. SRE Ch. 12, p. 176. The test must give one answer. SRE Ch. 17, p. 236 |
| A probabilistic fault | Search on a rate over a fixed sample | A flaky signal gives a wrong answer. SRE Ch. 17, p. 236–237 |
| Visible only under production load | Canary each candidate in turn | A canary is structured user acceptance, not a test. SRE Ch. 17, p. 229 |
| Slow corruption found weeks later | Search the change log back to the retention limit | Low-grade deletion bugs appear months later. SRE Ch. 26, p. 421 |
| Two metrics moved together | Design a test with mutually exclusive outcomes | Correlation is not causation. SRE Ch. 12, p. 171, p. 179, and p. 184–185 |
| No change in the window | Stop. The cause is not a change. | SRE Ch. 12, p. 184 |

## Table 6 — Rollout stage, exposure, and reverse cost

| Stage | Exposure | Reverse action | Reverse time |
|---|---|---|---|
| Not merged | 0 | Close the change | Seconds |
| Merged, not deployed | 0 | Add a revert commit | Minutes |
| Canary | 0.1% to 1% | Stop and drain the canary | Minutes |
| One region or one datacenter | 1% to 10% | Move the label back. Drain the pool. | Minutes |
| Every region | 100% | Full artifact rollback, then a staged restore of traffic | Tens of minutes |
| Contract step applied | 100%, and the data changed | Restore from a backup | Hours |

Ladder numbers. Start at 0.1% of user traffic. Scale by one order of magnitude each 24
hours, and vary the geography. SRE Ch. 17, fn. 89, p. 246. A feature flag group grows to
between 1% and 10%. SRE Ch. 27, p. 463. Restore traffic after a revert in the ladder 1%,
10%, 30%, 50%, 100%, with an SLO check between each step. SRE App. D, p. 583.

## Table 7 — The data reversal ladder

| Layer | Reaches back | Restore time | Guards against |
|---|---|---|---|
| Trash, visible to the user | 30 days in Gmail | Seconds | User error |
| Soft deletion | 15, 30, 45, or 60 days | Minutes | Developer error |
| Lazy deletion, in the storage layer | Up to a few weeks | Minutes | Internal developer error |
| Backup tier 1, next to the store | Hours to single-digit days | Minutes | A bad release |
| Backup tier 2, site-local filesystem | Single to low double-digit days | Hours | A bad job, a wide fault |
| Backup tier 3, offsite media | 30 to 90 days | Hours to days | A site fault, a defect in a low layer |
| Replication | 0 | 0 | Nothing. It copies the error to every replica. |

Sources. Trash and soft-deletion delays, SRE Ch. 26, p. 425. Layer ownership, p. 426.
Tier retention and restore times, p. 428–429. Replication is not recoverability, p. 421.
The 30 to 90 day figure is the outer line that Google drew for many services, p. 428. The
book gives no separate retention number for the offsite tier.

## Table 8 — Migration phase and revert safety

| Phase | What it does | Is a code revert safe? | Rule |
|---|---|---|---|
| Expand | Adds a nullable column, a table, an endpoint, a versioned asset URL | Yes | Add every future NOT NULL column as nullable. Add no referential integrity yet. Release It! Sec. 18.4, p. 333 |
| Migrate | Backfills, writes both shapes, moves reads | Yes, while both shapes stay current | Bridging triggers fill the new shape from old writes, and the old shape from new writes. Release It! Sec. 18.4, p. 333 |
| Contract | Drops a column, adds NOT NULL, adds a foreign key, deletes old assets | No | This is the point of no return. Run it only after the release bakes. Release It! Sec. 18.4, p. 334 |

`data-systems-design` owns the compatibility rule for this pattern. This skill owns the
reversibility consequence. Do not restate the sibling.

## Table 9 — Who may press the button?

| Actor | May act without a person | Must ask a person first |
|---|---|---|
| Monitoring or the release updater | Stop a stage. Return to the last known good state. | Any action that deletes data |
| AI coding agent | Commit a checkpoint. Revert its own unpublished branch. Open a revert request. | A push to a shared branch. A destructive migration. A data restore. |
| Human operator, on call | Revert an artifact. Disable a flag. Drain traffic. | A restore that overwrites live data |
| Bulk automation | Nothing above its rate limit | Every action past the limit |

Sources. The updater returns to the last known good state on its own, SRE Ch. 17, p. 244.
A tool reports a data defect and does not repair it by deleting data, SRE Ch. 17, p. 242.
Rate limiting and sanity checks were the fix after Diskerase, SRE Ch. 7, p. 118. One team
modifies the system during an incident, SRE Ch. 14, p. 202–203.

---

# 6. THE CHECKPOINT AND REVERT COMMAND SURFACE

Table 4 holds the commands for an AI coding agent and for a human operator. Four rules
govern every row. The commands are **Modern**. The rules come from the books.

### Rule 1. A checkpoint is a commit with a name.

The agent commits before the risky edit. The commit message names the state. A stash has
no name and no message. A second stash hides the first one. A clean operation drops it.
Catalog entry R-13 gives the full case.

SRE states the general form of this rule for a diagnostic session. Change the system in a
systematic and documented way. Then you can return to the state before the test.
SRE Ch. 12, p. 180.

### Rule 2. A history rewrite is safe only on a branch that one actor holds.

`git reset --hard` and a force push remove work with no record. On a shared branch, other
clones diverge. Add a revert commit instead. The revert commit keeps the audit trail.
Catalog entry R-11 gives the full case.

Google gates every release operation and logs every request. No person installs or
modifies a server with no audit trail. SRE Ch. 7, p. 113. SRE Ch. 8, p. 122–123.

### Rule 3. The agent reverts its own branch. A person approves the rest.

Table 9 gives the split. The agent commits a checkpoint, reverts its own unpublished
branch, and opens a revert request. A person approves a push to a shared branch, a
destructive migration, and a data restore. Give the agent its own worktree. Then the
agent cannot touch the main tree by accident.

### Rule 4. A command that reverts code reverts only code.

`git revert` does not restore a dropped column. It does not recall a sent email. It does
not return a tuned garbage collection flag to its previous value. Release It! states the
last case. Perfect settings for one release can be wrong for the next one.
Release It! Sec. 10.4, p. 215 and p. 217.

Answer question 2 and question 3 of the contract before you trust a command. Catalog
entries R-03, R-25, and R-26 give the three failure shapes.

---

# 7. THE REVERSAL GATE (MANDATORY)

Run `references/rollback-hazard-catalog.md` against the change before you accept it. The
catalog holds 50 named hazards with the prefix `R-`, in eight groups.

Each entry gives a **signature**, a **consequence**, and a **fix**. The signature is the
text that you search for in a difference, a pipeline file, or a plan.

Severity. 🔴 causes data loss, corruption, or an unrecoverable state. It blocks the merge.
🟡 causes an outage or a wrong result under load. 🔵 is a risk to operation or maintenance.

Report format:

```md
### Hazard Scan
| Hazard | Present | Evidence | Required fix |
|---|---|---|---|
| R-26 Code revert across a destructive migration | Yes | `migrations/031_drop_legacy.sql` runs in the same release | Split the drop into a later contract release |
| R-11 History rewrite on a shared branch | No | The workflow uses a revert commit | — |
```

List only the hazards that apply. **"Not applicable" is a valid and preferred result.** An
honest "not applicable. This change touches one file, no state, and no deploy." beats an
invented finding.

Use the catalog index by symptom when an incident is in progress. It maps a symptom, such
as "the revert did not fix the symptom", to the entries that explain it.

---

# 8. REVERSAL RECORD OUTPUT

Produce this record when the skill runs on a change. It is short on purpose. Every line is
a commitment that `verify` and `code-review` can check later.

```md
## Reversal Record: [Change]

### Classification
- Reversibility class: [pure code / configuration / schema additive / schema destructive / data / external side effect]
- Reason for the class: [the least reversible part of the change]
- Blast radius at full exposure: [users, rows, regions, money]

### The Four Questions
| Question | Answer |
|---|---|
| What artifact reverts? | [artifact name, version scheme, the exact reverse action] |
| What data reverts? | [tables, restore layer from Table 7, RPO, RTO. Both terms are Modern] |
| What side effect cannot revert? | [the effect, and the compensating action, or "none exists"] |
| Who may press the button? | [actor from Table 9, and the approver] |

### Reverse Path
1. [Step, one instruction per line]
2. [Step]
3. [Step]
- Command surface: [the exact command or the exact label move]
- Out-of-band path: [how to reach this when the normal interface is down]
- Measured time to reverse: [minutes, from the rehearsal]
- Date of the last rehearsal: [date. "Never" blocks the merge for a 🔴 class.]

### Rollout and Exposure
| Stage | Exposure | Bake time | Metric that gates the next stage | Reverse action |
|---|---|---|---|---|
| Canary | [0.1%] | [24 h] | [SLO name and threshold] | [action] |
| Region | [10%] | [24 h] | [ ] | [ ] |
| Global | [100%] | [ ] | [ ] | [ ] |

### Point of No Return
- The step that makes this change irreversible: [name it]
- The condition that permits that step: [bake period, validator result, approver]

### Hazard Scan
[Table from Section 7]

### Residual Risk
[What stays unrecoverable, and who accepted it.]
```

Three rules govern the record.

1. A 🔴 hazard blocks the merge.
2. The record is the input to `verify`. `verify` demonstrates the reverse path and writes the measured time into the record.
3. A number with no measurement is not an answer. Write "not measured" instead.

---

# 9. LANGUAGE DISCIPLINE

These phrases are **forbidden** in a plan, a review, or a record. Each one hides the
mechanism. Replace it with the named artifact, the named actor, and the measured time.

| Forbidden | Must be replaced with |
|---|---|
| "we can always roll it back" | The artifact, the actor, and the measured time from the last rehearsal |
| "just revert the commit" | What the commit does not revert. Data, configuration, and external effects |
| "the migration is backward compatible" | The phase from Table 8, and the step that is the point of no return |
| "we have backups" | The tier from Table 7, the retention window, and the date of the last restore test |
| "replication protects us" | The nonserving copy, its media, and its layer. SRE Ch. 26, p. 421–422 |
| "it is low risk, so no canary" | The canary stage, the exposure, and the bake time. SRE Ch. 13, p. 194 |
| "the flag turns it off" | Proof that the flag reverts this change alone and at once |
| "rollback is automatic" | The monitor that gates each stage, and the actor that holds the stop control |
| "we can restore from the replica" | The point-in-time restore, the RPO, and the RTO |
| "the agent can undo it" | The branch, the checkpoint, and the owner of the shared branch |
| "we tested it in staging" | The date of the last rehearsal of the reverse path against production |
| "it is only a config change" | The validation check, and the previous configuration version |
| "we will fix forward" | The corrective change, its own reverse path, and the data that stays wrong |

---

# 10. GLOSSARY

This skill uses one word for one meaning. Use these words in your output.

| Word | Meaning in this skill |
|---|---|
| **checkpoint** | A named commit that marks a state you can return to |
| **revert** | To add a change that undoes an earlier change. History stays intact. |
| **rollback** | To return a running system to a previous artifact or configuration |
| **restore** | To write data back from a backup or from a point in time |
| **compensating action** | A new effect that cancels an effect that already left the system |
| **reverse path** | The ordered steps that undo one change, plus the actor for each step |
| **blast radius** | The users, rows, regions, and money that the change reaches |
| **bake** | The observation period for a stage, before the next stage starts |
| **canary** | A small set of servers that runs the new version under real traffic |
| **drain** | To move traffic away from a pool before you change or remove it |
| **point of no return** | The step after which a restore is the only reverse action |
| **expand, migrate, contract** | The three phases of a schema change. Contract is destructive. |
| **soft deletion** | A mark that hides data from the application and keeps it for a delay |
| **label move** | Applying a release label to a different artifact version |
| **push** | Any change to a service's running software or its configuration |
| **record** | To write information into a document |

---

# 11. REFERENCE INDEX

| Reference | Sources | Read it when |
|---|---|---|
| `references/01-the-reversibility-contract.md` | SRE Ch. 8, Ch. 27. Release It! Sec. 18.4 | You plan any change |
| `references/02-release-engineering.md` | SRE Ch. 8 | You cannot rebuild or identify the prior state |
| `references/03-progressive-rollout-and-canary.md` | SRE Ch. 17, Ch. 27. Release It! Sec. 18.4 | You decide the exposure ladder |
| `references/04-change-induced-emergency.md` | SRE Ch. 13 | A change caused the outage |
| `references/05-locating-the-bad-change.md` | SRE Ch. 12. Release It! Ch. 16 | You do not know which change caused the fault |
| `references/06-data-and-schema-reversibility.md` | SRE Ch. 26 | The change touches data or schema |
| `references/07-automation-safety.md` | SRE Ch. 7 | An automated actor performs the change or the revert |
| `references/08-agent-and-operator-shortcuts.md` | **Modern**, built on SRE Ch. 7 and Ch. 8 | You need the exact command |
| `references/09-launch-coordination.md` | SRE Ch. 27, App. E | You build the pre-merge gate |
| `references/rollback-hazard-catalog.md` | Cross-cutting | Every review. Every plan for a change with a blast radius. |

**Adjacent references in the sibling skills.** This skill does not restate them.

| Overlap | Owner | What this skill does instead |
|---|---|---|
| Pipeline phases and phase state | `engineer-workflow` | Supplies a gate inside Design and Detail Planning, and a mode inside Verification |
| The phased plan and the architecture gate | `planner` | Adds two fields per phase. The reversibility class and the reverse path. |
| The rollback section and the schema compatibility matrix | `detail-planning` | Supplies the content of that section, plus Table 8 and the point-of-no-return rule |
| Expand, migrate, and contract execution | `implement` | States which phase makes the change irreversible |
| The failure-mode matrix and the evidence for a phase | `verify` | Adds one step. Demonstrate the reverse path and record the measured time. |
| Finding format, severity ranking, review modes | `code-review` | Supplies the `R-` catalog as a review mode |
| Deployment topology and capacity | `system-design` | States only the reversal property of each topology |
| Replication, isolation, encoding, and schema evolution | `data-systems-design` | Owns the restore side. Backup tiers, restore proof, soft deletion, retention. |
| Durable writes, atomic rename, and `fsync` | `systems-programming` | Points to those recipes for a local file restore. It repeats none of them. |
| Access control and audit records | `security-engineering` | States the requirement in Table 9. The sibling states the mechanism. |
| Incident command, roles, the written incident state, the postmortem | `incident-response` | States only the order of the reverse moves inside that command structure |
| The troubleshooting loop, hypothesis testing, and the Chapter 12 war stories | `production-troubleshooting` | Owns only the change-attribution half. Which change caused this, and can we say "no change"? |
| Alert design, dashboards, and the deploy marker | `observability` | States the requirement for a deploy marker and a gating metric. The sibling states the mechanism. |

---

# 12. PROPORTIONALITY RULE

Rigor scales with the blast radius of the change. A full Reversal Record for a copy change
is itself a failure mode.

| Change class | Required output |
|---|---|
| One file, no persisted state, no deploy | Nothing from this skill |
| A code change behind an existing flag | The four questions only |
| A configuration change | The four questions, plus the validation check R-23 |
| A new migration, a backfill, or a delete path | Full Reversal Record, plus the data section |
| A change to automation that acts in bulk | Full Reversal Record, plus group G of the catalog |
| A change with money, an external side effect, or user data | Full Reversal Record, plus a named human approver |

**This skill stays silent for a change that reaches no persisted state and no deploy.**
"Not applicable" is a valid result. A single-process change with no state needs no record.

**Simplicity reduces the reverse cost.** A release of 100 unrelated changes makes
attribution expensive. A release of one change is easy to attribute and easy to revert.
SRE Ch. 9, p. 135. Release It! Sec. 18.4, p. 328.

---

**Attribution:** the rules, the page numbers, and the incidents in this skill and its
references come from two books.

- *Site Reliability Engineering: How Google Runs Production Systems* — Betsy Beyer, Chris
  Jones, Jennifer Petoff, and Niall Richard Murphy, O'Reilly Media, 2016.
- *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard, Pragmatic
  Bookshelf, 2007.

Content that both books predate carries the tag **Modern**. Neither book covers Git
commands, worktrees, container image tags, or an AI coding agent. This skill never
attributes modern practice to either book.
