# Checkpoint and Revert Shortcuts for an Agent and an Operator

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 7 "The Evolution of Automation at Google", p. 97–119. Ch. 8 "Release Engineering",
p. 120–131. Supporting pages: Ch. 6, p. 83. Ch. 12, p. 176–180. Ch. 13, p. 193. Ch. 14,
p. 201–203. Ch. 15, p. 209. Ch. 17, p. 236–242.

**Every command in this file is Modern.** Neither book names a version control tool. The
books supply the safety rule. The command is the modern form of that rule. Keep a
command-line path that works when the normal interfaces do not. SRE Ch. 13, p. 193.

---

## 1. How to read this file

Each entry gives the **signal** that triggers the command, the **command**, the **safety
rule** with its book page, and the **hazard** code in `rollback-hazard-catalog.md`. Two
actors appear. The **agent** is an AI coding agent that edits files inside a session. The
**operator** is a person on call. `../SKILL.md`, Section 10 holds the glossary.

**A command is not a plan.** Read `01-the-reversibility-contract.md` before the change. Read
this file when you need the exact text.

---

## 2. The command surface

| Goal | Signal | Command | Safety rule |
|---|---|---|---|
| Mark a safe point | The agent is about to edit 2 or more files | `git commit -m "checkpoint: <name>"` | A stash is not a checkpoint. R-13 |
| Name a checkpoint | The session will last more than one turn | `git tag ckpt/<name>` | A tag survives a rebase of the branch |
| List the checkpoints | The operator asks what changed last | `git tag -l 'ckpt/*'` and `git log --oneline` | A configuration change is a push. SRE Ch. 6, p. 83 |
| Undo one published commit | The commit is on a branch that others use | `git revert <sha>` | It adds a commit. Every other clone stays valid. R-11 |
| Undo one file | One file is wrong and the rest is correct | `git restore --source=<ref> -- <path>` | It touches one path. It keeps the other work |
| Undo the last agent turn | The branch is local and unpublished | `git reset --hard <checkpoint>` | One actor holds the branch. Never a shared branch. R-11 |
| Recover a lost commit | A reset removed work that nobody published | `git reflog`, then `git reset --hard <sha>` | The reflog is local. It expires. It is the last resort |
| Isolate an agent | An agent and a person work at the same time | `git worktree add ../agent-<task> -b agent/<task>` | One actor writes per tree. SRE Ch. 14, p. 202–203. R-40 |
| Find the bad commit | The fault started between two known points | `git bisect start`, then `git bisect run <script>` | The test must give the same answer every time. SRE Ch. 17, p. 236–237. R-43 |
| List a release | The operator needs the candidate set | `git log --oneline <prev-tag>..<new-tag>` | Archive the report with the artifacts. SRE Ch. 8, p. 123 and p. 126. R-10 |
| Reverse a deployed artifact | The fault is in production | Move the release label to the previous version | Move a label. Never change the content behind a name. SRE Ch. 8, p. 124–125. R-08 |

---

## 3. Checkpoint before a risky edit

### 3.1 Take the checkpoint

**Signal.** The agent will edit 2 or more files, run a migration, or change a configuration
file that production reads.

```sh
git add -A
git commit -m "checkpoint: before <the risky edit>"
```

**Safety rule.** Record the change in a systematic and documented way. Then you can return
the system to the state before the test, instead of an unknown mixed configuration. SRE
Ch. 12, p. 180. R-14, agent edits with no checkpoint. The checkpoint costs seconds. The
recovery without it costs the whole session.

### 3.2 A stash is not a checkpoint

**Signal.** The agent typed `git stash` before a risky edit. A stash fails as a checkpoint
for four reasons.

1. A stash entry has no stable name. `stash@{0}` moves when the agent stashes a second time.
2. A stash does not appear in `git log`, so the operator cannot see it beside the symptom.
3. `git stash drop` and `git stash clear` remove entries with no confirmation.
4. A conflict during `git stash pop` can change the tree and keep the entry.

**Fix.** Commit a named checkpoint on a work branch instead. R-13.

### 3.3 Name the checkpoint

**Signal.** The session will last more than one turn. A later turn must find this point.

