# The Reversibility Contract

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 8 "Release Engineering", p. 120–130. Ch. 27 "Reliable Product Launches at Scale",
p. 448–470. *Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007). Sec. 18.4
"Releases Shouldn't Hurt", p. 327–335. Supporting citations name their own chapter.

Read this file when you plan any change. It defines the contract that every other file in
this skill enforces.

---

## 1. The contract

**Every change declares how a person undoes it before the change ships.**

That declaration is the reversibility contract. It answers four questions. It names one
class. It names one point of no return. The author writes it. The reviewer checks it.

SRE Ch. 27, p. 461 states the rule for a launch plan. Name the actions. Give each item an
owner. Identify the risk in each step. Implement a contingency measure for each risk. The
plan exists before the launch starts.

**The contract binds one change, not one release.** SRE Ch. 27, p. 463 requires a feature
flag framework to revert each change alone and at once. A release-wide reverse action is a
blunt tool. It removes good changes with the bad one.

**A change is any edit that reaches production, including configuration.** SRE Ch. 8, p. 127
states that configuration changes are a potential source of instability. SRE Ch. 27, p. 449
defines a launch as "Any new code that introduces an externally visible change to an
application." Under that definition Google counted up to 70 launches per week. p. 449.

**"Running reliable services requires reliable release processes."** SRE Ch. 8, p. 120. The
contract is how you make the reverse half of that process reliable.

---

## 2. Why the contract exists at plan time, and not at incident time

Four reasons. Each one has a source. Each one is a cost that you pay later if you skip it.

**Reason 1. The incident gives the responder no time to design a reverse path.** SRE Ch. 13,
p. 191 records the outcome. Google had not tested its rollback procedures in a test
environment. Those procedures were flawed. The outage lasted longer. The book now requires
thorough testing of rollback procedures before a large-scale test.

**Reason 2. Reversibility is a property of the deployment sequence.** Release It! Sec. 18.4,
p. 331 divides a deployment into three phases so that version N and version N+1 coexist.
Nygard names them Expansion, Rollout, and Cleanup. During Expansion and Rollout, a revert of
the application tier is a code-only action. The old columns still hold current data. The old
assets still exist at their own URLs (p. 332–334). Cleanup ends that property. You choose the
phasing when you design the change. You cannot choose it during the outage.

**Reason 3. Reversibility is bought at build time.** SRE Ch. 8, p. 122 requires a hermetic
build. Two people who build the same revision on different machines expect identical
results. SRE Ch. 8, p. 124 requires packages that are named, versioned with a unique hash,
and signed. Without those two properties, "return to the previous artifact" names content
that nobody can reconstruct.

**Reason 4. The reverse action costs money whether you plan it or not.** Release It!
Sec. 18.1, p. 311 names the total price of a change as activation energy. It is design
effort, plus development, plus testing, plus the cost of release. Section 6 below prices one
four-hour deployment at $40,000.

**The signal that you skipped the contract:** the change description says "rollback: revert
the commit". The same change also writes data, calls a partner, or drops a column.

---

## 3. The four questions

A change answers all four before it merges. An unanswered question is a defect, not a gap.

| # | Question | A complete answer names | Source |
|---|---|---|---|
| 1 | What artifact reverts? | The artifact, its version scheme, and the exact reverse action | SRE Ch. 8, p. 124–125 |
| 2 | What data reverts? | The tables, the restore layer, the data the restore loses, and the restore time | SRE Ch. 26, p. 421–429 |
| 3 | What side effect cannot revert? | The effect, and the compensating action, or the words "none exists" | SRE Ch. 13, p. 196–197 |
| 4 | Who may press the button? | The actor, and the approver for each destructive step | SRE Ch. 8, p. 122–123 and p. 125 |

### Question 1 — What artifact reverts?

**Name the artifact, not the commit.** A commit is the source. An artifact is the thing that
runs. SRE Ch. 8, p. 124–125 makes the reverse action cheap. Google labels a package version
`dev`, `canary`, or `production`. Applying an existing label to a new package moves the label
off the old package. A revert is a label move back to the previous package.

Four properties make the answer possible. Check each one.

1. The build is hermetic and pinned to known versions of tools and libraries. SRE Ch. 8, p. 122.
2. The build tools are versioned by the source revision of the project. SRE Ch. 8, p. 122.
3. Every binary reports its build date, revision number, and build identifier. SRE Ch. 8, p. 123.
4. The release carries a report of all changes since the previous release. SRE Ch. 8, p. 123 and p. 126.

**Property 2 is the one that teams miss.** A rebuild of last month's revision with this
month's compiler is not the artifact you ran last month. SRE Ch. 8, p. 122 states that the
newer compiler may contain incompatible or undesired features.

