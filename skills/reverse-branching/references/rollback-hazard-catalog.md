# Rollback Hazard Catalog

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016),
Ch. 6–9, Ch. 12–17, Ch. 26–27, Ch. 33, App. B, App. D. *Release It! Design and Deploy
Production-Ready Software* — Michael T. Nygard (Pragmatic Bookshelf), Sec. 10.4, Sec. 14.2,
Sec. 18.2–18.4.

A catalog of named failure modes that make a change hard to undo. The prefix is `R-`.

Each entry has three parts.

- **Signature** — what the code, the configuration, the design, or the process looks like.
  This is the text that you search for.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How to use it.** `code-review` runs the applicable groups against a difference. `verify`
runs them against an implemented phase. `planner` and `detail-planning` run them against a
plan. Report only the hazards that apply. "Not applicable. This change touches no persisted
state and no deploy." is a valid result. That result is better than an invented finding.

**Severity.** 🔴 causes data loss, corruption, or an unrecoverable state. It blocks the
merge. 🟡 causes an outage or a wrong result under load. 🔵 is a risk to operation or to
maintenance.

**The codes are stable.** One code names one hazard for the life of the catalog. A new entry
takes the next free number. The numbers inside a group are therefore not always contiguous.
The catalog holds 50 entries, R-01 through R-50, in eight groups.

**Every claim carries a chapter and a page.** An entry that carries **Modern** describes
practice that both books predate. Never attribute that practice to either book.

---

## A. The reversibility contract

### 🔴 R-01 — A change with no declared reverse path
**Signature:** a pull request, a plan file, or an `executor.md` phase with no rollback
section. A merged change whose description names no undo step.
**Consequence:** the team invents the reverse action during the outage. Nobody rehearsed the
action. Time pressure is high and the operator guesses.
**Fix:** write the Reversal Record before the merge. Name the action, the owner, and the risk
for each step. SRE Ch. 27, p. 461.

### 🔴 R-02 — An untested rollback procedure
**Signature:** a rollback step in a runbook that no test job runs. No rehearsal date appears
in the Reversal Record.
**Consequence:** the rollback fails at the moment you need it. Google attempted a permissions
rollback, the attempt failed, and the outage grew longer.
**Fix:** test the rollback procedure before the change ships. SRE Ch. 13, p. 190–191.

### 🔴 R-03 — An undeclared irreversible side effect
**Signature:** the change sends mail, calls a payment interface, publishes to a partner, or
erases a disk. The plan says "rollback: revert the commit".
**Consequence:** the code revert restores the code. The effect stays in the world. No commit
removes it.
**Fix:** name the compensating action, or block the change. Diskerase destroyed the disks and
forced a manual reinstall. Most capacity returned within three days. SRE Ch. 13, p. 194–195.

### 🟡 R-04 — A revert path that runs rarely
**Signature:** the revert job, the failover, or the restore last ran months ago. No schedule
exercises it.
**Consequence:** automation that runs at long intervals is fragile. The feedback cycle is
long, so a defect in the path stays hidden.
**Fix:** exercise the path on a schedule. Make the routine case use the same path. SRE Ch. 7,
p. 103.

### 🔴 R-05 — A revert path that runs through the failing system
**Signature:** the only revert control is a web console that the system under repair serves.
No command-line path exists.
**Consequence:** the fault removes the tool that repairs the fault. Google's troubleshooting
stack sat behind the jobs that were crash-looping.
**Fix:** keep command-line tools and other access methods that work when the normal
interfaces do not. Test them routinely. SRE Ch. 13, p. 193–194.

---

## B. Build and artifact identity

### 🔴 R-06 — A non-hermetic build
**Signature:** a build script reads a library from the build machine. It calls a network
service at build time. It pins no dependency version.
**Consequence:** two builds of one revision produce different artifacts. You cannot
reconstruct the artifact that you must return to.
**Fix:** make the build self-contained. Pin every compiler and every library to a known
version. Two engineers must get identical results at one revision. SRE Ch. 8, p. 122.

