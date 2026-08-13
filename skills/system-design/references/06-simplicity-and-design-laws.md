# Simplicity and the Laws of Software Design

**Source:** *Code Simplicity: The Science of Software Development* (Max Kanat-Alexander,
O'Reilly 2012).

This is the counterweight to everything else in this skill. The rest of `system-design`
tells you how to build a system that handles scale; this reference tells you when **not**
to, and it is the more frequently needed of the two.

> Over-engineering is a real disease of many engineers, who delight in design purity and
> ignore trade-offs. They are often unaware of the compounding costs of over-engineered
> systems, and many companies pay a high price for that ignorance. *(Xu, Ch. 3)*

---

## The six laws

**1. The purpose of software is to help people.**

Everything else is subordinate. A design that serves elegance, novelty, or résumés rather
than users has failed regardless of its technical properties.

The goals of software design follow from it:
1. Write software that is as helpful as possible.
2. Allow the software to *continue* to be as helpful as possible.
3. Design systems that can be created and maintained as easily as possible by their
   programmers — so they can continue to be helpful.

**2. The Equation of Software Design.**

```
D = (Vn + Vf) / (Ei + Em)
```

The desirability of a change is directly proportional to its value now plus its future
value, and inversely proportional to the effort of implementation plus the effort of
maintenance.

**As time goes on, this equation reduces**, because maintenance effort accumulates
indefinitely while implementation effort is paid once. The conclusion:

> **It is more important to reduce the effort of maintenance than it is to reduce the effort
> of implementation.**

A design that is fast to build and expensive to operate is a **bad design**, even if it
ships on time. This is the single most useful sentence in the book for architecture reviews.

**3. The Law of Change.** *The longer your program exists, the more probable it is that any
piece of it will have to change.*

Everything will change. This is an argument for making change cheap — not for predicting
which change will come.

**4. The Law of Defect Probability.** *The chance of introducing a defect into your program
is proportional to the size of the changes you make to it.*

Small changes are safer than large ones. This is the argument for incremental delivery, and
against the big-bang rewrite and the ten-file refactor bundled into a feature PR.

**5. The Law of Simplicity.** *The ease of maintenance of any piece of software is
proportional to the simplicity of its individual pieces.*

Combined with law 2: **maintenance effort is proportional to system complexity, and system
complexity comes from the complexity of its individual pieces.** Every component you add to
an architecture is a permanent tax on everyone who operates it.

**6. The Law of Testing.** *The degree to which you know how your software behaves is the
degree to which you have accurately tested it.*

Corollary: **unless you've tried it, you don't know that it works.** Applied to design: an
untested failover procedure is not a failover procedure, and an unmeasured capacity claim
is a hope.

---

## The three flaws

The three mistakes designers make in coping with the Law of Change:

**1. Writing code that isn't needed.**
> Don't write code until you actually need it, and remove any code that isn't being used.

In architecture: the service with one caller, the queue with no backpressure requirement,
the abstraction layer over the one database you will ever use, the feature flag system
built before the first flag.

**2. Not making the code easy to change.**
The opposite failure. Hardcoded assumptions, no seams, no way to modify behavior without
touching everything. In architecture: schemas with no evolution path, APIs with no version
strategy, deployments that must happen in a specific order.

**3. Being too generic.**
> Be only as generic as you know you need to be right now.

The plugin framework with one plugin. The configuration system for values that never change.
The generic pipeline that handles one job type. Genericity is complexity spent on a future
that may not arrive.

**All three are avoided by incremental development and design** — building in small pieces,
each of which is complete and useful.

---

## Do not predict the future

This is the sharpest edge of the book, and the one most directly applicable to system design:

> **There are some things about the future that you do not know.**
>
> **The most common and disastrous error that programmers make is predicting something
> about the future when in fact they cannot know.**
>
> **You are safest if you don't attempt to predict the future at all, and instead make all
> your design decisions based on immediately known present-time information.**

And:

> **Code should be designed based on what you know now, not on what you think will happen
> in the future.**

The apparent tension with capacity planning resolves cleanly:

- **Legitimate:** "We measured 2,000 QPS and are growing 15% per month, so we will exceed
  single-node capacity in roughly nine months." That is present-time information
  extrapolated with a stated rate.
- **Illegitimate:** "We might go viral, so let's build for a million QPS." That is a
  prediction, and it costs real complexity today for a scenario nobody can bound.

The resolution rule: **design for measured load with a stated growth rate; design the
architecture so the next rung of the ladder is reachable; do not build the next rung until
the number says so.** Which is exactly the intent behind:

> **The best design is the one that allows for the most change in the environment with the
> least change in the software.**

That is *not* the same as building for every hypothetical requirement — it is the opposite.

---

## Quality proportional to lifetime

> **The quality level of your design should be proportional to the length of future time in
> which your system will continue to help people.**

A prototype validating a hypothesis, a service that will run for a decade, and a one-off
migration script deserve genuinely different levels of design effort. Applying
ten-year-system rigor to a two-week experiment is waste; applying prototype rigor to a
payments system is negligence.

This is the principle behind the proportionality table in `SKILL.md` §7.

---

## Detecting over-engineering

> **When your design actually makes things more complex instead of simplifying things,
> you're overengineering.**

**The removal test.** For every component in the architecture, ask: *what breaks if we
remove it, at our current measured numbers?* If the answer is "nothing", remove it and add
it back when a number demands it.

**The justification test.** Every component must trace to either a measured number
(`02-estimation.md`) or a stated requirement. "We might need it" is not a justification.
"Everyone uses one" is not a justification.

**The operator test.** Can someone who did not design this operate it at 3 a.m. with the
runbook? If not, the complexity is not paid for.

**The complexity-source rule.**
> **Often, if something is getting very complex, that means there is an error in the design
> somewhere below the level where the complexity appears.**

When a component requires elaborate handling, the bug is usually not in that component. A
cache invalidation scheme that needs five special cases is usually pointing at a data model
that should not have been denormalized. **Fix the level below before adding machinery at the
level above.**

**The clarifying question.**
> **When presented with complexity, ask: "What problem are you trying to solve?"**

Ask it about your own design too. Frequently the honest answer is "a problem we do not
have."

---

## Sources of complexity

Kanat-Alexander's list of ways you create complexity, mapped to their architectural forms:

| Source | Architectural form |
|---|---|
| Expanding the purpose of your software | Scope creep; the service that grew three unrelated responsibilities |
| Adding programmers to the team | More coordination surface; more services created to avoid coordination |
| Changing things that don't need to be changed | Rewrites and migrations with no measured problem behind them |
| Being locked into bad technologies | The store nobody can operate; the framework with no upgrade path |
| Misunderstanding | Building the wrong thing correctly — prevented by Step 1 of the method |
| Poor design or no design | Accreted architecture; the diagram nobody can draw |
| Reinventing the wheel | The homegrown queue, the homegrown scheduler, the homegrown auth |
| Violating the purpose of your software | Designing for the architecture instead of the user |

**Judging a technology:** survival potential, interoperability, and attention to quality.
Note that popularity is not on the list, and neither is benchmark performance.

---

## Handling complexity you already have

> **To handle complexity in your system, redesign the individual pieces in small steps.**

Not a rewrite. **Rewriting is acceptable only in a very limited set of situations** — and
combined with the Law of Defect Probability (defects scale with change size), a rewrite is
the highest-risk action available. Prefer strangler-style incremental replacement, where
each step is independently valuable and independently reversible.

> **If you run into an unfixable complexity outside of your program, put a wrapper around it
> that is simple for other programmers.**

The anti-corruption layer. Contain what you cannot fix, and give everyone else a simple
interface to it.

> **Never "fix" anything unless it's a problem, and you have evidence showing that the
> problem really exists.**

This applies directly to performance work: optimizing without a profile, sharding without a
saturation measurement, caching without a hit-rate analysis. **Evidence first.**

> **In any particular system, any piece of information should, ideally, exist only once.**

Applied to architecture, this is the argument against dual writes and undisciplined
denormalization. Where you must duplicate for performance, make one copy the source of
truth and **derive** the others — never write both independently (hazard H-32).

---

## The design review questions

Add these to any architecture review:

1. **What problem are you trying to solve?** Is it a problem we actually have, with
   evidence?
2. **What is the maintenance cost** of this design, and who pays it?
3. **What in this design is predicting the future?** Remove it or justify it with a measured
   trend.
4. **What can we remove** and still meet the stated requirements at the measured numbers?
5. **Is the quality level proportional** to how long this will run?
6. **Where is complexity concentrated**, and is there a design error one level below it?
7. **Is any piece of information stored more than once**, and if so which copy is the source
   of truth?
8. **What have we actually tested** versus assumed? (Law of Testing — including the failover
   and restore paths.)

---

## The two sentences

Kanat-Alexander's own summary, which is worth keeping in working memory:

> - **It is more important to reduce the effort of maintenance than it is to reduce the
>   effort of implementation.**
> - **The effort of maintenance is proportional to the complexity of the system.**

Plus the purpose of software — to help people — and, he claims, you could re-derive the
entire science of software design. For system design work, those two sentences settle most
arguments about whether a component belongs in the diagram.