**A configuration-only change needs its own artifact answer.** SRE Ch. 8, p. 128 shows the
cheaper path. Google builds two packages, one for the binary and one for the configuration.
A wrong flag value is fixed by a cherry pick of the configuration change, a rebuild of the
configuration package, and a deploy. No new binary build is needed.

### Question 2 — What data reverts?

**A code revert does not move data.** Answer this question separately from question 1, even
when one change causes both.

State three things. State the layer that holds the recoverable copy. State the age of the
oldest copy. State the measured time to restore.

SRE Ch. 26, p. 421 gives the rule that most plans break. Replication and redundancy are not
recoverability. A wrong delete reaches every replica. SRE Ch. 26, p. 425 gives the common
soft-deletion delays. They are 15, 30, 45, or 60 days. SRE Ch. 26, p. 428 states that Google
draws the backup line between 30 and 90 days for many services.

**Set the retention window from the release cadence and the detection latency.** SRE Ch. 26,
p. 428 gives the rule of thumb. A team that pushes new code twice a week keeps those backups
for at least a week or two.

### Question 3 — What side effect cannot revert?

**Some effects leave the system and never return.** A sent message, a payment call, a file
delivered to a partner, and an erased disk are the four common shapes.

Name the effect. Then name the compensating action. A compensating action is a new forward
action that cancels the business meaning of the effect. A refund compensates a charge. A
correction notice compensates a wrong message. **No compensating action restores an erased
disk.**

SRE Ch. 13, p. 194–197 shows the extreme case. Automation sent the machines of every small
installation to the Diskerase queue. The drives were wiped. The reverse path was a manual
reinstall. Most capacity returned within three days. Stragglers took a month or two.

**The required output is a sentence, not a plan.** Write "the compensating action is X", or
write "none exists". The second answer is valid. It changes who approves the change, and it
changes the exposure ladder.

### Question 4 — Who may press the button?

**Name the actor for each step of the reverse path.** SRE Ch. 8, p. 122–123 lists the gated
release operations. They include approving source changes, creating a release, approving
each cherry pick, and deploying a release. SRE Ch. 8, p. 125 puts role-based access control
lists on those actions.

Three rules follow.

1. Document every manual step so that any team member can run it in an emergency. SRE Ch. 27, p. 459.
2. Treat a human as a single point of failure, and remove that dependency. SRE Ch. 27, p. 459.
3. Give a monitoring system that you can demonstrate to be reliable the authority to stop a stage. The book prefers it to the engineer who runs the stage. SRE App. B, p. 572.

SRE Ch. 27, p. 462 states the automatic form. Google's tools observe a newly started server
for a validation period. The tool reverts the change automatically when the change does not
pass that period.

---

## 4. The six reversibility classes

Classify the change before you answer the four questions. The class sets the reverse action,
the time to reverse, and the approver.

| Class | The change looks like | Reverse action | Time to reverse | Point of no return |
|---|---|---|---|---|
| Pure code | A function body. No persisted state. | Revert the commit. Deploy the prior artifact. | Minutes | None |
| Configuration | A flag value, a timeout, a route, an allowlist | Deploy the previous configuration version | Seconds to minutes | None, while the configuration is versioned |
| Schema, additive | A nullable column, a new table, a new endpoint | Stop writing the field. Remove it in a later release. | Minutes | The Cleanup step |
| Schema, destructive | A dropped column, a rename, a new NOT NULL | Restore the data from a backup | Hours | The moment the statement commits |
| Data | A backfill, a bulk update, a delete | Undo the soft delete, or restore to a point in time | Minutes to days | The end of the retention window |
| External side effect | A sent message, a payment call, a partner file, a wiped disk | A compensating action, when one exists | Never, for some | The moment the effect leaves the system |

**A change inherits the class of its least reversible part.** A change that edits one
function and also drops one column is a destructive schema change. Classify it that way. The
reviewer applies the rules of the higher class to the whole change.

**A change is irreversible when its class has no reverse action.** The external side effect
class is the only class that reaches this state by default. A destructive schema change
reaches it when no backup covers the affected rows. Report an irreversible change as
irreversible. Do not describe it as "hard to reverse".

### 4.1 Pure code

**The signal:** the difference touches no schema file, no configuration file, no migration,
and no outbound call.

The reverse action is a revert commit plus a deploy of the prior artifact. This is the only
class where "we can revert it" needs no further evidence.

### 4.2 Configuration

**The signal:** the difference changes a value that the running system reads, and the value
lives outside the binary.