### 🟡 R-07 — An unpinned build toolchain
**Signature:** a CI file names a compiler or an image tag of `latest`. No toolchain version
follows the source revision.
**Consequence:** a rebuild of an old revision uses today's compiler. That compiler can hold
features that the old code did not expect.
**Fix:** version the build tools by the source revision of the project. SRE Ch. 8, p. 122.

### 🔴 R-08 — A mutable artifact tag
**Signature:** a deploy names an image tag such as `latest` or `prod`. A package name holds
new content at a later date.
**Consequence:** "return to the previous artifact" resolves to unknown content. Two operators
read one name and get two artifacts.
**Fix:** name the package, version it with a unique hash, and sign it. Move a label. Never
change the content behind a name. SRE Ch. 8, p. 124–125.

### 🔵 R-09 — A binary with no build identity
**Signature:** no version flag, no build identifier in the startup log, and no version header
in a response.
**Consequence:** you cannot map a running process to a commit. You therefore cannot say which
artifact the operator must revert.
**Fix:** make every binary report the build date, the revision number, and the build
identifier. SRE Ch. 8, p. 123.

### 🟡 R-10 — A release with no change list
**Signature:** a deploy record that lists no commits since the previous deploy. No report is
archived beside the build artifacts.
**Consequence:** the search has no candidate set. The first diagnostic question has no data
to work on.
**Fix:** produce a report of all changes in the release. Archive it with the build artifacts.
SRE Ch. 8, p. 123 and p. 126.

---

## C. Branch and commit hygiene

### 🔴 R-11 — A history rewrite on a shared branch
**Signature:** `git reset --hard`, `git rebase`, or `push --force` against a branch that other
people use. The command sits in a script, an agent turn, or a runbook.
**Consequence:** other clones diverge. Work disappears, and no record names the actor who
removed it.
**Fix:** add a revert commit instead. The revert keeps every clone valid and leaves an audit
trail. **Modern**, built on the audit-trail rule in SRE Ch. 7, p. 113.

### 🟡 R-12 — A release built from a moving head
**Signature:** a deploy job builds from the main branch at deploy time. No release branch
exists, and no revision is pinned.
**Consequence:** the release holds changes that nobody selected. You cannot remove one change
without removing the others.
**Fix:** branch at a specific revision. Never merge that branch into the mainline. Fix on the
mainline, then cherry pick. SRE Ch. 8, p. 123–124.

### 🟡 R-13 — A stash used as a checkpoint
**Signature:** `git stash` before a risky edit. No commit and no tag mark the prior state.
**Consequence:** the entry carries no name and no message. A second stash hides the first. A
clean operation discards it.
**Fix:** commit a named checkpoint on a work branch. Tag the checkpoint when you must find it
later. **Modern**.

### 🔵 R-14 — Agent edits with no checkpoint
**Signature:** an agent session with many uncommitted file edits. No baseline commit marks
the state before the session.
**Consequence:** no state exists to return to. A partial undo leaves a mixed configuration
that nobody can describe.
**Fix:** commit a checkpoint before the risky edit. Record the change, so you can return to
the state before the test. SRE Ch. 12, p. 180. **Modern** for the command.

### 🟡 R-15 — One change that moves two systems at once
**Signature:** a single change edits both the producer and the consumer of one protocol, one
file format, or one interface.
**Consequence:** neither side reverts alone. Nygard calls this the Big Bang upgrade, and both
systems need downtime.
**Fix:** version the protocol, so either endpoint changes alone. Update the receiver before
the sender. Release It! Sec. 18.3, p. 323–325, and Sec. 18.4, p. 332.

---

## D. Rollout and exposure

### 🔴 R-16 — A one-step global rollout
**Signature:** a deploy workflow with one job that targets 100% of production. No stage, no
percentage, and no region order appear in the file.
**Consequence:** the outage is the first observation of the change. The blast radius is the
whole service.
**Fix:** stage the rollout. Apply the change to a small fraction of traffic and capacity at a
time, in different geographies. SRE App. B, p. 572. SRE Ch. 27, p. 461–462.

