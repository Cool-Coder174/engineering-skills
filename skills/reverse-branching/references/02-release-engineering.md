# Release Engineering: What Makes A Revert Possible

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 8 "Release Engineering", p. 120–130. Ch. 9 "Simplicity", p. 131–135. Supporting
citations from SRE Ch. 12, p. 177–178, SRE Ch. 13, p. 190–194, and SRE Ch. 27, p. 459–462.
*Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007), Sec. 14.2, p. 243–246.

---

A revert is not one command. A revert is a set of properties that the release system holds
before the incident starts. You must name the prior state. You must rebuild it. You must
install it. You must list what changed since.

**Release engineering decides whether a revert is possible. The incident does not.** SRE
Ch. 8, p. 120 states the rule. Reliable services require reliable release processes. The same
page names the antipattern. A release that nobody can reproduce is a "unique snowflake".

Read this file in three cases. You cannot rebuild a prior artifact. You cannot say which
version runs in production. You cannot list the changes in the last release. Each section
states one release property and the signal that tells you the property is missing.

---

## 1. The precondition chain

Each row names one revert step and the property that supplies it. A missing property removes
the step from your options.

| Revert step | Release property that supplies it | Source |
|---|---|---|
| Name the prior state | A unique version and a signed package | SRE Ch. 8, p. 124 |
| Rebuild the prior state | A hermetic build and a pinned toolchain | SRE Ch. 8, p. 122 |
| Identify what now runs | A build flag on every binary | SRE Ch. 8, p. 123 |
| List what changed | A change report per release | SRE Ch. 8, p. 123 and p. 126 |
| Select one change to remove | A release branch and a cherry pick | SRE Ch. 8, p. 123–124 |
| Install the prior state | A label move over an immutable package | SRE Ch. 8, p. 124–125 |
| Revert a setting alone | A separate configuration package | SRE Ch. 8, p. 128 |
| Limit the damage first | A rollout that advances in stages | SRE Ch. 8, p. 127 |
| Prove who may act | Gated operations and access control | SRE Ch. 8, p. 122–123 and p. 125 |

**A gap in this table is a gap in your rollback.** Repair the release system before you write
a rollback step that depends on the missing property.

---

## 2. The four philosophies of release engineering

SRE Ch. 8, p. 121 names four principles. The chapter presents them as a delivery philosophy.
Read them here as reversibility controls. Section 3 covers hermetic builds in full. Section 9
covers the enforcement of policies in full.

| Principle | Page | What it gives a revert |
|---|---|---|
| Self-Service Model | 121 | The team that owns the change also owns the reverse action |
| High Velocity | 121–122 | Fewer changes per release, so the search space stays small |
| Hermetic Builds | 122 | The prior artifact can be rebuilt at any later date |
| Enforcement of Policies and Procedures | 122–123 | A named actor, a review, and an audit trail for each step |

### 2.1 Self-Service Model

**Release engineering supplies the tools. The product team runs its own release.** SRE Ch. 8,
p. 121. Teams decide how often to release. Releases are automatic. An engineer joins only
when a problem appears. The consequence for a revert is ownership. The team that pushed the
change holds the reverse action.

**Signal that the property is missing.** The rollback step in your runbook names a team that
did not write the change.

### 2.2 High Velocity

**Frequent releases produce fewer changes between versions.** SRE Ch. 8, p. 121. The chapter
states the payoff. Testing and troubleshooting both become easier. Some teams build every
hour and then select which build to deploy, on the test results and the features in that
build. Other teams use the "Push on Green" model and deploy every build that passes all
tests. SRE Ch. 8, p. 122. The consequence for a revert is search cost. A release of one change
has one revert candidate.

**Signal that the property is missing.** Your last release contained more than one topic, and
nobody can say which change caused the symptom.

---

## 3. Hermetic builds and build reproducibility

**A build is hermetic when it does not depend on the build machine.** SRE Ch. 8, p. 122. The
build depends on known versions of the build tools, such as compilers, and on known versions
of the dependencies, such as libraries. The build process is self-contained. It must not call
a service outside the build environment.

**The test for hermeticity.** Two people build the same product at the same revision number
on different machines. The results must be identical. SRE Ch. 8, p. 122.

**A non-hermetic build removes the revert option.** You cannot return to an artifact that you
cannot reconstruct. Table 3 in `../SKILL.md` sends this case to the roll-forward option.

**The toolchain is part of the revision.** Version the build tools by the source revision of
the project. SRE Ch. 8, p. 122. A project built last month must not use this month's compiler
during a cherry pick, because the newer compiler can hold incompatible features.

**Signal that the property is missing.** A pipeline file names an image tag or a compiler
version of `latest`. A build step reads a dependency with no pinned version, or calls a
network service.