SRE Ch. 8, p. 127 requires configuration in the primary source repository under strict code
review. The reverse action is then a deploy of the previous configuration version. SRE Ch. 8,
p. 127 also names the failure of the loosest model. Configuration skew appears when the
checked-in file and the running file differ, because a job must restart to read the change.

**Configuration that lives only in a console has no previous version.** That change is not in
this class. It is an untracked change with no reverse path at all.

### 4.3 Schema, additive

`data-systems-design/references/04-encoding-and-evolution.md`, Section 4, owns the
compatibility mechanics of expand, migrate, and contract. That file holds the backfill rules,
the locking rules, and the tag-reuse rules. This section states only the reversibility
consequence of each phase.

**The signal:** the migration adds a nullable column, a table, an endpoint, or a versioned
asset URL, and removes nothing.

Release It! Sec. 18.4, p. 333 gives the two rules that keep this class reversible. Add every
future NOT NULL column as nullable, because the old version does not know how to fill it in.
Add no referential integrity rule yet, because the old version would violate it at once.

Release It! Sec. 18.4, p. 332 adds the asset rule. Give each revision of a URL-based asset
its own new URL. Give each revision of a service interface a new endpoint name. Never
overwrite the old name.

### 4.4 Schema, destructive

**The signal:** the migration drops a column, renames a column, adds NOT NULL, or adds a
foreign key.

Release It! Sec. 18.4, p. 333 states the failure that follows a code revert across this step.
The old version reads data from the new version as corrupt or incomplete. The code revert
therefore does not restore service.

The reverse action is a data restore, not a code revert. Time it in hours.
`06-data-and-schema-reversibility.md` gives the restore layers and the ladder.

### 4.5 Data

**The signal:** the change runs a backfill, a bulk update, or a delete against production
rows.

SRE Ch. 26, p. 426 orders the defense layers. A user-visible trash folder is the primary
defense against user error. Soft deletion is the primary defense against developer error.
Lazy deletion in the storage layer is the primary defense against internal developer error.

The reverse action is an undelete inside the retention window, or a restore to a point in
time. **The retention window is the point of no return for this class.**

### 4.6 External side effect

**The signal:** the change causes an action that another party observes.

The effect is outside your revert authority the moment it leaves the system. The reverse
action is a compensating action or nothing. Name which one applies before the change merges.

---

## 5. The point of no return

**Every change has one step that makes it irreversible. Name that step.**

Release It! Sec. 18.4, p. 334 names it for a schema change. The Cleanup phase removes the
bridging triggers and the extra service pools. It drops unused columns and tables. It deletes
the old static files. It converts columns to NOT NULL. It adds the referential integrity
rules. Every one of those actions removes a piece of the reverse path.
`06-data-and-schema-reversibility.md` names this step the contract step.

Two rules govern it.

1. **Never place the expand step and the Cleanup step in the same change.** Release It! Sec. 18.4, p. 332–334.
2. **Run the Cleanup step only after the release bakes.** Release It! Sec. 18.4, p. 334 lets a couple of servers run the new code for a day or more.

**State the condition that permits the irreversible step.** A bake period, a validator
result, and a named approver are the three usual conditions. A calendar date is not a
condition. Release It! Sec. 18.4, p. 330 records a meeting that approved an untested release
because the date was chosen months earlier.

---

## 6. War stories

**The wrong flag value, fixed with no binary build (SRE Ch. 8, p. 128).** A Google team
released a feature with a flag set to `first_folio`. The correct value was `bad_quarto`. The
binary and the configuration shipped as two separate packages. The fix was a cherry pick of
the configuration change, a rebuild of the configuration package, and a deploy. No new binary
build was needed. The story proves that a configuration fault has a cheaper reverse action
than a code fault. It earns that price only when the team split the packages in advance.

**Dead code kept behind a disabled flag (SRE Ch. 9, p. 132–133).** The book rejects three
answers to "what if we need that code later". Keeping it, commenting it out, and gating it
behind an always-disabled flag are all named as bad. Code that never executes is described as
a time bomb, and Knight Capital is cited as the demonstration. The remedy is deletion,
because source control already reverses the change. The story proves that a disabled flag is
not a reverse path. It is a dormant code path that a later deploy can activate.

**The release that resembles a launch sequence (Release It! Sec. 18.4, p. 327–328).** A
retailer starts its release in the afternoon and finishes in the small hours. More than
twenty people once held active roles, and under a dozen do now. Because each release is
arduous, the company performs few of them. Because it performs few, each one is unique. Each
unique release needs more planning, which makes the next release more painful. Nygard states
that the process is barely sustainable for three or four releases a year, and could never
work for twenty. The remedy is to release more often and to remove people from the process.
The story proves that reverse-path competence comes from repetition, not from documentation.