```sh
git tag ckpt/<task>-<step>
git tag -a ckpt/<task>-<step> -m "state before <the risky edit>"
```

**Safety rule.** A tag points at a commit and survives a rebase of the branch. Keep the tag
local. Push it only when the team agreed to that. Use one prefix for every checkpoint tag.

### 3.4 List the checkpoints

**Signal.** The operator asks which change touched the system last.

```sh
git tag -l 'ckpt/*'
git log --oneline --decorate -n 20
git diff --stat ckpt/<name>
```

**Safety rule.** A change to configuration counts as a change. SRE Ch. 6, p. 83 defines a
push as a change to running software or to its configuration. List the configuration commits
with the code commits. R-41, no change annotation on the metric timeline. The list answers
the first diagnostic question. SRE Ch. 12, p. 177–178.

---

## 4. Revert

### 4.1 Revert one published commit

**Signal.** The commit is on a branch that other people or other agents fetch.

```sh
git revert <sha>
```

**Safety rule.** The revert adds a new commit. It rewrites no history. Every other clone
stays valid. The new commit is the audit trail of who reversed what. The revert is itself
reversible. Run `git revert <the revert sha>` to restore the change.

### 4.2 Revert a merge commit, or a whole release

**Signal.** The change reached the branch as a merge. Or the release holds several bad
commits, and no single commit holds the fault.

```sh
git revert -m 1 <merge-sha>
git revert --no-commit <last-good-sha>..<head-sha>
git commit -m "revert: <release name>"
```

**Safety rule.** `-m 1` names the parent to keep, which is the branch you stayed on. Confirm
the parent order with `git log --oneline --graph` first. Revert the whole release, then search
for the cause. Revert first and diagnose afterward. SRE App. B, p. 572. A later merge of the
same branch does not restore the reverted content.

### 4.3 Revert one file to a checkpoint

**Signal.** One file is wrong. The other files in the turn are correct.

```sh
git diff ckpt/<name> -- <path>
git restore --source=ckpt/<name> -- <path>
```

**Safety rule.** Read the difference before you restore. The command touches one path and
keeps every other edit in the working tree. Use `git checkout ckpt/<name> -- <path>` when
`git restore` is absent. The command changes the file on disk only. Commit the restored file,
or the next checkpoint hides the reversal.

### 4.4 Revert the last agent turn

**Signal.** The branch is local. No other clone holds it.

```sh
git status -sb
git log --oneline origin/<branch>..HEAD
git reset --hard ckpt/<name>
```

**Safety rule.** Run the first two commands first. They prove that the commits are
unpublished. `git reset --hard` on a shared branch is R-11 and it destroys other clones.
`git reset --keep ckpt/<name>` is the softer form. It stops when the reset would discard an
uncommitted edit.

### 4.5 Remove the files that the turn created

**Signal.** The reset finished, and untracked files remain on disk.

```sh
git clean -nd
git clean -fd
```

**Safety rule.** Run `git clean -nd` first. `-n` only lists. `git clean -fd` deletes with no
undo, and git holds no copy of an untracked file. `-x` also deletes ignored files, which can
include a local environment file. Name the path instead of using `-x`.

---

## 5. Why revert beats reset on a shared branch

| Property | `git revert` | `git reset --hard` and a force push |
|---|---|---|
| History | Adds a commit | Removes commits |
| Other clones | Stay valid | Diverge on the next fetch |
| Audit trail | The revert commit names the actor | None |
| Reversible | Yes. Revert the revert. | No. The commits are unreachable. |
| Safe with 2 or more actors | Yes | No |

**A reversal must leave a record.** Google replaced free access with an authenticated service.
No engineer could then modify a server without an audit trail. SRE Ch. 7, p. 113. The
revert commit is that record for source code. A rewritten branch also loses the change list
that the release report needs. SRE Ch. 8, p. 123 and p. 126. R-10. The one case for a reset
is a local branch that one actor holds and no clone fetched. Prove all three conditions
first.

---

## 6. The reflog is the last resort

**Signal.** A reset or a rebase removed a commit, and no branch or tag points at it.