### 🔴 R-17 — A risk-based canary exemption
**Signature:** a pipeline rule skips the canary for a change that a label marks low risk,
configuration only, or documentation only.
**Consequence:** an untested combination fails everywhere. One change judged non-risky took a
less strict canary path and crashed the fleet.
**Fix:** canary every change, whatever its perceived risk. SRE Ch. 13, p. 194.

### 🟡 R-18 — A rollout with no bake time
**Signature:** rollout stages advance on job success. No observation window and no metric gate
the move to the next stage.
**Consequence:** the change reaches full exposure before the fault becomes visible. A slow
fault meets no gate at all.
**Fix:** define an observation period for each stage. Let two servers bake the new code for a
day or more. SRE Ch. 27, p. 461–462. Release It! Sec. 18.4, p. 334.

### 🟡 R-19 — A rollout judged by the actor that pushed it
**Signature:** the deploy script reads its own metrics and decides to continue. No independent
monitor gates the stage.
**Consequence:** the actor that wants to finish also holds the stop authority. The rollout
advances over a weak signal.
**Fix:** let a monitoring system that you can demonstrate to be reliable supervise every
stage. SRE App. B, p. 572. Separate the stop authority from the party that ships. SRE Ch. 33,
p. 562.

### 🟡 R-20 — A feature flag that cannot revert alone
**Signature:** one flag gates several unrelated changes. Or the operator needs a deploy before
the flag can change.
**Consequence:** the reverse action removes good changes with the bad one. The operator
hesitates, and the outage runs longer.
**Fix:** a flag framework must revert each change alone and at once. SRE Ch. 27, p. 463.

### 🔴 R-21 — Dead code kept behind a disabled flag
**Signature:** an `if (false)` block, a commented block, or a flag that a comment describes as
kept for rollback.
**Consequence:** a dormant code path becomes active later. SRE names dead code a time bomb and
cites Knight Capital.
**Fix:** delete the code. Source control reverses the change when you need it again. SRE
Ch. 9, p. 132–133.

### 🟡 R-46 — Session failover across two versions
**Signature:** one load balancer pool holds servers on the old version and the new version
during the rollout. No pool separation exists.
**Consequence:** a session starts on one version. The balancer then moves it to a server on
the other version, and the session state does not match.
**Fix:** create two service pools during the overlap. Keep request failover and session
failover inside one version. Release It! Sec. 18.4, p. 334.

---

## E. Configuration reversal

### 🔴 R-22 — Configuration outside version control
**Signature:** an operator changes a production setting through a console or an environment
variable. No file in the repository holds the value.
**Consequence:** no prior version exists to return to. No record names the actor, and no
record gives the reason for the change.
**Fix:** store all code and all configuration files in version control, under review. SRE
Ch. 27, p. 459. Release It! Sec. 14.2, p. 245.

### 🔴 R-23 — Configuration accepted with no validation
**Signature:** a loader reads the new file and serves it. No syntax check, no size check, and
no comparison against the previous version.
**Consequence:** the service serves the bad file at once. SRE names empty data, partial data,
and truncated data as the categories to expect.
**Fix:** validate syntax and semantics. Alert when the new file is N percent smaller than the
previous one. Keep the previous state and wait for a person. SRE App. B, p. 571–572.

### 🟡 R-24 — Configuration drift after an emergency edit
**Signature:** an operator edits a file on the host during an incident. The repository never
receives the edit.
**Consequence:** the repository and the running system disagree. A deploy from the repository
then reverses the fix, and no warning appears.
**Fix:** run an audit job that reports the difference. The job must not overwrite the
difference. Release It! Sec. 14.2, p. 245.