**War story — rebuilding an older release.** SRE Ch. 8, p. 122 describes how Google fixes a
bug in software that already runs in production. The team rebuilds at the same revision as
the original build. The team then includes the specific changes that arrived after that
point. The chapter names this tactic cherry picking. The build uses the compiler version that
the project revision pins, not the current one. The story proves that a patch of an old
release needs a versioned toolchain, and not only versioned source.

Catalog entries: R-06, R-07 in `rollback-hazard-catalog.md`.

---

## 4. Build identity and packaging

You cannot revert what you cannot name. SRE Ch. 8 describes four identity mechanisms.

| Mechanism | Rule | Page |
|---|---|---|
| Build flag on the binary | Every binary reports build date, revision number, and build identifier | 123 |
| Package name | Packages carry a name, such as a path in a namespace | 124 |
| Package version | Each version carries a unique hash, and a signature proves authenticity | 124 |
| Package label | A label marks the position in the release process, such as `dev`, `canary`, or `production` | 124–125 |

**A label moves. Package content never changes.** SRE Ch. 8, p. 124–125. Apply an existing
label to a new package, and the system moves that label off the old package. A person who
installs the labelled version always receives the newest package with that label. This is the
cheapest reverse action in the book. A revert becomes a label move onto the previous package
version, with no rebuild and no new artifact.

**Signal that the property is missing.** A deploy names a mutable tag. A package name is
reused for new content. A running process cannot report its own revision.

**War story — the label move.** SRE Ch. 8, p. 124–125 gives the mechanism for release
promotion. A package version gains the `canary` label. The label leaves the previous package
at the same moment. Promotion and demotion use the same operation, in opposite directions.
The story proves that release stage belongs to a pointer, and never to the artifact itself.

Catalog entries: R-08, R-09 in `rollback-hazard-catalog.md`.

---

## 5. The change report

**The release system produces a report of all changes in a release.** SRE Ch. 8, p. 123. The
system archives that report with the other build artifacts. The chapter states the purpose.
The report lets an SRE understand what a release contains, and it shortens troubleshooting
when a release causes a problem. The system also logs the result of every release step, and
creates a report of all changes since the last release. SRE Ch. 8, p. 126.

**Log every version deployment and every configuration change, at all layers of the stack.**
SRE Ch. 12, p. 177. The layers run from the server binaries that serve user traffic down to
the packages on each node. Annotate the error graph with the start time and the end time of
each deploy. SRE Ch. 12, p. 178.

**Signal that the property is missing.** A responder asks which change arrived last, and no
system holds the answer. Catalog entries: R-10, R-41 in `rollback-hazard-catalog.md`. Read
`05-locating-the-bad-change.md` for the search that consumes this data.

---

## 6. Branch per release from head, and cherry picks

SRE Ch. 8, p. 123–124 gives the branch model. Follow these steps in order.

1. Commit all code to the mainline, which is the main branch of the source tree.
2. Branch from the mainline at a specific revision.
3. Never merge changes from that branch back into the mainline.
4. Commit each bug fix to the mainline first.
5. Cherry pick the fix from the mainline onto the release branch.
6. Approve or reject each cherry pick request one at a time. SRE Ch. 8, p. 126.
7. Run the unit tests again on the release branch, and create an audit trail of the passes.

**Step 3 is the load-bearing rule.** It stops the release from acquiring unrelated mainline
changes that arrived after the original build. SRE Ch. 8, p. 124 states the result. The team
knows the exact contents of each release.

**Step 7 exists because a release branch can hold code that the mainline does not hold.** A
cherry pick creates that state. The tests must pass in the context of what you actually
release. SRE Ch. 8, p. 124.

Two more rules from SRE Ch. 8, p. 124 protect the branch point. Make the continuous build
test targets the same targets that gate the release. Create each release at the revision of
the last continuous build that passed all tests.

| Branch model | Revert property | Cost |
|---|---|---|
| Build from the mainline head at deploy time | You cannot remove one change without removing the others | The release contents are unknown |
| Branch at a revision, cherry pick forward | Each included change is named and approved alone | The team maintains the branch |

SRE Ch. 27, p. 459–460 repeats the same shape as a launch requirement. Store all code and all
configuration files in version control. Cut each release on a new release branch.

**Signal that the property is missing.** A deploy job builds from the mainline head. No
release branch exists. No revision is pinned. Catalog entry: R-12.

---

## 7. The build and deployment system

SRE Ch. 8, p. 123–127 names the parts. Read them as the surface that a reverse action drives.

| Component | Role | Page |
|---|---|---|
| Blaze, released as Bazel | Builds a target from its declared dependencies | 123 |
| Rapid | Runs the release, and holds blueprints, access control, and workflows | 123, 125 |
| Midas Package Manager | Assembles, names, versions, and signs the package | 124 |
| Sisyphus | Runs the rollout as a set of tasks, and reports its progress | 126 |