```sh
git reflog
git reset --hard <sha>
git fsck --lost-found
```

**Safety rule.** The reflog is local to one clone. It does not travel with a push or a fetch.
The default expiry is 90 days for a reachable entry and 30 days for an unreachable entry.
Read the local value with `git config gc.reflogExpire`. **Modern**.

**Treat the reflog as recovery, not as a plan.** A path that runs rarely is fragile, because
the feedback cycle is long. SRE Ch. 7, p. 103. R-04. Section 3 is the path you rehearse.

---

## 7. Worktree isolation for an agent

**Signal.** An agent edits the repository while a person also edits it.

```sh
git worktree add ../agent-<task> -b agent/<task>
git worktree list
git worktree remove ../agent-<task>
```

**Safety rule.** One actor writes to one tree. During an incident, one actor writes to
production, and every other participant proposes a change to that actor. SRE Ch. 14,
p. 201–203. R-40, two actors writing to production at once.

**What the worktree gives you.** The agent holds its own working directory, its own branch,
and its own index. A reset inside the agent tree cannot touch the operator's edits. The two
trees share one object store, so a checkpoint in one tree is visible from the other. The
worktree isolates no database and no deployed artifact. Read `07-automation-safety.md`.

---

## 8. The commands that reverse a release, not a file

A git revert reverses source code. It does not reverse a running service.

### 8.1 Move the release label

**Signal.** The bad change is deployed, and the prior artifact exists. The deploy system
moves the release label from the new package version to the previous version. The package
content never changes.

**Safety rule.** Name packages, version them with a unique hash, and sign them. Applying an
existing label to a package moves the label. SRE Ch. 8, p. 124–125. R-08, a mutable artifact
tag. A deploy that names `latest` resolves to unknown content, so the label move has no
target.

### 8.2 Record the build identity and the change list

**Signal.** The operator cannot map a process to a commit, or cannot list a release.

```sh
git describe --tags --always --dirty
git log --oneline <prev-tag>..<new-tag>
```

**Safety rule.** Every binary reports its build date, its revision number, and its build
identifier. R-09. The release system produces a report of all changes in a release, and
archives it with the build artifacts. SRE Ch. 8, p. 123 and p. 126.

### 8.3 Branch and cherry pick

**Signal.** A fix must reach the release without the other mainline changes.

```sh
git switch -c release/<version> <sha>
git cherry-pick <fix-sha>
```

**Safety rule.** Branch from the mainline at a specific revision. Never merge the branch back
into the mainline. Submit the fix to the mainline first, then cherry pick it into the branch.
SRE Ch. 8, p. 123–124. R-12, a release built from a moving head. Rebuild an old revision with
the toolchain version that the revision pins. SRE Ch. 8, p. 122. R-06 and R-07.

---

## 9. Search for the bad commit

**Signal.** The fault started between a known good point and a known bad point.

```sh
git bisect start
git bisect bad <bad-ref>
git bisect good <good-ref>
git bisect run ./scripts/reproduce.sh
git bisect reset
```

**Safety rule.** The script must give the same answer on every run. A flaky signal gives a
wrong answer, and an automated reversal on a flaky signal reverts good changes. SRE Ch. 17,
p. 236–237. R-43. The script exits 0 for a good commit, 1 for a bad commit, and 125 when it
cannot test the commit. **Modern**. Read `05-locating-the-bad-change.md` for the method.

---

## 10. Who may run which command

| Command | Agent, alone | Operator, alone | Needs a named approver |
|---|---|---|---|
| `git commit -m "checkpoint: <name>"` | Yes | Yes | No |
| `git tag ckpt/<name>` | Yes | Yes | No |
| `git revert <sha>` on its own branch | Yes | Yes | No |
| `git reset --hard` on an unpublished branch | Yes | Yes | No |
| `git clean -fd` | No | Yes | No |
| A push to a shared branch | No | Yes | No |
| A force push to any shared branch | No | No | Yes |
| A migration that drops a column, or a data restore | No | No | Yes |
| A release label move | No | Yes | No |