### 🔵 R-25 — Runtime settings not reverted with the code
**Signature:** garbage collection flags, pool sizes, or timeouts tuned for the new release.
The revert returns the code alone.
**Consequence:** the old code runs under settings that another release needed. Settings that
are correct for one release can be wrong for the next.
**Fix:** treat tuned settings as state that belongs to the release. Revert them with the code.
Release It! Sec. 10.4, p. 215 and p. 217.

---

## F. Data and schema reversal

### 🔴 R-26 — A code revert across a destructive migration
**Signature:** a migration drops a column, renames a column, or adds NOT NULL. The code that
needs the new shape ships in the same release.
**Consequence:** the old code reads data from the new version as corrupt or incomplete. The
revert therefore fails.
**Fix:** split the change into expand, migrate, and contract. Nygard names the three phases
Expansion, Rollout, and Cleanup. Run the contract step in a later release. Release It!
Sec. 18.4, p. 331–334.

### 🔴 R-27 — A constraint added during expansion
**Signature:** `SET NOT NULL` or a foreign key in the same change that adds the column.
**Consequence:** the old version violates the constraint at once. The two versions cannot
coexist, so no revert is safe.
**Fix:** add every future NOT NULL column as nullable. Defer every constraint to the contract
step. Release It! Sec. 18.4, p. 333.

### 🔴 R-28 — Replication treated as the backup
**Signature:** a recovery plan names replicas, a multi-region cluster, or a synchronized store
as the restore path.
**Consequence:** a corrupt row or an errant delete reaches every copy. It often arrives before
you can isolate the problem.
**Fix:** keep nonserving copies on diverse components at each layer. Replication and
redundancy are not recoverability. SRE Ch. 26, p. 421–422.

### 🔴 R-29 — A restore that was never run
**Signature:** a backup job with no automated restore test. The Reversal Record holds no
measured restore time.
**Consequence:** the recovery process stays in a latent broken state. You discover the break
during the incident.
**Fix:** test the recovery process continuously as part of normal operation. Alert when a
recovery run sends no success heartbeat. SRE Ch. 26, p. 434–435.

### 🔴 R-30 — A hard delete with no soft delete
**Signature:** a `DELETE` statement in an application path or a batch job. No tombstone and no
administrative undelete path exist.
**Consequence:** SRE names the most severe acute deletion cases as the work of developers who
did not know the existing code.
**Fix:** mark the data deleted, keep it for a defined delay, and add an undelete path. Common
delays are 15, 30, 45, or 60 days. SRE Ch. 26, p. 423–426.

### 🟡 R-31 — A retention window shorter than the detection window
**Signature:** seven days of backups. The validators run weekly, or no validator runs at all.
**Consequence:** a slow corruption is older than every restore point when you find it. SRE
found such defects weeks or months after the release.
**Fix:** set retention from the release cadence and the detection latency. Google drew the
line between 30 and 90 days. SRE Ch. 26, p. 421 and p. 428.

### 🟡 R-32 — An archive presented as a backup
**Signature:** the recovery plan names a long-term store. No application can load the data
back from that store.
**Consequence:** recovery cannot meet the uptime requirement. Users have no access from the
start of the fault to the end of recovery.
**Fix:** keep a tier that an application can load. A backup loads back into the application.
An archive does not. SRE Ch. 26, p. 416–417.

### 🟡 R-33 — No alert on the deletion rate
**Signature:** no alert watches the deletion rate across all users. Only per-user thresholds
exist, or no threshold exists.
**Consequence:** a runaway deletion runs until a user reports it. Google Music lost about
600,000 audio references before that report. SRE Ch. 26, p. 438–439.
**Fix:** alert when the deletion rate across all users crosses an extreme threshold, such as
10 times the observed 95th percentile. SRE Ch. 26, p. 447, footnote 133.

### 🟡 R-47 — An implicit column list in a write or a read
**Signature:** an `INSERT` or an `UPDATE` that names no column list. A `SELECT *` in
application code, in a report, or in a business intelligence query.
**Consequence:** the statement breaks when the expand step adds a column. The two versions
then cannot serve at the same time.
**Fix:** name explicit columns and values in every write. Remove every `SELECT *`. Apply the
rule from the start of the project. Release It! Sec. 18.4, p. 333.

