# Simplicity and the Laws of Software Design

**Source:** *Code Simplicity: The Science of Software Development* (Max Kanat-Alexander,
O'Reilly, 2012).

This reference balances the rest of this skill. The other references show how to build a
system that handles a large load. This reference shows when **not** to build one. You need
this reference more often.

Xu describes the fault it prevents:

> Over-engineering is a real disease of many engineers, who delight in design purity and
> ignore trade-offs. They are often unaware of the compounding costs of over-engineered
> systems, and many companies pay a high price for that ignorance.

---

## The six laws

### Law 1. Software exists to help people.

Every other rule is subordinate to this one. A design that serves elegance, novelty, or a
career instead of users has failed. Its technical properties do not change that result.

Three goals follow:

1. Write software that helps as much as possible.
2. Keep the software helpful over time.
3. Design systems that programmers can build and maintain easily, so that the software stays
   helpful.

### Law 2. The equation of software design.

```
D = (Vn + Vf) / (Ei + Em)
```

How much we want a change (D) is proportional to its value now (Vn) plus its future value
(Vf). It is inversely proportional to the effort to build it (Ei) plus the effort to maintain
it (Em).

**Over time the equation reduces**, because maintenance effort continues without end and
build effort occurs once. One conclusion remains:

> **Reducing the effort of maintenance matters more than reducing the effort of
> implementation.**

A design that is fast to build and expensive to operate is a **bad design**. It is still a
bad design when it ships on time. This is the most useful sentence in the book for an
architecture review.

### Law 3. The Law of Change.

*The longer your program exists, the more probable it is that any piece of it will have to
change.*

Everything changes. So make change cheap. This law is not a reason to predict which change
arrives.

### Law 4. The Law of Defect Probability.

*The chance of introducing a defect into your program is proportional to the size of the
changes you make to it.*

Small changes are safer than large ones. This law argues for incremental delivery. It argues
against a full rewrite. It argues against a refactor of ten files inside a feature change.

### Law 5. The Law of Simplicity.

*The ease of maintenance of any piece of software is proportional to the simplicity of its
individual pieces.*

Combine this law with Law 2. Maintenance effort is proportional to system complexity. System
complexity comes from the complexity of each part. **So every component that you add to an
architecture is a permanent cost for everyone who operates it.**

### Law 6. The Law of Testing.

*The degree to which you know how your software behaves is the degree to which you have
accurately tested it.*

The rule that follows: **unless you have tried it, you do not know that it works.**

Applied to a design: a failover procedure that nobody tested is not a failover procedure. A
capacity claim that nobody measured is a hope.

---

## The three flaws

These are the three mistakes that designers make when they respond to the Law of Change.

### Flaw 1. Writing code that is not needed.

> Don't write code until you actually need it, and remove any code that isn't being used.

In an architecture, this flaw takes these forms:

- A service with one caller.
- A queue with no requirement for backpressure.
- An abstraction over the one database that you will ever use.
- A feature-flag system built before the first flag.

### Flaw 2. Not making the code easy to change.

This is the opposite failure. The code contains fixed assumptions, no seams, and no way to
change behavior without touching everything.

In an architecture, this flaw looks like: a schema with no path to change it, an API with no
version plan, or a deploy that must happen in one exact order.

### Flaw 3. Being too generic.

> Be only as generic as you know you need to be right now.

Examples: a plugin framework with one plugin, a configuration system for values that never
change, or a generic pipeline that runs one job type.

Generality is complexity that you spend on a future that may not arrive.

**Incremental development prevents all three flaws.** Build in small parts. Make each part
complete and useful on its own.

---

## Do not predict the future

This is the sharpest rule in the book. It applies directly to system design.

> **There are some things about the future that you do not know.**
>
> **The most common and disastrous error that programmers make is predicting something about
> the future when in fact they cannot know.**
>
> **You are safest if you don't attempt to predict the future at all, and instead make all
> your design decisions based on immediately known present-time information.**

And:

> **Code should be designed based on what you know now, not on what you think will happen in
> the future.**

This rule appears to conflict with capacity planning. It does not. Compare two statements.

**Permitted.** "We measured 2,000 QPS. The load grows 15% each month. So we exceed the
capacity of one node in about nine months." This statement uses present information and a
stated rate.

**Not permitted.** "We might become popular, so let us build for one million QPS." This
statement is a prediction. It costs real complexity today for a case that nobody can bound.

**The rule that resolves the conflict:**

1. Design for the load that you measure, with a stated growth rate.
2. Design the architecture so that the next step of the scaling ladder is reachable.
3. Do not build that next step until a number requires it.

Kanat-Alexander states the same idea from the other direction:

> **The best design is the one that allows for the most change in the environment with the
> least change in the software.**

That is not a design that covers every possible future requirement. It is the opposite.

---

## Match the design effort to the lifetime

> **The quality level of your design should be proportional to the length of future time in
> which your system will continue to help people.**

Three systems deserve three different levels of effort:

| System | Effort |
|---|---|
| A prototype that tests one hypothesis | Low |
| A service that runs for a decade | High |
| A migration script that runs once | Low |

Applying ten-year rigor to a two-week experiment is waste. Applying prototype rigor to a
payment system is negligence.

This law produces the table in `../SKILL.md`, Section 7.

---

## How to detect an architecture that is too large

> **When your design actually makes things more complex instead of simplifying things, you're
> overengineering.**

Apply four tests.

### Test 1. The removal test

For each component in the architecture, ask this question. *What breaks if we remove this
component, at the load that we measure today?*

If the answer is "nothing", remove it. Add it again when a number requires it.

### Test 2. The justification test

Every component must connect to one of two things: a measured number from `02-estimation.md`,
or a stated requirement.

"We might need it" is not a justification. "Everyone uses one" is not a justification.

### Test 3. The operator test

Can a person who did not design the system operate it at 3 a.m. with the runbook?

If not, nobody has paid for the complexity.

### Test 4. The complexity-source test

> **Often, if something is getting very complex, that means there is an error in the design
> somewhere below the level where the complexity appears.**

When one component needs elaborate handling, the fault is usually elsewhere. Example: a cache
invalidation scheme needs five special cases. That scheme usually points at a data model that
should not have duplicated the data.

**Correct the level below before you add machinery at the level above.**

### The question to ask about any complexity

> **When presented with complexity, ask: "What problem are you trying to solve?"**

Ask this question about your own design. The honest answer is often "a problem that we do not
have".

---

## Where complexity comes from

Kanat-Alexander lists the sources. The right column gives the form that each source takes in
an architecture.

| Source | Form in an architecture |
|---|---|
| Expanding the purpose of the software | Scope growth. One service acquires three unrelated jobs. |
| Adding programmers to the team | More coordination. Teams create services to avoid coordination. |
| Changing things that do not need to change | Rewrites and migrations with no measured problem behind them |
| Being locked into bad technologies | A database that nobody can operate. A framework with no upgrade path. |
| Misunderstanding | Building the wrong thing correctly. Step 1 of the method prevents this. |
| Poor design, or no design | An architecture that grew by accident. Nobody can draw the diagram. |
| Reinventing the wheel | A queue, a scheduler, or an authentication system built in-house |
| Violating the purpose of the software | Designing for the architecture instead of the user |

**How to judge a technology:** survival potential, interoperability, and attention to quality.
Popularity is not on the list. Benchmark performance is not on the list.

---

## How to reduce complexity that already exists

> **To handle complexity in your system, redesign the individual pieces in small steps.**

This is not a rewrite. **Rewriting is acceptable only in a very limited set of situations.**
Combine that rule with the Law of Defect Probability, where defects grow with the size of the
change. A rewrite is then the highest-risk action available.

Prefer incremental replacement. Make each step valuable on its own. Make each step reversible.

> **If you run into an unfixable complexity outside of your program, put a wrapper around it
> that is simple for other programmers.**

Contain what you cannot correct. Give everyone else a simple interface to it.

> **Never "fix" anything unless it's a problem, and you have evidence showing that the problem
> really exists.**

This rule applies directly to performance work. Do not optimize without a profile. Do not
partition without a saturation measurement. Do not add a cache without a hit-rate analysis.
**Get the evidence first.**

> **In any particular system, any piece of information should, ideally, exist only once.**

In an architecture, this rule argues against dual writes and against duplicating data without
control. Where you must duplicate data for performance, make one copy authoritative and
**derive** the others. Never write both copies independently. That is hazard H-32.

---

## Questions for a design review

Add these eight questions to any architecture review.

1. What problem are you solving? Do we have that problem? What is the evidence?
2. What does this design cost to maintain? Who pays that cost?
3. Which part of this design is a prediction? Remove it, or connect it to a measured trend.
4. What can we remove and still meet the requirements at the measured load?
5. Does the design effort match the lifetime of the system?
6. Where is the complexity concentrated? Is there a design error one level below it?
7. Does any piece of information exist more than once? Which copy is authoritative?
8. What have we tested, and what have we assumed? This question covers the failover path and
   the restore path, not only the feature code.

---

## The two sentences

Kanat-Alexander's own summary. Keep both sentences in mind during a design.

> - **It is more important to reduce the effort of maintenance than it is to reduce the effort
>   of implementation.**
> - **The effort of maintenance is proportional to the complexity of the system.**

Add the purpose of software, which is to help people. He claims that these three statements
permit you to derive the whole science of software design.

For system design work, the two sentences settle most arguments about whether a component
belongs in the diagram.