**The typical release process has four steps.** SRE Ch. 8, p. 126.

1. Rapid uses the requested integration revision number to create a release branch.
2. Rapid compiles the binaries and runs the unit tests in dedicated environments.
3. The artifacts become available for system tests and for a canary deployment. A canary
   deployment starts a few jobs in production after the system tests finish.
4. The system logs each step, and creates a report of all changes since the last release.

**The rollout consumes a build label, not a rebuild.** SRE Ch. 8, p. 126. Rapid creates the
rollout in a long-running Sisyphus job and passes the build label. Sisyphus uses that label
to select which package version to deploy. A revert is therefore a new rollout at the
previous label, with the same control surface and the same dashboard.

**Fit the deployment process to the risk profile of the service.** SRE Ch. 8, p. 127 gives
three shapes.

| Service class | Rollout shape | Reverse property |
|---|---|---|
| Development or pre-production | Build hourly, push automatically when all tests pass | The exposure is zero, so no reverse action is needed |
| Large user-facing service | Start in one cluster, then expand exponentially | Each stage limits the population that a revert must repair |
| Sensitive infrastructure | Extend over several days, across geographic regions | A fault appears in one region before it reaches the rest |

Read `03-progressive-rollout-and-canary.md` for the exposure ladder and the bake period.

---

## 8. Configuration management and its trade-offs

**A configuration change is a change.** SRE Ch. 8, p. 127 states the risk in one clause.
Configuration changes are a potential source of instability. One rule applies to every model
below. Store the configuration in the primary source code repository, and enforce a strict
code review requirement. SRE Ch. 8, p. 127 names four distribution models. Select one per
project.

| Model | What it does | Reverse action | Cost |
|---|---|---|---|
| Use the mainline for configuration | Edit at the head of the main branch, review, then apply | Commit the previous content, then update the jobs | Skew between the stored file and the running file, because jobs must update to receive the change |
| One package for binary and configuration | Ship both artifacts together | Install the previous package version | The binary and the configuration bind tightly, so a setting fix needs a new binary |
| A separate configuration package | Apply the hermetic principle to configuration | Install the previous configuration package alone | Two packages to build, and a shared label to join them |
| An external store | Hold settings that change while the binary runs | Write the previous value into the store | The change sits outside the commit history, so a code revert does not reverse it |

**The third model gives the best reverse properties.** SRE Ch. 8, p. 128. The build system
snapshots the configuration beside the binary. The build identifier reconstructs the
configuration at a point in time. A grouping label marks which package versions install
together.

**The fourth model sits outside the commit-revert model.** SRE Ch. 8, p. 128 names the stores
for settings that change often, or that change while the binary runs. An operator must know
which settings live there, because a revert of the repository does not touch them.

**War story — `first_folio` and `bad_quarto`.** SRE Ch. 8, p. 128 describes a feature that
shipped with a flag value of `first_folio`. The team then found that the value should be
`bad_quarto`. The team had built the binary and the configuration as two packages. The fix
was therefore a cherry pick of the configuration change onto the release branch, a rebuild of
the configuration package, and a deploy. The story proves the rule. A separated configuration
package makes a setting revert cost no new binary build.

### The four constraints on configuration in version control

*Release It!* Sec. 14.2, p. 245 adds four constraints. They apply to any of the models above.

1. Keep the repository secure, because these files hold database passwords.
2. Join version control to a change control process, so a reader can see why a change was
   made and not only that a change happened.
3. Deploy authorized changes automatically, directly from the repository.
4. Run an automated audit that reports the difference between the repository and the running
   system.

Constraint 4 exists for one named reason. People edit files on the host during an incident to
restore service, and the repository never receives the edit. *Release It!* Sec. 14.2, p. 245.
The audit must find the difference. The audit must not overwrite it. Sec. 14.2, p. 246 adds
the fleet check. Verify at intervals that the machines in a horizontally scaled layer are
still synchronized. The rule is "Trust, but verify."

Catalog entries: R-22, R-23, R-24, R-25 in `rollback-hazard-catalog.md`.

---

## 9. Gated operations

**Several layers of access control decide who may perform each release operation.** SRE
Ch. 8, p. 122–123. The chapter lists six gated operations.

1. Approving a source code change.
2. Specifying the actions that the release process performs.
3. Creating a new release.
4. Approving the initial integration proposal and each later cherry pick.
5. Deploying a new release.
6. Changing the build configuration of a project.

Two more rules complete the control. Almost all changes to the codebase require a code
review. SRE Ch. 8, p. 123. Role-based access control lists decide who may perform each action
on a release project. SRE Ch. 8, p. 125.

