# Engineering Skills

A suite of Agent Skills that make an AI coding agent work like a senior engineer. The skills
enforce one order of work:

1. Clarify the problem before you design it.
2. Size it in numbers before you choose an architecture.
3. Record each design decision with the alternative you rejected.
4. Write the code against robustness invariants.
5. Check the code against the spec, with evidence.
6. Review for the failure modes that appear only under concurrency, failure, attack, and scale.
7. Operate it: set the reliability target, see the system, find the fault, undo the change,
   and learn from the incident.

Grounded in ten books:

| Book | What it contributes |
|---|---|
| **[*Designing Data-Intensive Applications*][ddia]** — Kleppmann | Correctness under concurrency, replication, and failure. The 48-hazard catalog. |
| **[*Cryptography and Network Security*][stallings]** — Stallings | Security services and mechanisms, threat framing, the 64-vulnerability catalog |
| **[*Advanced Programming in the UNIX Environment*][apue]** — Stevens and Rago | System call semantics, atomicity, durable file writes, signals, threads. Most of the 55-failure catalog. |
| **[*Systems Programming*][donovan]** — Donovan | The machine model, the design procedure for a system program, the four loader functions |
| **[*System Design Interview: An Insider's Guide*][xu]** — Alex Xu | The design method, back-of-the-envelope estimation, the scaling ladder |
| **[*Grokking the System Design Interview*][grok]** — Design Gurus | Building-block selection. The seven-step design process. |
| **[*Database Internals*][dbi]** — Alex Petrov | Database selection methodology, storage engine trade-offs, the RUM conjecture |
| **[*Code Simplicity*][cs]** — Max Kanat-Alexander | The laws of software design. The anti-over-engineering discipline. |
| **[*Site Reliability Engineering*][sre]** — Beyer, Jones, Petoff, Murphy | Error budgets, the four golden signals, the troubleshooting model, incident command, cascading failure, release reversibility |
| **[*Release It!*][relit]** — Michael T. Nygard | The eleven stability antipatterns and the eight stability patterns. Integration-point failure, capacity antipatterns, Transparency |

---

## Why this exists

Agents are good at producing code that works on the happy path in a single-threaded test.
They are much worse at two things that cause real problems.

**They skip the design process.** They answer before clarifying, propose architectures with
no numbers behind them, and choose Kafka at 40 writes per second. The estimation gate and
the simplicity pass exist to stop that.

**They miss the failure modes that a passing test does not show.** These fall into three
groups.

*Under concurrency and scale:*

- read-modify-write races that silently lose updates
- check-then-act logic that double-books, double-charges, or oversells
- migrations that break during the rolling deploy window
- retries without idempotency that duplicate side effects
- dual writes that leave two datastores permanently divergent
- a missing timeout that makes one slow dependency a total outage
- wall-clock timestamps used to order events across machines

*Against the kernel, on one machine:*

- a file updated in place, so a crash leaves half the old content and half the new
- a write reported as durable that never reached the disk, because nothing called `fsync`
- a rename that reaches the disk before the data, leaving the right name and no content
- a short `read` or `write` treated as the whole transfer
- a signal handler that calls `malloc` or `printf` and corrupts the heap
- a condition variable waited on with `if` instead of `while`
- a `fork` in a threaded process, where a library mutex stays locked forever

*When someone is attacking:*

- tokens and salts generated from `Math.random()`, which is unpredictable to a statistician
  and trivially predictable to an attacker
- encryption used where integrity was needed, so the ciphertext is malleable
- a signature that proves authorship and is silently assumed to prove freshness, leaving the
  endpoint replayable
- authentication mistaken for authorization — a logged-in user reading another tenant's row
- an internal service that trusts an `X-User-Id` header any pod can set
- a reset endpoint with no throttling, turning a six-digit code into a solved problem
- an attack that succeeds and leaves no audit record, because the only log is one the
  attacker can edit

*After it ships, while it runs:*

- a change that nobody can undo, because the migration dropped the column the old code reads
- a `git reset --hard` on a branch that two other people already pulled
- an agent that edited nine files across four turns, with no checkpoint between them
- a retry loop that turns one slow vendor into a self-inflicted denial of service
- a call to a vendor API with no read timeout, so one hung socket drains the pool
- a circuit breaker whose state no dashboard shows, so nobody knows the feature is off
- an alert that pages a human who can do nothing about it, until humans stop reading pages
- a service with a 99.99% target that depends on four services with 99.9% targets
- an availability measured on the server, where it cannot see the failure the user saw
- an outage debugged by guessing, because nobody wrote down which hypothesis is already dead
- a load test that passed because it aimed at QA, against a database with 100 rows
- a backup that nobody ever restored, which means there is no backup

This suite turns all four groups into **named, detectable catalogs** — 48 data hazards, 55
systems failures, 64 security vulnerabilities, and 328 operations defects across seven
catalogs. The planning, implementation, verification, and review skills all check against
them.

---

## Structure

```
skills/
├── engineer-workflow/SKILL.md      # Orchestrator: routes phases, owns shared state
├── planner/SKILL.md                # Recon → requirements → estimates → gates → phased plan.md
├── detail-planning/SKILL.md        # One phase → implementable spec in executor.md
├── implement/SKILL.md              # Spec → code, with robustness invariants
├── verify/SKILL.md                 # Code vs. spec, with evidence
├── code-review/SKILL.md            # 6 review modes, including /review-data, /review-security
├── system-design/                  # WHAT to build
│   ├── SKILL.md                    # 7-step method, design doc, red flags, proportionality
│   └── references/
│       ├── 01-design-method.md             # The steps in depth + question banks
│       ├── 02-estimation.md                # Powers of two, latency numbers, nines, formulas
│       ├── 03-scaling-ladder.md            # Single server → sharding, with triggers
│       ├── 04-building-blocks.md           # LB, cache, CDN, queues, rate limiting, IDs, protocols
│       ├── 05-database-selection.md        # Evaluation method, B-tree vs LSM, RUM conjecture
│       ├── 06-simplicity-and-design-laws.md # Six laws, three flaws, over-engineering tests
│       └── 07-reference-architectures.md   # Fan-out, chat, crawler, file sync, autocomplete…
├── data-systems-design/            # Whether it stays CORRECT
│   ├── SKILL.md                    # Decision tables, design record, language discipline
│   └── references/
│       ├── 01-reliability-scalability-maintainability.md
│       ├── 02-data-models.md
│       ├── 03-storage-and-retrieval.md
│       ├── 04-encoding-and-evolution.md
│       ├── 05-replication.md
│       ├── 06-partitioning.md
│       ├── 07-transactions.md
│       ├── 08-distributed-systems-faults.md
│       ├── 09-consistency-and-consensus.md
│       ├── 10-batch-processing.md
│       ├── 11-stream-processing.md
│       ├── 12-correctness-and-integrity.md
│       └── hazard-catalog.md       # 48 named hazards: signature → consequence → fix
├── systems-programming/            # Whether the CODE survives the kernel
│   ├── SKILL.md                    # 5 syscall rules, 7 file recipes, review scan, glossary
│   └── references/
│       ├── 01-file-descriptors-and-io.md    # fds, short counts, atomicity, fsync, mmap, locks
│       ├── 02-files-and-directories.md      # stat, links, rename, durable update, path safety
│       ├── 03-standard-io-and-buffering.md  # Buffer modes, flush vs fsync, fork duplication
│       ├── 04-processes-and-execution.md    # fork/exec/wait, exit status, races, daemon rules
│       ├── 05-signals.md                    # sigaction, async-signal-safety, EINTR, self-pipe
│       ├── 06-threads-and-concurrency.md    # Mutexes, condvars, lock order, fork with threads
│       ├── 07-ipc-and-sockets.md            # Pipes, framing, shared memory, fd passing
│       ├── 08-toolchain-and-machine-model.md # Assemble/link/load, symbols, storage classes
│       └── failure-catalog.md      # 55 named failures: signature → consequence → fix
├── security-engineering/           # Whether it survives an ATTACKER
│   ├── SKILL.md                    # Threat model, security services, decision tables, security record
│   └── references/
│       ├── 01-threat-model-and-security-services.md
│       ├── 02-cryptographic-primitives-and-modes.md
│       ├── 03-randomness-and-key-management.md
│       ├── 04-public-key-and-key-exchange.md
│       ├── 05-integrity-hashes-and-macs.md
│       ├── 06-signatures-and-authentication-protocols.md
│       ├── 07-identity-certificates-and-pki.md
│       ├── 08-transport-and-channel-security.md
│       ├── 09-authentication-and-access-control.md
│       ├── 10-intrusion-detection-and-audit.md
│       ├── 11-malicious-software-and-availability.md
│       ├── 12-perimeter-and-trusted-systems.md
│       └── vulnerability-catalog.md # 64 named vulnerabilities: signature → consequence → fix
├── slo-engineering/                # What "reliable enough" MEANS, as a number
│   ├── SKILL.md                    # Risk, indicators, objectives, the error budget, the toil budget
│   └── references/
│       ├── 01-risk-and-the-error-budget.md
│       ├── 02-slis-slos-and-slas.md
│       ├── 03-availability-math.md          # The nines table, dependency multiplication, burn rate
│       ├── 04-the-toil-budget.md            # Toil defined. The 50% cap
│       ├── 05-gathering-availability-requirements.md
│       └── slo-defect-catalog.md            # 47 named defects: signature → consequence → fix
├── observability/                  # Whether you can SEE it
│   ├── SKILL.md                    # Golden signals, symptom alerting, what may page a human
│   └── references/
│       ├── 01-the-four-golden-signals.md
│       ├── 02-symptom-based-alerting.md     # The four questions to ask before any alert
│       ├── 03-white-box-and-black-box.md
│       ├── 04-time-series-and-rules.md
│       ├── 05-transparency-and-logging.md   # A log line is a user interface
│       ├── 06-the-operations-database.md
│       └── observability-gap-catalog.md     # 44 named gaps: signature → consequence → fix
├── capacity-engineering/           # Which resource RUNS OUT first
│   ├── SKILL.md                    # The constrained resource, load testing, the capacity plan
│   └── references/
│       ├── 01-defining-capacity.md          # The capacity myths, each stated then refuted
│       ├── 02-capacity-antipatterns.md      # The ten capacity antipatterns
│       ├── 03-capacity-patterns.md
│       ├── 04-load-testing.md               # Why a test aimed at QA passes and production fails
│       ├── 05-intent-based-capacity-planning.md
│       ├── 06-load-balancing-and-utilization.md
│       └── capacity-defect-catalog.md       # 47 named defects: signature → consequence → fix
├── self-healing-apis/              # Whether a VENDOR can take you down
│   ├── SKILL.md                    # Detect → localize → classify → remediate → verify → escalate
│   └── references/
│       ├── 01-integration-points.md         # The six ways one remote call fails
│       ├── 02-stability-antipatterns.md     # Nygard's eleven
│       ├── 03-stability-patterns.md         # Nygard's eight, with the breaker state machine
│       ├── 04-fault-localization.md         # Us, the network, the vendor, or a shared dependency
│       ├── 05-overload-and-load-shedding.md
│       ├── 06-cascading-failure.md
│       ├── 07-automated-remediation-safety.md # What a self-healer may and may not do
│       ├── 08-fault-injection-and-test-harness.md
│       ├── 09-vendor-slas-and-degradation.md
│       └── integration-fault-catalog.md     # 48 named faults: signature → consequence → fix
├── production-troubleshooting/     # HOW to find the cause, instead of guessing
│   ├── SKILL.md                    # Triage → examine → diagnose → test and treat → cure
│   └── references/
│       ├── 01-the-troubleshooting-model.md
│       ├── 02-triage-first-diagnose-second.md # Mitigate before you understand
│       ├── 03-examine-and-diagnose.md       # Bisect the request path
│       ├── 04-test-and-treat.md
│       ├── 05-negative-results-and-bias.md  # A negative result is a result
│       ├── 06-making-troubleshooting-easier.md
│       └── diagnostic-trap-catalog.md       # 40 named traps: signature → consequence → fix
├── incident-response/              # How to RUN the outage, and learn from it
│   ├── SKILL.md                    # Command roles, live state, handoff, blameless postmortem
│   └── references/
│       ├── 01-incident-command.md           # Commander, operations lead, communications lead
│       ├── 02-emergency-response.md
│       ├── 03-on-call.md
│       ├── 04-postmortem-culture.md
│       ├── 05-tracking-outages.md
│       ├── 06-interrupts-and-overload.md
│       └── incident-failure-catalog.md      # 52 named failures: signature → consequence → fix
└── reverse-branching/              # How to UNDO it
    ├── SKILL.md                    # The reversibility contract, revert vs. roll forward
    └── references/
        ├── 01-the-reversibility-contract.md   # Declare the undo before the change ships
        ├── 02-release-engineering.md          # Hermetic builds, versioned configuration
        ├── 03-progressive-rollout-and-canary.md
        ├── 04-change-induced-emergency.md     # Revert first. Diagnose after
        ├── 05-locating-the-bad-change.md      # Bisection. "What changed last"
        ├── 06-data-and-schema-reversibility.md # Code reverts. Data does not
        ├── 07-automation-safety.md            # The automation is the blast radius
        ├── 08-agent-and-operator-shortcuts.md # The checkpoint and revert command surface
        ├── 09-launch-coordination.md
        └── rollback-hazard-catalog.md         # 50 named hazards: signature → consequence → fix
```

Each skill is a self-contained folder with a `SKILL.md`, which is the layout Claude Code,
Cursor, and compatible agents expect.

**The gate skills split the problem deliberately.** Each one answers a different question.

| Gate skill | The question it answers |
|---|---|
| `system-design` | What must we build? Scope, capacity, components, trade-offs |
| `data-systems-design` | Does it stay correct across machines? Isolation, replication, ordering |
| `systems-programming` | Does the code survive one kernel? Short reads, atomicity, durability, signals |
| `security-engineering` | Does it survive an adversary? Trust boundaries, identity, secrets |
| `slo-engineering` | How reliable must it be, in a number we can spend? |
| `observability` | Can we see it? Which signals exist, and which of them may page a human |
| `capacity-engineering` | Which resource runs out first, and at what load? |
| `self-healing-apis` | Can a vendor take us down, and what do we do without a human? |
| `production-troubleshooting` | When it breaks, how do we find the cause instead of guessing? |
| `incident-response` | Who runs the outage, and what do we learn from it? |
| `reverse-branching` | How do we undo this change? |

These are different questions. Agents most often merge the fourth into the second by
mistake. Correctness analysis assumes that faults are random. Security analysis assumes that
an attacker chooses them. A queue consumer that is fully idempotent under duplicate delivery
can still be a replay vulnerability. Idempotency answers "did this happen twice" and not "did
the right party ask for it".

The second common merge is design into operation. A design gate asks whether the system can
be correct. An operations gate asks what happens at 03:00 when it is not. A design that is
correct and unobservable is an outage that nobody can end.

A new public service runs the first four. Adding a retry to an existing call runs only
`data-systems-design`. A file-ingest pipeline or a daemon runs `systems-programming`. Adding
an authenticated endpoint runs `security-engineering`. Any change that reaches production
runs `reverse-branching`. Any new call to a system you do not own runs `self-healing-apis`.
An outage in progress runs `production-troubleshooting` and `incident-response`.

---

## Installation

### Claude Code plugin

Register this repository as a marketplace, then install the complete skill suite:

```text
/plugin marketplace add Cool-Coder174/engineering-skills
/plugin install engineering-skills@engineering-skills
```

The plugin is installed at user scope and its skills load automatically.

### Manual installation

Alternatively, copy the skill folders into your agent's skills directory:

```bash
# Claude Code / Cursor (user-level)
cp -r skills/* ~/.claude/skills/
cp -r skills/* ~/.cursor/skills/

# Project-level
cp -r skills/* .claude/skills/
```

On Windows (PowerShell):

```powershell
Copy-Item -Recurse -Force skills\* $HOME\.cursor\skills\
```

The agent finds each skill by its folder name. Install the ten catalog skills together with
the rest: `data-systems-design`, `systems-programming`, `security-engineering`,
`slo-engineering`, `observability`, `capacity-engineering`, `self-healing-apis`,
`production-troubleshooting`, `incident-response`, and `reverse-branching`. The scans in the
other skills refer to those catalogs by path, and a missing folder turns a scan into a
silent no-op.

---

## The pipeline

```
PLANNING → ARCHITECTURE → DESIGN → SYSTEMS → SECURITY → DETAIL PLANNING → IMPLEMENT → VERIFY → REVIEW
 plan.md    ##Architecture  ##Design  ##Systems  ##Security   executor.md      code      report   findings
```

The pipeline ends at merge. The operations skills run on either side of it. Four of them run
*before* the merge, because a reliability target, a signal, a capacity number, and a reverse
path all have to exist before the change ships. Three of them run *after* it, when something
has already gone wrong.

```
   BEFORE THE MERGE                          AFTER IT SHIPS
   slo-engineering      the target        production-troubleshooting   find the cause
   observability        the signals       incident-response            run it, then learn
   capacity-engineering the ceiling       reverse-branching            undo the change
   self-healing-apis    the degraded mode
   reverse-branching    the reverse path
```

| Command | What happens |
|---|---|
| `engineer-workflow: [idea]` | Start the full pipeline |
| `planner` | Recon the codebase, quantify requirements, estimate capacity, produce phased `plan.md` |
| `design` / `system design` | Run the 7-step design method, estimation, and simplicity pass |
| `estimate` | Just the back-of-the-envelope numbers |
| `data design` | Run the data-systems design gate and hazard scan |
| `systems` / `low level` | Run the systems gate: syscall rules, file recipes, failure scan |
| `security design` | Run the security gate: threat model and vulnerability scan of the design |
| `detail F1` | Expand phase F1 into a spec with contracts, failure modes, rollback |
| `implement F1` | Write the code, enforcing the robustness invariants |
| `verify F1` | Compare code to spec, with file-and-line evidence |
| `/review-diff` | Review the branch vs. main, plus PR/CI babysit status |
| `/review-uncommitted` | Catch work that looks finished but isn't |
| `/review` | Full repository audit |
| `/review-inscope` | Check the change against the stated task |
| `/review-data` | Deep data/concurrency/distribution hazard audit |
| `/review-security` | Deep security audit: threat model, then the vulnerability catalog |
| `slo` / `error budget` | Set the indicator, the objective, and the budget policy |
| `observability` / `what should page` | Choose the signals, then decide what may wake a human |
| `capacity` / `will this hold` | Find the constrained resource and the load that saturates it |
| `self-heal` / `vendor is down` | Localize the fault, then choose a remediation inside the safety envelope |
| `troubleshoot` / `why is it broken` | Run the triage, examine, diagnose, test loop |
| `incident` / `declare an incident` | Assign the roles, open the live state, run the response |
| `postmortem` | Write the blameless record, with owned and dated action items |
| `reverse` / `revert this` / `checkpoint` | Produce the reverse path, or run the checkpoint and revert commands |

Each phase stops when it's done and waits for you. One phase per cycle, by design.

### Example

```
> engineer-workflow: add a wallet with balance top-ups via Stripe

  → Planning: recon, requirements, explicit non-goals, phases F1–F4
  → Estimation: 40 top-ups/min peak, 2 KB/row, 1.2 GB/yr
      ⇒ single Postgres primary has years of headroom. No queue, no sharding.
  → Architecture gate: API contract, data model, write path walked end to end,
      trade-off table (ledger table vs. balance column → ledger, for auditability)
      Simplicity pass: Kafka removed (no number justifies it), Redis kept (p99 target)
  → Design gate triggers (money + persistence + external side effects):
      INV-1  balance never negative  → CHECK constraint + atomic UPDATE
      INV-2  one charge per request  → unique index on idempotency_key
      H-14 non-idempotent retry, H-06 side effect in transaction → mitigated in F2
  → Systems gate: not triggered. No file, process, or signal work in this change.
  → Security gate triggers (money + a Stripe webhook + a new endpoint):
      T-1  unauthenticated caller replaying a captured webhook
           → signature verified, 5-minute window, seen event_id cached (V-25, V-26)
      T-2  authenticated tenant reading another tenant's wallet
           → tenant predicate in the repository, negative test in F2 (V-44)
      Residual risk accepted: no per-request proof-of-possession on the session
      cookie. Mitigated by a 30-minute lifetime and server-side revocation (V-28)
  → STOP

> detail F2
  → Spec: files, signatures, failure-mode table, idempotency contract,
    isolation level, migration compatibility matrix, observability, rollback
  → STOP

> implement F2
  → Code + concurrency tests, validation run, deviations logged
  → STOP

> verify F2
  → ✗ FAIL: dedup uses a pre-flight SELECT outside the transaction (H-03/H-14)
    with the exact fix
```

---

## What the architecture gate produces

For a new system, a new datastore, a new component on the request path, or a scaling change:

- **Requirements** clarified before anyone proposes a solution, with explicit non-goals and
  labeled assumptions. Answering fast without clarifying is how you build the wrong system
  correctly
- **Capacity estimates** with derivations shown, each row marked measured or estimated, and
  the conclusion the numbers force ("sharding is not justified. The bottleneck at 10× is the
  `events` write path.")
- **An interface contract** written before the architecture, which is what makes the
  requirements concrete
- **A data model driven by access patterns**, with a datastore decision record listing the
  rejected alternative, the operability story, and the exit cost
- **Deep dives on the two or three components where the difficulty actually lives** — not
  even attention across every box
- **A bottleneck, failure-mode, and operations table**, including what saturates first, the
  degraded mode per dependency, and the next scale curve
- **A simplicity pass** that removes every component the numbers do not justify

---

## What the design gate produces

For anything touching persistence, concurrency, distribution, money, auth, or PII:

- **Requirements** as numbers — load parameters, p50/p99 (never averages, never averaged
  percentiles), RPO/RTO, and a consistency level chosen for a stated user-visible reason
- **A fault model** — which faults are tolerated, the resulting behavior, and blast radius
- **Decisions with rejected alternatives** — data model, storage engine, replication,
  partition key, isolation level, delivery semantics
- **Invariants with enforcement mechanisms** — "the application checks it first" is not a
  mechanism under concurrency
- **An evolution plan** — expand → migrate → contract, with a rollback path
- **"What breaks at 10x"** — the specific resource that saturates first
- **A hazard scan** against the catalog

It also enforces **language discipline**: phrases like "eventually consistent",
"we'll retry", "exactly once", "we'll keep them in sync", "add a cache", and
"use a distributed lock" are rejected unless accompanied by the actual mechanism.

---

## The hazard catalog

48 named hazards, each with a **detection signature** (what to grep for), a
**consequence**, and a **required fix** — grouped as:

| Group | Examples |
|---|---|
| A. Concurrency & transactions | H-01 lost update · H-02 write skew · H-03 uniqueness in app code · H-06 side effect in transaction |
| B. Schema & API evolution | H-09 breaking change in one deploy · H-11 unknown-field dropping · H-12 blocking DDL |
| C. Distributed calls & retries | H-14 non-idempotent retry · H-15 missing timeout · H-17 nested retries |
| D. Clocks, locks, leadership | H-20 wall-clock ordering · H-22 lock without fencing token · H-24 cross-channel race |
| E. Replication & partitioning | H-25 read-after-write from a replica · H-29 hot partition key · H-30 `hash mod N` |
| F. Streams, queues, derived data | H-32 dual write · H-33 unbounded queue · H-36 processing-time windowing |
| G. Reliability & operability | H-41 averages instead of percentiles · H-43 N+1 · H-44 cache without invalidation |

Every skill shares the catalog. `code-review` scans a diff with it, `verify` scans an
implementation, and `planner` scans a proposed design. The same defect therefore gets caught
at whichever stage it appears.

---

## What the systems gate produces

For a change that opens, writes, moves, or locks a file, that starts a process, that handles a
signal, or that shares memory between threads:

- **A durable-write recipe, named and applied** — temporary file in the same directory,
  `fsync`, `rename`, then `fsync` on the directory that holds it. "We write the file" is not a
  durability claim, and step 6 is the step engineers omit most often
- **Each pair of calls that forms one logical operation, replaced by the atomic call.** Use
  `O_APPEND` instead of seek-then-write, `O_EXCL` instead of test-then-create, and `pread`
  instead of seek-then-read
- **A loop around every `read` and `write`** that crosses a pipe, a socket, or a large buffer.
  A short count is normal behavior and not an error
- **A signal contract** — which signals the process handles, which functions each handler may
  call, and which work it defers to the main loop
- **A lock-order statement** for any code path that holds two mutexes
- **A language exposure check** — which of the five system call rules the runtime hides, and
  which it does not. No runtime makes two calls atomic. No runtime calls `fsync` for you.
- **A failure scan** against the catalog

The gate also names the pass structure for any program that transforms data. One question
decides it: does any output depend on input the program has not read yet? If yes, the program
needs two passes or a patch list.

---

## The failure catalog

55 named failures, each with a **detection signature**, a **consequence**, and a **required
fix** — grouped as:

| Group | Examples |
|---|---|
| A. File descriptors and I/O | S-01 unchecked `write` · S-03 short `read` as the whole file · S-06 `stat` before `open` · S-07 durable write with no `fsync` |
| B. Files, names, directories | S-11 write over a file in place · S-13 data reaching the disk after the rename · S-16 unchecked path from a user |
| C. Buffered streams | S-20 output a pipeline never shows · S-21 output duplicated after `fork` · S-22 stream and descriptor on one file |
| D. Processes | S-25 no `wait` for a child · S-28 command built from user input · S-29 descriptor leaked into a child |
| E. Signals | S-31 unsafe function in a handler · S-33 handler that loses `errno` · S-35 `SIGPIPE` with no handling |
| F. Threads | S-37 `pthread_cond_wait` inside an `if` · S-38 no lock order · S-42 `fork` in a process that has threads |
| G. IPC and sockets | S-45 stream with no framing · S-46 peer length used as an allocation size · S-47 network call with no timeout |
| H. Memory and toolchain | S-50 pointer to a returned local · S-51 size computed by multiplication · S-55 `struct` written to a socket |

Each entry states which languages it applies to. Most of them apply to Python, Go, Java, Rust,
and Node.js without change. A runtime hides `EINTR`. It does not hide the missing `fsync`, and
it does not make two calls atomic.

---

## What the security gate produces

For anything touching identity, authorization, secrets, cryptography, untrusted input, a new
reachable surface, or regulated data:

- **A named adversary with a stated capability** — a control chosen without an attacker in
  mind is a guess. The usual failure is a defense against the wrong attacker
- **The assets and trust boundaries**, so "internal" stops being a security argument
- **The security services owed** — authentication, access control, confidentiality, integrity,
  nonrepudiation, availability. Choose them per asset. Never assume them from the fact that
  something is encrypted
- **A mechanism per service**, at a named enforcement point in the code
- **A vulnerability scan** against the catalog, run against the design and again against the
  implementation
- **Detection** — which security events are emitted, with which fields, and where the audit
  trail lives
- **Residual risk, stated and owned** — the attacks this design does not stop, and why that is
  acceptable

It enforces the same language discipline as the design gate: "we validate the input", "it's
behind the VPN", "we sanitize it", "we encrypt it", and "internal tool, low risk" are rejected
unless accompanied by the boundary, the canonical form, and the mechanism.

---

## The vulnerability catalog

64 named vulnerabilities, each with a **detection signature**, a **consequence** traced to the
principle it violates, and a **required fix** — grouped as:

| Group | Examples |
|---|---|
| A. Cipher selection and modes | V-01 ECB · V-02 reused IV · V-05 keystream reuse · V-08 home-grown construction |
| B. Randomness, keys, and secrets | V-09 non-cryptographic PRNG · V-12 long-lived key used directly · V-14 secret in source |
| C. Integrity and message authentication | V-15 encryption as integrity · V-18 naive keyed hash · V-21 non-constant-time compare |
| D. Authentication protocols and freshness | V-25 no replay protection · V-28 reusable bearer credential · V-30 session not renewed |
| E. Identity, certificates, and trust | V-32 verification disabled · V-33 key from an unauthenticated channel · V-35 unauthenticated DH |
| F. Channel and transport security | V-37 unprotected sensitive traffic · V-38 downgrade permitted · V-42 handshake not bound |
| G. Access control and privilege | V-43 no authorization check · V-44 incomplete mediation · V-46 trust by network position |
| H. Passwords and credential storage | V-48 recoverable password · V-50 fast hash · V-53 unthrottled guessing · V-54 default credentials |
| I. Audit, detection, and logging | V-55 event not audited · V-56 modifiable audit trail · V-58 secrets in logs |
| J. Untrusted input, malicious code, availability | V-59 input reaching an interpreter · V-60 development backdoor · V-62 unbounded work |

The same sharing rule applies. The planner scans the design. `detail-planning` turns each
mitigation into a step with a negative test. `implement` enforces it. `verify` and
`code-review` then check that the mechanism exists where the record claimed it would.

---

## What the operations gates produce

The four design gates ask whether the system can be correct. The operations gates ask what
happens when it is not. They run on any change that reaches a real user.

**`slo-engineering`** turns "it should be reliable" into a number that can be spent. An
indicator measured where the user feels it, an objective with a stated window, an error
budget in minutes, and a written policy for what happens when the budget is gone. It refuses
a 100% target, because a 100% target makes every release a negotiation with no rule. It also
computes the ceiling your dependencies impose, so a 99.99% promise on top of four 99.9%
services gets caught on the whiteboard instead of in the postmortem.

**`observability`** decides what the system exposes and what may interrupt a person. Latency
split into successful and failed requests, traffic, errors, saturation. Then the harder half:
an alert must be urgent, actionable, and about a symptom a user can feel. Everything else is
a ticket or a log line. An alert that a human cannot act on is a defect in the alert.

**`capacity-engineering`** finds the resource that runs out first and names the load that
gets there. Requests per second is not a capacity number. Connections, threads, file
descriptors, memory per session, and database connections are. It also rejects the load test
that aims at QA, because the test that passes against 100 rows is the test that taught the
team nothing.

**`self-healing-apis`** treats every call to a system you do not own as a risk with six
distinct failure shapes, not one. A timeout on every blocking wait, a circuit breaker with a
visible state, a bulkhead that keeps one vendor from draining the shared pool, and a written
degraded mode per integration point. It then bounds what the healing may do on its own:
idempotent, rate limited, observable, reversible, and with a stop control.

**`production-troubleshooting`** replaces guessing with a loop. Triage, examine, diagnose,
test and treat, cure. Mitigate before you understand, but preserve the evidence first.
Write down the hypothesis you killed, because a negative result is a result and the next
responder needs it.

**`incident-response`** gives the outage a commander, an operations lead, a communications
lead, and one live document that everybody reads. It declares early, because an incident you
closed in ten minutes costs less than an incident nobody declared. Then it writes the
postmortem without naming a person, and rejects an action item with no owner and no date.

**`reverse-branching`** makes the undo explicit before the change ships. Every change
declares its reversibility class, its reverse action, the actor allowed to run it, and the
measured time it takes. It knows the part everyone forgets: code reverts, data does not. A
`git revert` past a dropped column restores the code and leaves the corruption.

---

## The operations catalogs

Seven catalogs, 328 named defects, in the same format as the other three — a **detection
signature**, a **consequence**, and a **required fix**.

| Catalog | Codes | Groups |
|---|---|---|
| `slo-engineering/references/slo-defect-catalog.md` | 47 (`L-01` …) | Objective definition · indicator and measurement point · aggregation · the error budget · release policy · dependency arithmetic · toil · expectation |
| `observability/references/observability-gap-catalog.md` | 44 (`M-01` …) | Signals that do not exist · measurements that lie · alerts that must not page · the outside view · logs as an operator interface · the observer itself |
| `capacity-engineering/references/capacity-defect-catalog.md` | 47 (`C-01` …) | The capacity number · the load test · pooled resources · sessions · per-request waste · the database · cache and memory · load distribution · the plan |
| `self-healing-apis/references/integration-fault-catalog.md` | 48 (`I-01` …) | Transport · timeouts and deadlines · retries and load amplification · circuit breakers · bulkheads · health checks · payload · telemetry · degradation · remediation safety |
| `production-troubleshooting/references/diagnostic-trap-catalog.md` | 40 (`D-01` …) | Traps in the order of work · in reasoning · in the telemetry · in the log and the alert · in the test · in the change hypothesis · in the tools · in the record |
| `incident-response/references/incident-failure-catalog.md` | 52 (`N-01` …) | Command and coordination · the live record · response order · diagnosis under pressure · detection and alert hygiene · the postmortem · on-call load |
| `reverse-branching/references/rollback-hazard-catalog.md` | 50 (`R-01` …) | The reversibility contract · build and artifact identity · branch and commit hygiene · rollout and exposure · configuration reversal · data and schema reversal · automation and agent safety · detection and record |

Ten catalogs now share one format and one severity key. 🔴 blocks the merge. 🟡 causes an
outage or a wrong result under load. 🔵 is a risk to operation or maintenance. Each entry
cites the chapter and page it comes from. An entry that current practice added after the
books were written carries the **Modern** tag instead, so nothing modern is attributed to a
2007 or 2016 text.

---

## Automated reverse branching

`reverse-branching` answers a question an AI coding agent raises that a human team rarely
did: an agent edited nine files across four turns, and something is wrong. What exactly do
you undo?

The skill gives both actors the same command surface, and attaches a safety rule to every
command.

| Goal | Command | Safety rule |
|---|---|---|
| Mark a safe point | `git commit -m "checkpoint: <name>"` | A stash is not a checkpoint. It has no stable name, and `git stash drop` asks nothing |
| Name it | `git tag ckpt/<task>-<step>` | A tag survives a rebase of the branch |
| Undo a published commit | `git revert <sha>` | It adds a commit. Every other clone stays valid |
| Undo one file | `git restore --source=<ref> -- <path>` | It keeps the rest of the work |
| Undo the last agent turn | `git reset --hard <checkpoint>` | Local, unpublished branches only. Never a shared branch |
| Recover lost work | `git reflog` | Local, and it expires. The last resort, not the plan |
| Isolate an agent | `git worktree add ../agent-<task>` | One actor writes per tree |
| Find the bad change | `git bisect run <script>` | The test must give the same answer every time |

Every command in that file is marked **Modern**. Neither book names a version-control tool.
The books supply the safety rule. The command is the modern form of the rule.

The rest of the skill is the part that matters more. A reverse path that stops at the commit
is not a reverse path:

- **A configuration change is a change.** SRE defines a push as any change to the running
  software *or its configuration*.
- **A change inherits the class of its least reversible part.** A pull request that adds a
  function and drops a column is a destructive schema change.
- **Some effects never come back.** A sent message, a captured payment, a file delivered to a
  partner, an erased disk. The skill makes you name them before the merge, not after.
- **Replication is not a backup, and a backup nobody restored is not a backup.** SRE spends a
  chapter on this, including the restore that took seven days.
- **The automation is the blast radius.** Automation applies one mistake everywhere at once,
  so any automated reverter needs a rate limit, a canary, and a stop control.

---

## Self-healing APIs

`self-healing-apis` answers the second half of the same problem: a vendor API you integrated
is misbehaving, and you want the system to work out *where* the fault is before it wakes
anybody.

It starts from Nygard's point that one remote call does not fail one way. It fails six:
the connection is refused, the connection hangs in the TCP stack, the connection is accepted
and never answered, the answer is slow, the answer is protocol garbage, or the answer is a
well-formed error. A `try`/`catch` around the call handles one of them.

The localization step is a decision procedure, not a hunch. It separates four cases — the
fault is in us, in the network, in the vendor, or in a dependency we share with them — and
names the evidence that distinguishes each. Then it chooses a remediation from a list ordered
by blast radius, and every entry has a precondition and a stop condition:

```
retry one request → open a breaker → shed load → fail over → restart → roll back → page a human
```

The safety envelope is the point. An automated remediation must be idempotent, rate limited,
observable, and reversible, and it must never make a change whose effect it cannot measure.
The retry is where teams get this wrong: a retry at every layer multiplies, and an automatic
retry storm is a denial of service you inflicted on yourself.

---

## Design principles

**Proportionality.** Every skill has an explicit anti-over-engineering rule. A copy change
gets a three-bullet micro-plan and no hazard scan. A new source of truth gets the full
design record. Applying distributed-systems ceremony to a single-file fix is itself a
failure mode. Kanat-Alexander's version: the quality level of a design should be
proportional to how long the system will keep helping people.

**Numbers before architecture.** No component enters a design without an estimate or a
requirement behind it. "It should scale fine" is not a design, and neither is a diagram with
a queue nobody sized.

**Don't design for a future you can't measure.** The most common and disastrous design error
is predicting something about the future when you cannot know it. Design for measured load
with a stated growth rate. Make the next rung of the scaling ladder reachable. Don't build
it until a number says so.

**Assume the process dies between any two steps.** The kernel can stop a process between any
two calls. The power can fail between a write and the disk. Any operation that takes two calls
is not atomic, at any level of the stack. This is the same rule as check-then-act in a
database, one layer down.

**Maintenance cost outweighs implementation cost.** A design that's fast to build and
expensive to operate is a bad design. Every component added is a permanent tax on whoever
is on call.

**Evidence over assertion.** Verification and review findings must cite `file:line`.
"No critical issues found" is a valid result. Padding a review with invented nits trains
people to ignore reviews.

**Mechanisms over intentions.** Every invariant names the thing that enforces it. Every
retry names its idempotency key. Every cache names its invalidation trigger.

**Stop and ask.** Ambiguity that changes the design, irreversible operations, and
spec/code conflicts stop the pipeline rather than getting a guess.

---

## Using skills independently

Nothing requires the full pipeline:

- `code-review` reads the codebase and git state directly — no `plan.md` needed.
- `system-design` runs standalone for design work: "design a notification service", "will
  this scale?", "should we use Postgres or Cassandra here?", "estimate the storage for this".
  It's also useful as a review lens on an architecture someone else wrote.
- `data-systems-design` answers correctness questions on its own ("should this be
  serializable?", "is this partition key safe?").
- `systems-programming` answers low-level questions on its own: "how do I replace this file
  without losing it on a crash?", "why does my progress output vanish in a pipeline?", "is
  this signal handler safe?", "why is this `undefined reference` when the library is right
  there?". Its seven file-management recipes also work alone. Use them for a document-ingest,
  indexing, or RAG pipeline that must not lose or corrupt a file.
- `security-engineering` answers security questions on its own ("is this token design sound?",
  "how should we store these credentials?", "what can an attacker do with this endpoint?"), and
  works as a threat-modelling lens on a design someone else wrote.
- `planner` is useful alone for turning a vague request into a grounded plan.
- `slo-engineering` answers "what should our target be?", "how do we measure this?", and
  "can we promise 99.99% on top of these dependencies?" without any of the other skills.
- `observability` answers "should this page someone?", "what are we missing?", and "why does
  this dashboard not tell us anything". Its four questions work as a review of an alert
  someone else wrote.
- `capacity-engineering` answers "will this hold on launch day?" and "which number runs out
  first?". Its load-testing reference works alone as a review of an existing test plan.
- `self-healing-apis` answers "this vendor is flaky, what do we do?", "where should the
  timeout go?", and "is our retry making it worse?". Use it as a design lens on any new
  integration, before an outage rather than during one.
- `production-troubleshooting` runs standalone during a live problem. It is the most useful
  skill in the suite for an agent, because an agent's default under uncertainty is to guess,
  and this replaces the guess with a loop.
- `incident-response` runs standalone for an outage in progress, and its postmortem section
  runs alone afterwards on an incident that was handled badly.
- `reverse-branching` answers "how do I undo this?" on its own. Its checkpoint commands are
  worth reading before an agent starts a long editing session, not after.

---

## Migrating from the previous layout

Skills previously lived as flat files at the repository root. They are now folders under
`skills/`, matching the standard agent-skill layout:

| Before | After |
|---|---|
| `engineer-workflow.md` | `skills/engineer-workflow/SKILL.md` |
| `planner.md` | `skills/planner/SKILL.md` |
| `detail_planning.md` | `skills/detail-planning/SKILL.md` |
| `implement.md` | `skills/implement/SKILL.md` |
| `verify.md` | `skills/verify/SKILL.md` |
| `code-review.md` | `skills/code-review/SKILL.md` |
| — | `skills/data-systems-design/` (new) |
| — | `skills/system-design/` (new) |
| — | `skills/systems-programming/` (new) |
| — | `skills/security-engineering/` (new) |
| — | `skills/slo-engineering/` (new) |
| — | `skills/observability/` (new) |
| — | `skills/capacity-engineering/` (new) |
| — | `skills/self-healing-apis/` (new) |
| — | `skills/production-troubleshooting/` (new) |
| — | `skills/incident-response/` (new) |
| — | `skills/reverse-branching/` (new) |

If you installed the old flat files, remove them before you install the new folders.
Otherwise the agent loads two versions of the same skill.

---

## Dependencies

None. These are Markdown skill definitions for AI agents.

## Contributing

Fork, branch, PR. New hazards, failures, vulnerabilities, and operations defects are welcome.
Follow the format of the catalog you add to (signature → consequence → fix) and cite the
source of the failure mode. In the failure catalog, state which languages each entry applies
to. In the vulnerability catalog, mark guidance that post-dates the 4th edition with the
**Modern** tag rather than attributing it to Stallings. Apply the same rule to the seven
operations catalogs: anything the 2016 and 2007 editions could not have known — container
orchestration, service meshes, hosted CI, feature-flag platforms, LLM coding agents — carries
**Modern** and is not attributed to either book.

Prose in this repository follows **ASD-STE100 Simplified Technical English**. Short
declarative sentences. Active voice with a named actor. No semicolons. No phrasal verbs. A
rule or a procedure step stays under 20 words, and description stays under 25. Match the
surrounding files.

## License

MIT

## Attribution

Concepts, terminology, methodology, and reference numbers come from:

- [*Designing Data-Intensive Applications*][ddia] — Martin Kleppmann (O'Reilly, 2017).
  Page references throughout `data-systems-design` refer to that edition.
- [*Cryptography and Network Security: Principles and Practices*][stallings] — William
  Stallings (Prentice Hall, 4th ed., 2005). Security services and mechanisms, the attack
  taxonomy, and the reference-monitor and audit-record models. Page and section references
  throughout `security-engineering` refer to that edition. Guidance that post-dates it is
  marked **Modern** rather than attributed to the book.
- [*System Design Interview: An Insider's Guide*][xu] — Alex Xu (2020). The design
  framework, back-of-the-envelope estimation, and the scaling progression.
- [*Grokking the System Design Interview*][grok] — Design Gurus. The seven-step process and
  the building-block catalog.
- [*Database Internals*][dbi] — Alex Petrov (O'Reilly, 2019). Database evaluation
  methodology, storage engine trade-offs, and the RUM conjecture.
- [*Code Simplicity: The Science of Software Development*][cs] — Max Kanat-Alexander
  (O'Reilly, 2012). The laws of software design and the anti-over-engineering discipline.
- [*Advanced Programming in the UNIX Environment*][apue] — W. Richard Stevens and Stephen A.
  Rago (Addison-Wesley, 3rd ed., 2013). The system call semantics, atomicity rules, file
  management, signals, and threads throughout `systems-programming`.
- [*Systems Programming*][donovan] — John J. Donovan (McGraw-Hill, 1972). The machine model,
  the design procedure for a system program, and the four functions of a loader.
- [*Site Reliability Engineering: How Google Runs Production Systems*][sre] — Betsy Beyer,
  Chris Jones, Jennifer Petoff and Niall Richard Murphy (O'Reilly, 2016). Error budgets and
  service level objectives, the four golden signals, the troubleshooting model, incident
  command, cascading failure, release engineering, and data integrity. Chapter and page
  references throughout `slo-engineering`, `observability`, `production-troubleshooting`,
  `incident-response`, `capacity-engineering`, and `reverse-branching` refer to that edition.
- [*Release It! Design and Deploy Production-Ready Software*][relit] — Michael T. Nygard
  (Pragmatic Bookshelf, 2007). The eleven stability antipatterns and the eight stability
  patterns, integration-point failure modes, the capacity antipatterns, and Transparency.
  Section and page references throughout `self-healing-apis` and `capacity-engineering` refer
  to the first edition. The link goes to the current edition, which is the one you can buy.

This repository contains original prose that applies those concepts to agent workflows. It is
not a reproduction of any of the books. Read them.

[ddia]: https://dataintensive.net/
[stallings]: https://williamstallings.com/Cryptography/
[xu]: https://www.systemdesigninsider.com/
[grok]: https://www.designgurus.io/course/grokking-the-system-design-interview
[dbi]: https://www.databass.dev/
[cs]: https://www.codesimplicity.com/
[apue]: https://www.apuebook.com/
[donovan]: https://archive.org/details/systemsprogrammi0000dono
[sre]: https://sre.google/sre-book/table-of-contents/
[relit]: https://pragprog.com/titles/mnee2/release-it-second-edition/