### 🟡 R-48 — No schema version check at start-up
**Signature:** the schema holds no structure revision table. The application starts against
any schema version.
**Consequence:** a reverted binary runs against a newer schema. Nygard reports strange runtime
errors, transaction rollbacks, or corrupt data.
**Fix:** add a revision table with one row and one column. Refuse to start on an incompatible
version. Raise the number for a change in interpretation too. Release It! Sec. 18.2, p. 319.

### 🟡 R-49 — Data destroyed before its first backup
**Signature:** a deletion path or a bulk job acts on records that the backup schedule has not
yet covered.
**Consequence:** the restore point holds none of that data. Google Music lost 161,000 tracks
before any backup held them.
**Fix:** state the recovery point objective for the change. The term is **Modern**. Name the
upstream source that can rebuild the gap. SRE Ch. 26, p. 440 and p. 442.

---

## G. Automation and agent safety

### 🔴 R-34 — An empty target set treated as every target
**Signature:** a selector with no lower bound. Code that runs the action when the filter list
is empty.
**Consequence:** the empty set becomes a value that means everything. Google's automation
erased the disks of every machine in every colo.
**Fix:** treat an empty or degenerate target set as an error. Add sanity checks to the
commands that automation sends. SRE Ch. 7, p. 117–118. SRE Ch. 13, p. 196.

### 🔴 R-35 — Automated reversal with no rate limit
**Signature:** a revert loop, a repair loop, or a bulk operation with no cap on hosts, rows,
or actions per minute. No stop control exists.
**Consequence:** one logic defect becomes a fleet-wide event in minutes.
**Fix:** add rate limiting inside the automation. SRE Ch. 7, p. 118. Give an operator a stop
control. SRE Ch. 13, p. 195.

### 🔴 R-36 — A non-idempotent workflow restarted from step one
**Signature:** a multi-step job with no checkpoint. A retry runs the job again from the
beginning.
**Consequence:** the second run reads state that the first run already changed. This is the
exact shape of the Diskerase incident.
**Fix:** make the workflow idempotent, so a repeated run causes no damage. SRE Ch. 7, p. 110
and p. 118.

### 🔴 R-37 — Safety inferred from an absent signal
**Signature:** automation reads missing data as permission to act. An unused disk, an empty
directory, or a missing lock file means "safe to erase".
**Consequence:** a deliberate configuration exception becomes a hazard. Google erased a
multi-petabyte Bigtable dataset at once.
**Fix:** require an explicit positive signal before a destructive action. SRE Ch. 7, p. 107.

### 🟡 R-38 — Automated repair that deletes data
**Signature:** a detector whose remediation deletes or rewrites user data. No person stands in
the path.
**Consequence:** the repair causes more loss than the defect. SRE states that the value of
such a tool is the report.
**Fix:** report the problem before it becomes an outage. Do not repair a data defect by
deleting user data. SRE Ch. 17, p. 242.

### 🟡 R-39 — A repair loop with no stop rule
**Signature:** a fix that retries with no bound and no escalation to a person.
**Consequence:** the loop hides a real fault. It also consumes the production path that users
need.
**Fix:** stop after repeated failure. Notify a person. SRE Ch. 7, p. 110.

### 🔴 R-40 — Two actors writing to production at once
**Signature:** an agent holds deploy rights while a human operator also deploys. No single
write holder exists during the incident.
**Consequence:** an uncoordinated change during an incident made a bad situation far worse.
**Fix:** name one write holder for the incident. Every other participant proposes a change to
that holder. SRE Ch. 14, p. 201–203.

---

## H. Detection, attribution, and record