**A reverse action crosses at least two of these gates.** It creates a release, and it deploys
a release. An automated revert that misses the gates is an unreviewed production change. A
machine-written revert commit is still a change, and it needs the same review. Read
`07-automation-safety.md` for the limits on an automated actor. Read `../SKILL.md`, Table 9,
for the actor that may act without a person.

---

## 10. Two rules from Ch. 9 that protect a revert

### Delete dead code

**Do not comment out code. Do not gate dead code behind a flag that stays disabled. Delete
it.** SRE Ch. 9, p. 132–133. The chapter names the justification. Source control systems make
it easy to reverse a change.

**War story — Knight Capital.** SRE Ch. 9, p. 133 cites the Knight Capital regulatory order
to show what dead code does. Code that never runs, held alive behind a flag that stays
disabled, is a time bomb. A later deployment or a later configuration state reactivates the
dormant path. The story proves that a disabled flag is not a revert. Only deletion, backed by
the source history, removes the path. Catalog entry: R-21 in `rollback-hazard-catalog.md`.

### Release in small batches

**A batch of 100 unrelated changes makes attribution expensive.** SRE Ch. 9, p. 135. The
chapter states that measuring the effect of a single change is much easier than measuring the
effect of a batch. It offers the gradient descent analogy. Take a small step, then evaluate
whether the step improved or degraded the system. Batch size is the same lever as High
Velocity in Section 2.2, and it sets the search cost in `05-locating-the-bad-change.md`.

---

## 11. What a rollback still needs from outside the release system

Two properties sit outside Ch. 8. Record both in the Reversal Record.

**A rollback procedure that no test exercised does not exist.** SRE Ch. 13, p. 190–191.

**War story — the untested permissions rollback.** SRE Ch. 13, p. 189–191 describes a planned
test that blocked all access to one database out of a hundred. Within minutes many dependent
services lost access for internal and external users. The team stopped the test and tried to
reverse the permissions change. The reversal failed, because nobody had ever tested that
procedure in a test environment. The team restored access by a different route that had been
tested, and full access returned within an hour. The chapter states the rule that followed.
Test rollback procedures thoroughly before a large-scale test.

**The revert path must not run through the failing system.** SRE Ch. 13, p. 193.

**War story — the global crash-loop.** SRE Ch. 13, p. 191–194 describes a configuration change
to abuse-protection infrastructure that was pushed globally. The fleet began to crash-loop
almost at once, and internal applications failed with it. The push engineer pushed a second
configuration change to reverse the first within five minutes, and services recovered. The
chapter names what helped. Command-line tools and other access methods allowed updates and
reversals while the normal interfaces were unreachable. The chapter also names what was
missing. The earlier canary had not exercised a rare configuration keyword together with the
new feature. The team had judged the change non-risky, so it took a less strict canary path.

Two rules follow. Keep command-line tools and other access methods that work when the normal
interface is down, and test them at regular intervals. SRE Ch. 13, p. 193. Apply the same
canary to a change that looks safe. SRE Ch. 13, p. 194. Catalog entries: R-02, R-05, R-17 in
`rollback-hazard-catalog.md`. Read `04-change-induced-emergency.md` for the order of moves
during the incident itself.

---

## 12. Review checklist

Run this table against the pipeline files, the build files, and the deploy job. Report only
the rows that fail. Catalog codes point into `rollback-hazard-catalog.md`.

| Question | Evidence to find | Catalog code |
|---|---|---|
| Do two builds of one revision match? | A pinned toolchain and pinned dependencies | R-06, R-07 |
| Can a person name the running version? | A build flag, and the build identifier in a log line | R-09 |
| Does the artifact name resolve to fixed content? | A unique hash per version, and a signature | R-08 |
| Does a release carry its change list? | An archived report beside the build artifacts | R-10 |
| Is the release cut at a pinned revision? | A release branch, and no merge back to the mainline | R-12 |
| Does a setting revert alone? | A separate configuration package, or a versioned file | R-22 |
| Does the repository match the running system? | An audit job that reports the difference | R-24 |
| Does the rollout advance in stages? | Named stages with a fraction and a region order | R-16 |
| Did anyone test the rollback? | A rehearsal date and a measured time | R-02 |
| Does a reverse path exist when the interface is down? | A command-line path that does not use the failing system | R-05 |

---

## 13. When this file does not apply

This file describes the release system. It does not apply to a change that never reaches a
release. Three examples follow. A local script that no pipeline deploys. A document change
with no build step. A change on a branch that one actor holds before any push. For those
cases, report "Not applicable". That result is better than an invented finding.

For the reversibility class of the change, read `01-the-reversibility-contract.md`. For data
and schema, read `06-data-and-schema-reversibility.md`. For the exact command surface, read
`08-agent-and-operator-shortcuts.md`.
