# Intent-Based Capacity Planning

**Sources:** *Site Reliability Engineering: How Google Runs Production Systems* — Beyer, Jones,
Petoff, Murphy (O'Reilly, 2016). Ch. 18, "Software Engineering in SRE", p. 248–269. The Auxon
case study covers p. 250–269. Supporting citations: SRE Ch. 5, p. 75–80 for toil, and SRE Ch. 3,
p. 58–61 for the error budget. Also SRE Ch. 4, p. 62–63 for the SLO. For continuous capacity
monitoring, *Release It!* — Michael T. Nygard (Pragmatic Bookshelf, 2007), Sec. 8.6, p. 174.

This file answers one question. How do you write a capacity plan that survives a change?

The chapter gives the motto in one line. "Specify the requirements, not the implementation."
(Ch. 18, p. 253.)

**A plan that you cannot recompute is a plan that you will not update.** Chapter 18 does not
write that sentence. It proves it. The chapter shows a plan that breaks on a minor change, and
then a plan that a machine regenerates from stored inputs.

---

## 1. The traditional cycle

SRE names four steps in the traditional cycle (Ch. 18, p. 250–251).

| Step | The action | The question it answers |
|---|---|---|
| 1 | Collect demand forecasts | How many resources, and when and where? |
| 2 | Devise build and allocation plans | What is the best way to meet the demand with more supply? |
| 3 | Review and approve the plan | Is the forecast reasonable, and does it match the budget? |
| 4 | Deploy and configure resources | Which services use the resources that arrived? |

Step 1 uses the best data of today to plan into the future. It covers several quarters to
years (Ch. 18, p. 250).

**The cycle never ends.** Assumptions change. Deployments slip. Budgets shrink. Each revision
of the plan propagates into every later quarter (Ch. 18, p. 251). A shortfall in this quarter
must be recovered in a future quarter.

**Traditional planning treats demand as the driver, and shapes supply by hand for each change**
(Ch. 18, p. 251). That last clause is the defect. The hand work is the part that does not
scale.

---

## 2. Why the traditional plan fails

The chapter names two failure properties. Learn both names.

### Brittle by nature (Ch. 18, p. 251)

Any small change can disrupt the allocation plan. SRE lists four examples.

1. A service becomes less efficient, and it needs more resources for the same demand.
2. Customer adoption rises, so the projected demand rises.
3. The delivery date of a new cluster slips.
4. A product decision changes the footprint of the service.

**A minor change forces a cross-check of the whole plan. A larger change forces a rewrite**
(Ch. 18, p. 251). A delivery slip in one cluster can affect the redundancy or the latency
requirement of several services. The allocations in other clusters must rise to cover the slip.

**Signal that you have a brittle plan:** one input moved by a small amount, and a person had to
read every row again. See `capacity-defect-catalog.md`, entry C-37.

### Laborious and imprecise (Ch. 18, p. 252–253)

The data collection for the forecast is slow and error-prone (Ch. 18, p. 252).

**Resources are not fungible.** SRE gives the case directly. A latency requirement can bind a
service to the same continent as the user. Then extra resources in North America do not relieve
a shortfall in Asia (Ch. 18, p. 252).

**Bin packing by hand is the wrong tool.** Bin packing is an NP-hard problem, and it is hard for
a person to compute by hand (Ch. 18, p. 252). The result is a large expenditure of human effort
for a packing that is approximate at best. No known bounds on an optimal solution exist
(Ch. 18, p. 252–253).

The chapter also names the tooling problem. Spreadsheets have scalability problems and limited
error checking. Data becomes stale. Change tracking is hard. Teams then simplify their real
requirements to keep the problem tractable (Ch. 18, p. 252).

**The reasons are lost in transit.** A capacity request usually arrives as an inflexible demand.
"X cores in cluster Y." The reasons for X and for Y are lost by the time the request reaches
a human. Every degree of freedom around them is lost too (Ch. 18, p. 252). See `capacity-defect-catalog.md`,
entry C-38.

Hedge that the book keeps: bin packing is still far from a solved problem, because some types
are still considered NP-hard. Today's algorithms can solve to a known optimal solution
(Ch. 18, p. 253).

---

## 3. Encode the intent, not the allocation

**Intent is the rationale for how a service owner wants to run their service** (Ch. 18, p. 253).

The method has two parts (Ch. 18, p. 253).

1. Encode the dependencies and the parameters of the needs of the service in a program.
2. Generate the allocation plan from that encoding.

When demand, supply, or a requirement changes, you generate a new plan. The new plan is the new
best distribution of resources (Ch. 18, p. 253).

Move from a concrete demand to the reason behind it. That move often needs several layers of
abstraction (Ch. 18, p. 253).

### The four levels of intent

| Level | The statement | Degrees of freedom | Use it when |
|---|---|---|---|
| 1 | "50 cores in clusters X, Y, and Z for service Foo" | None | A hard placement rule exists |
| 2 | "A 50-core footprint in any 3 clusters in region YYY" | Location | The region is fixed |
| 3 | "Meet the demand of Foo in each region, with N+2 redundancy" | Location and size | **Default.** SRE reports the best wins here |
| 4 | "Run service Foo at 5 nines of reliability" | Location, size, and redundancy | A particularly sophisticated service |

The table restates the chain of abstraction on Ch. 18, p. 253–254. `../SKILL.md`, Table L holds
the same four rows.

**Level 3 is the default target.** In Google's experience, services tend to achieve the best
wins as they cross to step 3 (Ch. 18, p. 254). Level 3 gives good degrees of freedom, and the
consequence of a refusal is stated in terms a person understands.

**Level 4 states the consequence directly.** If the requirement is not met, reliability suffers
(Ch. 18, p. 254). Level 4 also carries the largest flexibility. N+2 may not be optimal for that
service, and another deployment plan may fit better (Ch. 18, p. 254). Level 4 is the level where
the plan meets the SLO (SRE Ch. 4, p. 63) and the error budget (SRE Ch. 3, p. 59).

**Support all four levels together.** SRE states the ideal plainly. All levels of intent should
be supported together, and a service benefits more as it moves from implementation toward intent
(Ch. 18, p. 254).

**Signal that you are stuck at level 1:** the request names a cluster and a core count. No row
of the plan records the demand it serves.

---

## 4. The three precursors to intent

You cannot capture intent without three inputs (Ch. 18, p. 254–256).

### Dependencies (Ch. 18, p. 255)

Production dependencies constrain placement. The chapter gives one example. A user-facing
service Foo depends on Bar, an infrastructure storage service. Foo states that Bar must sit
within 30 milliseconds of network latency of Foo. That requirement decides where both services
can run.

**Dependencies are nested.** Bar depends on Baz, a lower-level distributed storage service, and
on Qux, an application management service. So the placement of Foo depends on the placement of
Bar, Baz, and Qux (Ch. 18, p. 255). A set of production dependencies can be shared, and each
sharer can state a different intent.

### Performance metrics (Ch. 18, p. 255)

Demand for one service produces demand for other services. The dependency chain gives the shape
of the bin packing problem. It does not give the resource usage.

**Performance metrics are the glue between dependencies** (Ch. 18, p. 255). They convert one or
more higher-level resource types into one or more lower-level resource types. They answer two
questions. How many compute resources does Foo need for N user queries? How many Mbps does Bar
receive for every N queries of Foo?

**Load testing and resource usage monitoring produce these metrics** (Ch. 18, p. 255). This is
the point where the load test pays for itself. Read `04-load-testing.md` for a test that
produces a number you can trust. Read `01-defining-capacity.md` for the driving variable and the
following variable that the test measures.

**Do not express the metric only in queries per second.** A request rate is a moving target.
Capacity belongs in available resources such as CPU cores (SRE Ch. 21, p. 297–298). See
`capacity-defect-catalog.md`, entry C-36.

### Prioritization (Ch. 18, p. 255–256)

Resource constraints force trade-offs. Ask one question. Which requirements do you sacrifice
when the capacity is insufficient?

The chapter gives two examples. N+2 redundancy for Foo may matter more than N+1 redundancy for
Bar. The launch of feature X may matter less than N+0 redundancy for Baz (Ch. 18, p. 255).

**Intent-driven planning forces these decisions to be made transparently, openly, and
consistently** (Ch. 18, p. 256). The trade-offs exist either way. Without a stated
prioritization, the decision is ad hoc and opaque to the service owners (Ch. 18, p. 256).
Prioritization can be as granular or as coarse as you need.

**Signal that the prioritization is missing:** the plan states what you get, and no line states
what you lose first. See `capacity-defect-catalog.md`, entry C-39.

---

## 5. The Auxon case study

Auxon is Google's implementation of intent-based capacity planning and resource allocation
(Ch. 18, p. 256). A small group of software engineers and one technical program manager inside
SRE built it over two years. It plans many millions of dollars of machine resources, and it
became a critical component for several major divisions (Ch. 18, p. 256).

Auxon collects requirements through a user configuration language or through a programmatic API
(Ch. 18, p. 256). A requirement reads like "My service must be N + 2 per continent". The
requirements are represented internally as a giant mixed-integer or linear program. Auxon solves
that program and uses the resulting bin packing solution to form the allocation plan
(Ch. 18, p. 256).

### The components (Figure 18-1, Ch. 18, p. 257–258)

| Component | What it holds | Role in the program |
|---|---|---|
| Performance Data | How the service scales, per unit of demand | The conversion between resource types |
| Per-Service Demand Forecast Data | The usage trend for a forecast demand signal | The demand side |
| Resource Supply | The base resources available at a future time | The upper bound |
| Resource Pricing | The cost of base resources, which varies by facility | The objective to minimize |
| Intent Config | What a service is, and how services relate | The human-readable wiring layer |
| Configuration Language Engine | Light sanity checking, and a protocol buffer | The gateway from human intent to a machine request |
| Auxon Solver | The mixed-integer or linear program | The brain, run in parallel over many machines |
| Allocation Plan | Which resources go to which services, and where | The output |

Some services derive their demand from a forecast of queries per second by continent. Not every
service has one. A storage service derives its demand purely from the services that depend on it
(Ch. 18, p. 257).

**The output must also report failure.** The Allocation Plan includes information about any
requirement that could not be satisfied (Ch. 18, p. 258). The chapter names two causes. A lack
of resources, and competing requirements that were too strict.

**Read this rule as a specification for your own plan.** A plan that reports only the satisfied
requirements hides the shortfall. The shortfall then arrives as an incident.

---

## 6. Approximation

SRE states the rule at the head of the section. Do not focus on perfection and purity of
solution, in particular when the bounds of the problem are not well known. Launch and iterate
(Ch. 18, p. 259).

**Approximate behind an interface that you can replace.** This is the discipline that makes
approximation safe. The chapter states that the whole solver interface was abstracted inside
Auxon, so the solver internals could be replaced at a later date (Ch. 18, p. 259).

**Use fuzzy requirements as a reason to make the software general and modular** (Ch. 18, p. 260).
Some uncertainty need not stop the work.

**Keep a real implementation next to the general design.** A general solution needs a
real-world specific implementation. That implementation demonstrates the utility of the design
(Ch. 18, p. 260).

| Question | A safe approximation | An unsafe approximation |
|---|---|---|
| Can you replace the algorithm later? | The interface hides it | Callers depend on its internals |
| Does the output state its own limits? | It lists the unsatisfied requirements | It reports one number |
| Does a real service use it today? | Yes, one team runs on it | It waits for the full design |

---

## 7. War stories from Chapter 18

**The Stupid Solver (p. 259–260).** Linear programming was uncharted territory for the Auxon
team, and linear programming looked central to the product. The team did not stall. It built a
deliberately simplified solver that applied simple heuristics to arrange services against the
stated requirements. The team called it the "Stupid Solver". It would never yield a truly
optimal solution. It did prove that the vision for Auxon was achievable. The team hid the whole
solver interface inside Auxon. So the later replacement with a unified linear programming model
was a simple operation. The lesson is not "approximate". The lesson is "approximate behind an
interface you can replace".

**The agnostic Allocation Plan (p. 260).** One aim was to let automation systems enact the
Allocation Plan directly on production, and assign, resize, or stop services. At the time the
automation world was in flux, with a large variety of approaches in use. The team refused to
design a unique integration per tool. It shaped the Allocation Plan to be universally useful, so
each automation system could build its own integration point. That agnostic shape became the key
to onboarding new customers. A team could adopt Auxon without a change of its turnup automation,
its forecasting tool, or its performance data tool.

**The wrong first customers (p. 262).** Larger teams already had home-grown capacity planning
solutions that worked passably well. Those teams did not experience enough pain to try a new
tool, in particular an alpha release with rough edges. The initial versions of Auxon
deliberately targeted teams with no capacity planning process at all. Those teams had to invest
configuration effort either way, so they were interested in the newest tool. The early successes
demonstrated the utility of the project and turned those customers into advocates.

**The Business Area case study (p. 262).** Auxon onboarded one of Google's Business Areas. The
team wrote a case study of the process and compared the results before and after. The time
savings and the reduction of human toil alone gave other teams a large incentive to try Auxon. A
measured before-and-after comparison moved adoption further than a presentation or an
announcement email (Ch. 18, p. 261–262).

**"I have a design doc. Why do we need requirements?" (p. 265).** An engineer on an early SRE
software development project asked that question. SRE reports it as the conventional approach to
software inside SRE. SREs often have strong coding skills and no experience of a product team
that handles customer feature requests. The remedy is a partnership with engineers, technical
program managers, or product managers who know user-facing development.

---

## 8. The recompute rule

**A plan that you cannot recompute is a plan that you will not update.**

The traditional plan is toil. SRE gives six attributes of toil. Manual, repetitive, automatable,
tactical, without enduring value, and O(n) with service growth (SRE Ch. 5, p. 76). A quarterly
plan that one person rebuilds by hand matches five of the six.

### Name the inputs, then name the trigger

Store these four inputs (Ch. 18, p. 257–258). Then regenerate the plan when one of them moves.

| Input | Regenerate the plan when |
|---|---|
| Performance data | A release changes the resource cost of a request |
| Demand forecast | Adoption, a launch, or a seasonal peak moves the forecast |
| Resource supply | A delivery date slips, or a region gains or loses capacity |
| Resource pricing | The cost of a machine or a facility changes |

Nygard states the operational half of the same rule. Monitor capacity continuously, because each
release can affect scalability and performance (Release It! Sec. 8.6, p. 174).

**Modern:** an autoscaler or a Terraform plan is one consumer of the allocation plan. It is not
the plan. Neither book states this. An autoscaler reacts to today's load. The plan states what
you buy for next quarter, and what you sacrifice when it does not arrive.

### The recompute checklist

1. Write the intent at level 3, or state why the service needs level 1 or 2.
2. Store the four inputs in files that a script reads.
3. Write the script that produces the allocation from those files.
4. Record the prioritization order for a shortfall.
5. Make the output list every requirement that it could not satisfy, and the reason.
6. State the trigger that runs the script again.

**A plan fails this checklist when step 3 is a person.**

---

## 9. What to write in the Capacity Record

`../SKILL.md` defines the Capacity Record. This file fills the `### Intent` block.

```md
### Intent
- Level (1 to 4): [3]
- The requirement, stated without an allocation: []
- Dependencies and their placement constraints: []
- Performance metrics, and the load test that produced them: []
- Inputs that regenerate this plan: [performance data, forecast, supply, pricing]
- Prioritization under a shortfall: [what is sacrificed, in order]
- Requirements that this plan could not satisfy, and why: []
- Recompute trigger: []
```

---

## 10. Proportionality

This file does not apply to every change.

| Situation | Apply this file? |
|---|---|
| A quarterly or annual resource plan | Yes, in full |
| A launch that adds a new region or a new dependency | Yes, sections 3, 4, and 8 |
| A service with one machine and no forecast | No. Record the constraint and stop |
| A bug fix with no change in resource cost | No |
| A load test that produced a new performance metric | Yes, section 4 only |

"Not applicable. This change adds no demand and moves no resource." is a valid result. That
result is better than an invented plan.

**Do not target 100% adoption.** SRE names that pitfall directly, and reports diminishing returns
on the last mile (Ch. 18, p. 263).

---

## Cross-references

| Read this | For |
|---|---|
| `01-defining-capacity.md` | The constraint, the driving variable, and the knee |
| `04-load-testing.md` | The test that produces the performance metric of section 4 |
| `06-load-balancing-and-utilization.md` | Where the allocation meets the running fleet |
| `capacity-defect-catalog.md` | Entries C-36, C-37, C-38, C-39, C-40, and C-47 |
| `../SKILL.md`, Table L | The same four levels of intent, in the gate form |