### 🟡 R-41 — No change annotation on the metric timeline
**Signature:** dashboards with no deploy marker and no configuration-change marker. No
production log records version pushes at each layer.
**Consequence:** the first diagnostic question has no answer. Nobody can say what touched the
service last.
**Fix:** log new version deployments and configuration changes at all layers of the stack.
Annotate the error graph with the start and the end of each deploy. SRE Ch. 12, p. 177–178.

### 🟡 R-42 — Correlation accepted as cause
**Signature:** an operator chooses a revert because two metrics moved together. No test
separates the candidate causes.
**Consequence:** the team reverts or rebuilds the wrong thing. One Google team followed a
`merge_join` theory and built indexes that it did not need.
**Fix:** design a test with mutually exclusive outcomes. Run the tests in decreasing order of
likelihood. SRE Ch. 12, p. 171, p. 179, and p. 184–185.

### 🟡 R-43 — A revert trigger on a flaky signal
**Signature:** an automatic rollback wired to a test or a metric whose repeated runs give
different answers.
**Consequence:** the automation reverts good changes and thrashes the fleet. SRE computes that
42,000 test results demand over 99.9999% correctness per test.
**Fix:** measure the flake rate first. Hold the signal to the accuracy that the volume
demands. SRE Ch. 17, p. 236–237.

### 🔵 R-44 — A revert that leaves no record
**Signature:** a rollback with no incident record and no cause tag. No note says that the
revert did not fix the symptom.
**Consequence:** the next responder repeats the same failed revert. The lesson never reaches
the team.
**Fix:** treat a release rollback as a postmortem trigger. Record the cause, the action, and
the result. SRE Ch. 15, p. 209. SRE Ch. 16, p. 219.

### 🔵 R-45 — Monitoring removed by the change
**Signature:** a deploy workflow or a teardown workflow also removes alerts, dashboards, or
probes for the affected component.
**Consequence:** the automation blinds the responders at the moment they need sight. Google's
on-call engineers reverted the monitoring changes before they could assess the damage.
**Fix:** revert the monitoring change first. Keep observability changes on a separate reverse
path. SRE Ch. 13, p. 196.

### 🟡 R-50 — An alert storm that blocks the reversal
**Signature:** one fault pages every service. No rule groups the related alerts into one
incident.
**Consequence:** the alerts fired repeatedly and overwhelmed the on-call engineers. They also
filled the channels that the reversal needs.
**Fix:** group related alerts into one incident. Keep the alert path separate from the revert
path. SRE Ch. 13, p. 193–194. SRE Ch. 16, p. 219.

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| The revert did not fix the symptom | R-26, R-30, R-42, R-44 |
| We cannot rebuild the prior artifact | R-06, R-07, R-08 |
| We do not know which change caused it | R-09, R-10, R-12, R-41 |
| The revert failed when we ran it | R-02, R-04, R-05, R-29 |
| The old code cannot read the data | R-26, R-27, R-47, R-48 |
| The data is gone from every copy | R-28, R-30, R-33, R-49 |
| The automation damaged the whole fleet | R-34, R-35, R-36, R-37 |
| The repository and production disagree | R-22, R-24, R-25 |
| The fault appeared only at full exposure | R-16, R-17, R-18, R-19 |
| The responders could not see or communicate | R-05, R-45, R-50 |
| Two actors changed production at once | R-15, R-40, R-46 |

---

## Where to read the detail

| Group | Reference file |
|---|---|
| A. The reversibility contract | `01-the-reversibility-contract.md` |
| B. Build and artifact identity | `02-release-engineering.md` |
| C. Branch and commit hygiene | `08-agent-and-operator-shortcuts.md` |
| D. Rollout and exposure | `03-progressive-rollout-and-canary.md` |
| E. Configuration reversal | `01-the-reversibility-contract.md` |
| F. Data and schema reversal | `06-data-and-schema-reversibility.md` |
| G. Automation and agent safety | `07-automation-safety.md` |
| H. Detection, attribution, and record | `05-locating-the-bad-change.md` |

A change-induced outage sends you to `04-change-induced-emergency.md` first. A pre-merge gate
sends you to `09-launch-coordination.md`.