**The four-hour deployment that costs $40,000 (Release It! Sec. 18.4, p. 330–331).** A
company sets its cost of downtime at $10,000 per hour. A four-hour deployment then costs
$40,000. The operations calendar does not change that number. The same section shows the
accounting dodge. A month at 99.5% availability means fewer than 216 minutes of surprises.
The system may also have been down for five hours inside change windows. The story proves
that a plan which reaches for a downtime window has already spent the money that a reversible
design saves.

**Kill switches built before the traffic arrived (SRE Ch. 27, p. 448–449).** Keyhole serves
satellite imagery for Google Maps and Google Earth at up to several thousand images per
second. On Christmas Eve 2011 it received 25 times its normal peak, above one million
requests per second, because of a Christmas site hosted with NORAD. The launch had a deadline
that could not move, heavy publicity, and a steep traffic ramp. SRE prepared the
infrastructure and built kill switches into the experience. The team named them
"Make-children-cry switches". The story proves that a reverse action for a feature is built
before the launch, by the team that expects to need it.

---

## 7. The contract block in the Reversal Record

The skill produces a Reversal Record. These two sections carry the contract. `../SKILL.md`,
Section 8 holds the full record.

```md
### Classification
- Reversibility class: [pure code / configuration / schema additive /
  schema destructive / data / external side effect]
- Reason for the class: [the least reversible part of the change]
- Blast radius at full exposure: [users, rows, regions, money]

### The Four Questions
| Question | Answer |
|---|---|
| What artifact reverts? | [artifact, version scheme, exact reverse action] |
| What data reverts? | [tables, restore layer, data lost, restore time] |
| What side effect cannot revert? | [effect and compensating action, or "none exists"] |
| Who may press the button? | [actor, and the approver for each destructive step] |

### Point of No Return
- The step that makes this change irreversible: [name it]
- The condition that permits that step: [bake period, validator result, approver]
```

---

## 8. Language discipline

Replace each banned phrase with the required content. A reviewer rejects the phrase, not the
change.

| Banned phrase | Why it fails | Required replacement |
|---|---|---|
| "We can always roll it back" | It names no artifact, no actor, and no time | The artifact, the actor, and the measured time from the last rehearsal |
| "It's just a config change" | SRE Ch. 8, p. 127 names configuration as a source of instability | The configuration version that you deploy to reverse it |
| "Low risk, so we can skip the canary" | SRE Ch. 13, p. 194 requires a canary regardless of perceived risk | The canary stage and the metric that gates it |
| "We'll fix it forward if needed" | It defers the design of the reverse path to the incident | The condition under which a forward fix is the chosen path |
| "The migration is backward compatible" | The word hides which phase runs | The phase name, and whether a code revert stays safe in it |
| "Rollback is easy" | It states a feeling, not a procedure | The numbered steps, and the date of the last rehearsal |

---

## 9. Proportionality

**Rigor scales with the blast radius. It does not scale with the size of the difference.**

Apply the full contract when any of these is true.

- The change touches a schema, a migration, or production rows.
- The change causes an effect that another party observes.
- The change reaches more than a canary share of users.
- The change edits configuration that the running system reads.
- The change edits automation that acts on many targets.

**Skip the contract when all of these are true.** The change touches one file. It persists no
state. It calls no external party. It reverts with one commit. Write "pure code, revert the
commit" and stop.

SRE Ch. 27, p. 468 supports the light path. Google classed a launch as low risk when it
introduced no new server executables and increased traffic by under 10 percent. Those
launches received an almost trivial checklist. By 2008, 30 percent of reviews took that path.

**"Not applicable" is a valid result.** An invented hazard costs the reviewer's attention,
and attention is the resource that finds the real one.

---

## 10. Where to read next

| Signal | Read |
|---|---|
| You cannot name or rebuild the prior artifact | `02-release-engineering.md` |
| You must choose the exposure ladder and the bake time | `03-progressive-rollout-and-canary.md` |
| A change already caused the outage | `04-change-induced-emergency.md` |
| You do not know which change caused the fault | `05-locating-the-bad-change.md` |
| The change touches data or schema | `06-data-and-schema-reversibility.md` |
| An automated actor makes the change or the revert | `07-automation-safety.md` |
| You need the exact command | `08-agent-and-operator-shortcuts.md` |
| You are building the pre-merge gate | `09-launch-coordination.md` |
| You are reviewing a change | `rollback-hazard-catalog.md` |

The catalog entries that this file creates are R-01 (no declared reverse path), R-02
(untested rollback procedure), and R-03 (undeclared irreversible side effect). All three are
🔴. All three block the merge.
