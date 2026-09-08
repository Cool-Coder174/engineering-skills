# Launch Coordination and the Reversal Checklist

**Sources:** *Site Reliability Engineering* — Beyer, Jones, Petoff, Murphy (O'Reilly, 2016).
Ch. 27, "Reliable Product Launches at Scale", p. 448–470. Appendix E, "Launch Coordination
Checklist", p. 585–587. Staged rollout rules from Appendix B, p. 572.

A launch is the point where a team proves the reverse path. Before the launch the reverse
path is a claim. After the launch it is a measured fact. Read this file when you build the
gate that runs before a merge. Read `03-progressive-rollout-and-canary.md` for the exposure
ladder itself, and `01-the-reversibility-contract.md` for the four questions that the gate
asks.

---

## 1. What counts as a launch

**Google defines a launch as any new code that introduces an externally visible change to an
application.** Ch. 27, p. 449.

That definition triggers the process, not the word "release". Under it Google performs up to
70 launches per week. p. 449.

Two consequences follow for the reversal gate. A change with no externally visible effect
does not need the gate, and Section 10 sets that limit. A configuration change with an
externally visible effect does need it. A push is any change to running software or to its
configuration. Ch. 6, p. 83. Configuration is a potential source of instability. Ch. 8,
p. 127.

**A high launch rate is the reason a launch process can exist at all.** A company that
launches once every three years cannot build one. Most of the process is out of date by the
next launch, and the company never gains enough experience. p. 449.

### War story — the switches built for Santa

Keyhole serves satellite imagery for Google Maps and Google Earth, normally several thousand
images per second. On Christmas Eve 2011 it received 25 times its normal peak, above one
million requests per second. A Christmas site hosted with NORAD let users watch Santa deliver
presents over that imagery. The launch had a deadline that could not move, heavy
publicity, an audience of millions, and a very steep traffic ramp. SRE prepared the
infrastructure and built kill switches into the experience. The team named them
"Make-children-cry switches". Ch. 27, p. 448–449. The lesson for the gate is direct. When the
deadline cannot move, the reverse action must exist before launch day, and it must be a
switch rather than a deploy.

---

## 2. Launch Coordination Engineering

Google created a dedicated consulting team inside SRE for the technical side of a launch.
Software engineers and systems engineers staff it. Ch. 27, p. 449–450. LCE performs five
functions. It audits a product against the reliability standards and gives specific
improvement actions. It acts as the liaison between the teams in a launch. It drives the
technical tasks so that they keep momentum. It acts as a gatekeeper and approves a launch
that it judges safe. It teaches developers the practices and the integration paths. p. 450.

A dedicated team gives three advantages. p. 451.

| Advantage | What it gives the reverse path |
|---|---|
| Breadth of experience | The team saw the same reverse path fail on another product. |
| Cross-functional perspective | One holistic view over a launch that spans many teams and time zones. |
| Objectivity | A nonpartisan advisor mediates between SRE, developers, product managers, and marketing. |

**The reviewer of the reverse path must not be the author of the change.** LCE is an SRE
role, so an LCE has an incentive to prioritize reliability over the other concerns. A company
may choose a different incentive structure. That applies when the company does not share
Google's reliability goals but does share its rapid rate of change. p. 451.

Ch. 33, p. 562 states the same separation for the trading enforcement team. The party that
halts the system is not the party that gains from shipping. Hazard R-19 in
`rollback-hazard-catalog.md` is the version of this rule for a rollout.

---

## 3. The five criteria of a launch process

A good launch process meets five criteria. Ch. 27, p. 451–452.

| Criterion | Meaning | What the reversal gate must do |
|---|---|---|
| Lightweight | Easy on developers | Ask four questions for a small change, not forty. |
| "Robust", the book's word | Catches obvious errors | Block a change with no named reverse action. |
| Thorough | Addresses details consistently and reproducibly | Use the same hazard codes every time. |
| Scalable | Accommodates many simple launches and fewer complex ones | Give a low-risk lane. Section 8. |
| Adaptable | Works for a common launch type and for a new one | Keep the intent of each question. Section 9. |

**The book states that these criteria conflict.** A process cannot be lightweight and
thorough at the same time. Google balances them with three tactics. p. 452.

- **Simplicity.** Get the basics correct. Do not plan for every eventuality.
- **A high touch approach.** An experienced engineer adapts the process to each launch.
- **Fast common paths.** Identify a class of launch that always follows one pattern, and give
  that class a simplified process.

**Signal that the gate is too heavy: engineers avoid it.** Engineers bypass a process
that they judge too burdensome or of low value. The risk rises in crunch mode, when the team
reads the process as one more item that blocks the launch. p. 452–453. A gate that nobody
runs protects nothing.

---

## 4. The two rules that keep a checklist alive

LCE uses a launch checklist for launch qualification. The checklist pairs each question with
a concrete action item and a pointer to more information. One example. The question asks
whether you store persistent data. The action item tells you to implement backups, and it
links the instructions. p. 453.

The number of possible questions about a system is near infinite, so a checklist grows until
nobody can use it. p. 453. LCE holds two curation rules.

1. **Every question's importance must be substantiated, ideally by a previous launch
   disaster.** p. 453.
2. **Every instruction must be concrete, practical, and reasonable for developers to
   accomplish.** p. 454.

At one point Google required approval from a vice president to add a new question. p. 453.
LCEs also make small updates continuously. Once or twice a year one member reviews the whole
checklist, finds the obsolete items, and modernizes sections with the service owners. p. 454.

### War story — the rate limiting section collapsed to one line

Engineers in a large organization do not know which shared infrastructure already exists, so
they reimplement it. The launch checklist became the vehicle that drove convergence. Long
sections about rate limiting requirements were replaced by a single line. The line said to
implement rate limiting with system X. Ch. 27, p. 454. The lesson for the reversal gate is
that a named shared mechanism removes a whole section of questions. When one release tool
owns the label move, the gate stops asking how the team reverts an artifact.

---

## 5. The checklist questions that bear on reversal

Ch. 27 names nine themes, p. 456–461. Appendix E gives the original checklist of 2005 in ten
sections, p. 585–587. The table takes the items that decide whether a change can be undone.

| Checklist item | Source | Reversal reading | Hazard |
|---|---|---|---|
| Check all code and configuration files into the version control system | Ch. 27, p. 460 | Configuration with no prior version has no reverse action | R-22 |
| Cut each release on a new release branch | Ch. 27, p. 460 | A release branch lets you fix one bug without the unrelated mainline changes | R-12 |
| Are you storing persistent data? Implement backups | Ch. 27, p. 453 | The backup is the reverse path that a code revert cannot reach | R-29 |
| Data backup and restore, disaster recovery | App. E, p. 586 | Name the restore layer and the measured restore time | R-29, R-32 |
| Document all manual processes | Ch. 27, p. 459 | Any team member must run the reverse path in an emergency | R-02 |
| Document the process for restoring data from backups | Ch. 27, p. 459 | The restore is a manual process with a named owner | R-29 |
| Minimize single points of failure, which include humans | Ch. 27, p. 459 | One person who knows the reverse path is a single point of failure | R-02 |
| Release process, repeatable builds, canaries under live traffic, staged rollouts | App. E, p. 586 | A repeatable build is the precondition for a return to the prior artifact | R-06, R-16 |
| Methods and change control to update servers, data, and configs | App. E, p. 586 | Data and configuration each need their own change control | R-22, R-30 |
| Monitoring internal state, end-to-end behavior, and the monitoring itself | App. E, p. 586 | A change that removes monitoring blinds the responder | R-45 |
| Do you have any single points of failure in your design? | Ch. 27, p. 458 | A reverse path that runs through the failing system is one of them | R-05 |
| Do any partners depend on your service? Do they need notification? | Ch. 27, p. 460 | A reverse action reaches a partner. Name the notification path. | R-03 |
| Hard deadlines, external events, Mondays or Fridays | App. E, p. 587 | The launch day decides how many people can run the reverse path | R-04 |
| Identify risk in the individual launch steps and implement contingency measures | Ch. 27, p. 461 | This is the reverse path, written before the launch | R-01 |

**Rollout planning holds the strongest statement in the chapter.** Few events in a large
distributed system happen at one instant. A complicated launch enables individual features on
several subsystems, and each configuration change can take hours. A working configuration in
a test instance does not guarantee that the same configuration works in the live instance.
p. 460. The reverse path inherits that shape. It is also ordered, also slow, and also
untested against the live instance until you run it.

**External dependencies carry six named failure kinds.** A vendor outage, a bug, a systematic
error, a security issue, an unexpected scalability limit, and a missed hard deadline. Google
used proxies, transcoding pipelines, and caches to reduce these risks. p. 460.

---

## 6. Gradual and staged rollouts

**Any change represents risk. The risk is larger for a replicated, globally distributed
system.** Ch. 27, p. 461. The chapter cites the old system administration adage, "never
change a running system". Very few launches at Google are of the push-button kind, where one
product reaches the whole world at one time. Almost every update proceeds in stages, with
verification steps between them. p. 461.

The named ladder. Ch. 27, p. 461–462.

1. Install the new server on a few machines in one datacenter.
2. Observe for a defined period.
3. If the signals stay clean, install on all machines in that datacenter.
4. Observe again.
5. Install on all machines globally.

The first stages are the canaries. The name comes from the canaries that miners carried into
a coal mine. A canary server detects a dangerous effect of the new software under real user
traffic. **Canary testing covers configuration, not only binaries.** Google embeds it in the
internal tools that make automated changes, and in the systems that change configuration
files. p. 462.

**The automatic reverse action is the rule, not the exception.** A tool that installs new
software observes the new server for a period. If the change does not pass the validation
period, the tool returns the server to the previous version automatically. p. 462. Appendix
B, p. 572 states the same requirement as a hard rule. A nonemergency rollout proceeds in
stages. Different stages run in different geographies. A monitoring system that you can
demonstrate to be reliable supervises each stage.

| Pattern | Where it applies | Reverse action | Source |
|---|---|---|---|
| Staged server rollout | Server software in datacenters | Stop the stage. Return to the previous version. | p. 461–462 |
| Canary with a validation period | Any automated change, including configuration | The tool returns to the previous version by itself | p. 462 |
| Gradual client rollout | A mobile application | Stop the percentage increase. Offer the prior version. | p. 462 |
| Invite system, rate-limited signups | A new user-facing service | Reduce the daily signup limit to zero | p. 462 |
| Feature flag | A code path inside a running binary | Disable the flag. Section 7. | p. 462–464 |

**A gradual client rollout protects the backend, not only the client.** Google offers an
updated Android version to a subset of installs and raises the percentage over time. Google
therefore observes the effect on the backend servers and finds a problem early. p. 462.

---

## 7. Feature flag frameworks

A feature flag framework is a mechanism that releases a change slowly and lets you observe
total system behavior under a real workload. The book states where the mechanism earns its
cost. It is useful when a realistic test environment is impractical, and for a complex launch
whose effects are hard to predict. The investment returns value in reliability, in
engineering velocity, and in time to market. p. 462. Keep that hedge. The chapter does not
claim that every change needs a flag.

**A Google feature flag framework meets six requirements.** p. 463.

| Requirement | What it gives the reverse path | Hazard if absent |
|---|---|---|
| Release many changes in parallel, each to a few servers, users, entities, or datacenters | The reverse action touches one change, not the release | R-20 |
| Increase to a larger but limited group of users, usually between 1 and 10 percent | The exposure stays small while the fault is still unknown | R-16 |
| Direct traffic through different servers by user, session, object, or location | You can drain one population instead of the fleet | R-16 |
| Handle failure of the new code paths by design, without affecting users | A defect in the new path does not become an outage | R-17 |
| Revert each change independently and immediately on a serious bug or side effect | This is the reverse action itself | R-20 |
| Measure how much each change improves the user experience | The revert decision carries a metric | R-42 |

Requirement five is the sentence that this skill depends on. **A framework that cannot revert
one change alone and at once is not a feature flag framework.** p. 463.

Google's frameworks have two classes. One class mainly serves user interface improvements.
The other supports arbitrary server-side changes and business logic changes. p. 463.

**The simplest form for a stateless service is an HTTP payload rewriter at the frontend
application servers.** It is limited to a subset of cookies, or to a similar request attribute
or response attribute. The configuration names four things. An identifier for the new code
paths. The scope of the change, such as a cookie hash mod range. A whitelist. A blacklist.
p. 463–464.

**A stateful service scopes the flag to an identity instead.** It uses a logged-in user
identifier, or the identifier of the product entity, such as a document, a spreadsheet, or a
storage object. It proxies or reroutes the request to a different server. p. 464.

**Google hardened the framework so that most of its uses need no LCE involvement.** p. 463.
When the framework guarantees the reverse action, the gate stops asking for one per change.

**Dormant functionality is the client-side form of the same reverse action.** A client
application can hold the code for a new feature before anybody activates it. The
client downloads a configuration file from the server at intervals. The file enables or
disables the feature and sets parameters such as the sync frequency and the retry frequency.
Two results follow. The team avoids a combinatorial explosion of client versions. The team
aborts a launch by disabling the feature, then iterates and releases an updated version.
Ch. 27, p. 465.

**Do not read this pattern as permission to keep dead code.** A flag that a comment describes
as kept for rollback is hazard R-21. Ch. 9, p. 132–133 requires you to delete dead code and
to let source control hold the history.

---

## 8. Convert the checklist into a pre-merge reversibility gate

**Modern.** A pull request, a continuous integration pipeline, and an AI coding agent
postdate both books. The five conversion rules come from the book. The artifacts do not.

1. **Every gate question traces to a hazard, and every hazard traces to a disaster.** The
   curation rule of p. 453. `rollback-hazard-catalog.md` holds the incident for each code.
2. **Every gate action names a command, a label move, or a flag.** The curation rule of
   p. 454. "Improve the rollback story" is not an action item.
3. **Risk-tier the gate.** The low-risk lane below gives the criterion.
4. **Curate on a cadence.** Small updates continuously. One full review each year or twice a
   year, with the service owners. p. 454.
5. **Converge on one named mechanism, then delete the section that the mechanism replaced.**
   p. 454.

### The gate questions

Ask a question only when its signal fires.

| # | Gate question | Signal that raises it | Required artifact | Hazard |
|---|---|---|---|---|
| 1 | What is the reverse action for this change? | Every change with an external effect | The reverse path in the Reversal Record | R-01 |
| 2 | Who runs it, and with what authority? | Every change | A named actor from Table 9 of the SKILL | R-40 |
| 3 | When did somebody last run it, and how long did it take? | The class is schema destructive, data, or external | A rehearsal date and a measured time | R-02, R-29 |
| 4 | Is the prior artifact reproducible? | The change ships a new binary | A hermetic build and a pinned toolchain | R-06, R-07 |
| 5 | Is the configuration in version control and reviewed? | The change edits a setting, a limit, or a route | The file path and the review link | R-22 |
| 6 | Does the release cut a new branch at a fixed revision? | The change enters a release | The release branch name | R-12 |
| 7 | What are the stages, the exposure, and the bake time? | The change reaches production | The rollout table of the Reversal Record | R-16, R-18 |
| 8 | Which independent monitor gates each stage? | The change has more than one stage | The monitor name and the threshold | R-19 |
| 9 | Can the flag revert this change alone and at once? | The change sits behind a flag | The flag identifier and its scope | R-20 |
| 10 | Does the change store persistent data or delete it? | A migration, a backfill, or a delete path | The backup tier, the RPO, and the RTO (**Modern** terms) | R-29, R-30 |
| 11 | Does the change carry an effect that leaves the system? | Mail, money, a partner file, or a disk erase | The compensating action, or "none exists" | R-03 |
| 12 | Does the change alter monitoring or alerting? | The diff touches a dashboard, an alert, or a probe | A separate reverse path for the monitoring | R-45 |
| 13 | Does the reverse path run through the system that fails? | The only control is a console in the same service | A command-line path that works when the console does not | R-05 |
| 14 | Do the manual steps have documentation that any member can follow? | The reverse path has a manual step | The runbook path | R-02 |
| 15 | Which step makes this change irreversible, and what permits it? | The change has a contract step | The bake period and the named approver | R-26, R-27 |

### The low-risk lane

**LCE identified a class of launch that is highly unlikely to cause a mishap, and gave that
class an almost trivial checklist.** The criterion is a launch with no new server executables
and a traffic increase under 10 percent. By 2008 that class covered 30 percent of reviews.
Ch. 27, p. 468. Copy the shape, not the number. The 10 percent figure is Google's threshold
for its own services. Your equivalent threshold is a local decision, and you must write it
down.

### War story — the gate that scaled, and the gate that became a legend

One LCE ran 350 launches through the checklist in three and a half years. A team of about
five engineers therefore handled more than 1,500 launches in the same period. A new LCE needs
about six months of training, because each simple question hides real complexity. The
low-risk lane is what let the team keep that rate. Ch. 27, p. 467–468. The chapter also
records the other side. By 2009 the difficulty of launching a small new service at Google had
become a legend, despite continuous effort to keep the bureaucracy small. p. 468. A gate is
a service. Track its cost, because its latency is a real number.

---

## 9. When the checklist does not fit the change

### War story — the checklist that Android could not use

Before Android, Google had rarely handled mass consumer devices that carry client-side logic
outside its control. Google can fix a Gmail defect within hours, because it pushes new
JavaScript to browsers. That option does not exist on a mobile device. The LCEs on mobile
launches engaged domain experts, decided which sections of the existing checklists applied,
and wrote the new questions. The chapter then states the transferable rule. Keep the intent
of each question, and do not apply a concrete item that does not fit the product. p. 455.

The procedure for a new domain. p. 455.

1. Synthesize the experience of the relevant domain experts.
2. Structure the checklist around broad themes such as reliability, failure modes, and
   processes.
3. Ask the experts which sections of the existing checklist apply and which do not.
4. Return to the first principles of a safe launch.
5. Respecialize those principles into concrete questions that a developer can act on.

**The reversal question survives the respecialization.** The concrete action changes. The
question does not. Ask what undoes this change, who runs it, and how long it takes.

### War story — the logging death spiral

A service logged debugging information when a backend returned an error. That logging cost
more than a normal backend response. As the service became overloaded, it timed out backend
responses inside its own RPC stack and spent still more CPU on logging them. It stopped
completely. It is very hard to predict how a service reacts to overload from first
principles, so a load test is required for most launches. Ch. 27, p. 466. Apply the same
doubt to the reverse path. It also runs while the system is overloaded, and nobody measured
it there.

---

## 10. Proportionality

Rigor scales with the externally visible effect of the change.

| Change | Gate depth |
|---|---|
| No externally visible change. No persisted state. No deploy. | Nothing from this file |
| A change behind an existing hardened flag framework | Questions 1, 2, and 9 |
| A configuration change | Questions 1, 2, 5, and 7 |
| A new binary that reaches production | Questions 1 to 8, and 12 to 14 |
| A migration, a backfill, or a delete path | Every question |
| An effect that leaves the system, or money | Every question, plus a named human approver |

**"Not applicable" is a valid result, and it is the preferred one.** A gate that reports a
finding on every change teaches the team to ignore it. That is the process abandonment
failure of p. 452–453.

---

## 11. What the launch process does not fix

Ch. 27, p. 468–469 names three problems that LCE did not solve. A scalability change, when
usage grows more than two orders of magnitude past the estimate. Growing operational load,
as notification noise and deployment complexity increase. Infrastructure churn, when platform
teams replace features and service owners edit configurations to stay in the same place.
State these limits, because a gate that claims to solve them loses trust.

**Apply a churn reduction policy.** Prohibit an infrastructure engineer from releasing a
backward-incompatible feature until that engineer also automates the client migration. p. 469.

---

## 12. Where to read next

| Question | File |
|---|---|
| What are the four questions and the six classes? | `01-the-reversibility-contract.md` |
| How do I identify and rebuild the prior artifact? | `02-release-engineering.md` |
| How do I size the exposure ladder and the bake time? | `03-progressive-rollout-and-canary.md` |
| A change caused the outage. What is the order of moves? | `04-change-induced-emergency.md` |
| Which change caused it? | `05-locating-the-bad-change.md` |
| The change touches data or schema. | `06-data-and-schema-reversibility.md` |
| An automated actor performs the change or the reversal. | `07-automation-safety.md` |
| I need the exact command. | `08-agent-and-operator-shortcuts.md` |
| Which hazards apply to this diff? | `rollback-hazard-catalog.md` |