**Rule.** An automated actor reports a data defect. It does not repair the defect by deleting
data. SRE Ch. 17, p. 242. R-38. Rate limit every automated reversal. SRE Ch. 7, p. 118. Give it
a stop control. SRE Ch. 13, p. 195. R-35.

---

## 11. War stories

**Diskerase, SRE Ch. 7, p. 117–118.** Google decommissions racks with automation that
overwrites the full disk content of every machine in the rack. One decommission failed after
the erase step succeeded. An engineer restarted the workflow from the beginning to debug it.
On that run the set of machines that still needed the erase was correctly empty. The code
treated the empty set as a special value that meant everything. The automation wiped the disks of almost
every machine in every content delivery site within minutes. Recovery took the better part of
two days, then weeks of audit. The fixes were sanity checks, rate limiting, and an idempotent
workflow. R-34 and R-36.

**The Bigtable disk that looked free, SRE Ch. 7, p. 107.** One multi-petabyte Bigtable cluster
skipped the first disk on each 12-disk machine, for latency reasons. A year later, unrelated
automation read an unused first disk as a sign that the machine held no storage and was safe
to erase. All of the data was wiped at once, and real-time replicas saved the service. R-37.
An agent that finds no lock file and no record must stop and ask a person.

**The unknown mixed configuration, SRE Ch. 12, p. 180.** The troubleshooting chapter tells the
engineer to record which ideas they had, which tests they ran, and what each test returned.
The stated reason is recovery. An engineer who changed the system in a systematic and
documented way can return it to the state before the test. An engineer who did not is left
with a configuration that nobody can describe. An agent that edits files while it diagnoses
is in that position. The named checkpoint is its record.

**Freelancing during an incident, SRE Ch. 14, p. 201–203.** The unmanaged incident opens the
incident management chapter. One engineer changed a setting on a few machines and told nobody.
He believed the change would fix the problem. The change was wrong, and it made a bad situation
far worse. The chapter answers with one write holder for the incident. `incident-response` owns
the full account. This is the rule behind Section 7 and Section 10.

---

## 12. Session protocol for an agent

Run these steps in order. **Modern**.

1. Read `git status -sb` first. Report an unclean tree to the operator.
2. Commit a checkpoint. Tag it `ckpt/<task>-0`.
3. Make the edit, then run the test.
4. Commit the result with a message that names the intent.
5. Repeat step 2 to step 4 for each risky edit.
6. Revert with `git restore` when one file is wrong.
7. Revert with `git reset --hard ckpt/<task>-<n>` when the whole turn is wrong.
8. Stop and ask a person before any push to a shared branch.
9. Write the reverse path and the measured time into the Reversal Record.

**Report rule.** A revert is a reportable event. It triggers a postmortem. SRE Ch. 15,
p. 209. Record the cause, the action, and the result. Record it also when the revert did not
fix the symptom. R-44.

---

## 13. Proportionality

| The change is | Required from this file |
|---|---|
| One file, no persisted state, no deploy | Nothing |
| Two or more files in one agent turn | Section 3, the checkpoint |
| A commit on a shared branch | Section 4 and Section 5 |
| A configuration file that production reads | Section 3 and Section 8 |
| A deployed artifact | Section 8 |
| A migration, a backfill, or a delete path | Also `06-data-and-schema-reversibility.md` |
| An automated actor that acts in bulk | Also `07-automation-safety.md` |

"Not applicable" is a valid result.

---

## 14. Cross-references

| You need | Read |
|---|---|
| The four questions before a change | `01-the-reversibility-contract.md` |
| Hermetic builds, labels, release branches | `02-release-engineering.md` |
| The exposure ladder and the canary | `03-progressive-rollout-and-canary.md` |
| The order of moves during an outage | `04-change-induced-emergency.md` |
| The search for the bad change | `05-locating-the-bad-change.md` |
| Backups, soft deletion, restore proof | `06-data-and-schema-reversibility.md` |
| Rate limits, idempotency, stop controls | `07-automation-safety.md` |
| The pre-merge gate | `09-launch-coordination.md` |
| The hazard codes named above | `rollback-hazard-catalog.md` |
| The glossary, the gates, the Reversal Record | `../SKILL.md` |
| A durable local file replacement | The `systems-programming` skill |
