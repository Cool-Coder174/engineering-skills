# Defining and Structuring the Problem

**Sources:** *The Minto Pyramid Principle: Logic in Writing, Thinking and Problem Solving* —
Barbara Minto (Pearson, 3rd ed.), Ch. 8 "Defining the Problem", Ch. 9 "Structuring the Analysis of
the Problem", App. A "Problem Solving in Structureless Situations". Content that the book predates
carries **Modern**.

This file holds the first stage of the consultation. You define the problem. You structure the
analysis. Only after both stages do you gather evidence. **A problem is a gap.** Minto describes a
gap between the result you get now and the result you would rather have. "That gap is the
problem." Minto Ch. 8, "Laying out the Elements", p. 123. The gap has a history. It resulted from
an existing situation and developed in response to a set of circumstances. Minto Ch. 8, p. 121.
**Minto writes about business memos. The reader of this file works on software.** Every paragraph
marked **In engineering** is a translation, not a report of what Minto wrote.

---

## 1. The four elements

**Specify four elements before you say that you defined the problem.** Minto Ch. 8, "Lay Out the
Problem", p. 127. The Solution then tells you how to get from R1 to R2. Minto Ch. 8, p. 121.

| Element | The question it answers | In engineering (translation) |
|---|---|---|
| Starting Point / Opening Scene | What is going on? | The structure or process that the decision sits inside, drawn from the repository |
| Disturbing Event | What changed? | The metric, the incident, the deprecation notice, or the new requirement |
| R1, the Undesired Result | What do we not like about it? | One sentence that names the present bad result |
| R2, the Desired Result | What do we want instead? | The threshold table. One row per desired result |

Minto states the first three questions as one set, and she gates the next stage on them. Minto
Ch. 8, "Laying out the Elements", p. 123. Only after you answer them can you determine the
Question and search for the Solution. **Signal that you are not ready.** You can name the
candidate options and you cannot state R2. Stop at section 1.4.

### 1.1 The Starting Point / Opening Scene

**The Opening Scene is a specific place at a particular moment in time.** Minto explains it in
dramatic terms. You sit in a dark theatre. The curtain parts. You see a set. Minto Ch. 8, "Lay Out
the Problem", p. 127. Four rules govern it, all from Minto Ch. 8, "The Starting Point/Opening
Scene", p. 128.

1. The Opening Scene is a structure or a process that you can visualize.
2. Sketch it at the general knowledge level of a normal reader of *Fortune* or *Business Week*.
3. Apply the friend test. Ask what a friend must see to understand the story.
4. Keep the visualization simple and the description short. Expand the prose later.

**The Opening Scene decides the analysis.** The Solution generally derives from a change to the
structure that you drew, and the causes of the gap generally lie in its activities. Minto Ch. 8,
"Laying out the Elements", p. 123. So you decompose the scene, not the symptom. **In
engineering.** Read the manifest, the lockfile, the container file, the CI file, and the code
paths that the decision touches. Draw one diagram, and give every claim in it a file path.
**Modern.** Minto has no concept of a lockfile. She supplies the rule that produces the practice.
See `advice-defect-catalog.md`, X-10.

### 1.2 The Disturbing Event

**The Disturbing Event threatens the stable situation of the Opening Scene, and it triggers R1.**
It can be an event that already happened, or one that could happen or would be likely to happen.
Minto Ch. 8, "The Disturbing Event", p. 129.

| Type | Minto's examples | Engineering instance (translation) |
|---|---|---|
| External | A new competitor. A conversion to a new technology. A shift in government or customer policy | A vendor deprecation notice. A licence change. A new regulation |
| Internal | An added business process. A new computer system. A new market. A redirected product line | A new service. A migration in progress. A traffic tier the team accepted |
| Recently Recognized | Lagging performance. Sub-par operating results. Research that implies a shift in customer attitude | A p99 latency trend. A rising error budget burn. A failing build |

**Never manufacture a Disturbing Event.** Minto anticipates the case where nobody gave you enough
information to identify the trigger. Her instruction is to move directly to R1. Minto Ch. 8,
p. 129. **Signal.** The team cannot say what changed. Record it as unknown, and state instead what
the reader dislikes in the structure.

### 1.3 R1, the Undesired Result

**R1 is the problem that your reader tries to solve, or is likely to face, or the opportunity he
could embrace.** Minto Ch. 8, "R1 (Undesired Result)", p. 129. State it as briefly as possible in
the diagram. One disturbance can produce more than one R1. Minto Ch. 8, p. 130. Keep her hedge.
She does not say that it always does. Minto lists what the Disturbing Event will likely have done.
It may have revealed an unrecognized opportunity. More likely it affected the processes adversely,
disrupted one area, or challenged basic assumptions about customers, markets, or technology. Minto
Ch. 8, pp. 129–130.

**In engineering.** Write R1 in one sentence, with the observation that supports it, and mark it
*inferred* when you inferred it. The observation is a metric, an incident record, a deprecation
notice, or a failed build.

### 1.4 R2, the Desired Result

**State R2 as specifically and quantifiably as you can, so that you can tell when you achieved
it.** Minto Ch. 8, "R2 (Desired Result)", p. 130. R2 takes end-product terms. The wording carries
a specific number, or it indicates a specific end state. Three of her examples are these. Meet
year-end growth goals. Reduce time to market by 1/3. Have sufficient capacity to cope with
projected demand.

**Why you cannot choose a Solution without R2.** Without an end-product description of the Desired
Result, you cannot easily choose between the possible Solutions that your thinking generates.
Minto Ch. 8, p. 130. Appendix B is sharper. So-called alternatives arise when R2 is ambiguously
stated, because you cannot judge that you hold a solution when you see one. Minto App. B, "Dealing
with Alternative Solutions", p. 225. **Compare each option to R2, never to the other options.**
The comparison table therefore carries the threshold in its header row. Minto App. B, p. 225. See
`09-the-tradeoff-evaluation.md`.

**Fallback rule.** If you cannot state R2 as a specific end product, write the general state that
you want when the problem is solved. Determining the specific R2 is then the first step of your
problem solving. Minto Ch. 8, p. 130. **In engineering.** The R2 table becomes the deliverable,
and the comparison waits.

**Modern.** Do not run a benchmark to discover what good means. See `advice-defect-catalog.md`,
X-05.

**The R2 table is a translation.** Minto supplies the wording rule, not the table. One row per
desired result. Four columns. The statement, the threshold or end state, the observation, and the
person who set it.

| R2 row | Not admissible | Admissible |
|---|---|---|
| Latency | "Better performance" | "p99 write latency under 50 ms at 10k writes/s" |
| Licence | "The licence must be acceptable" | "The licence permits distribution in a binary" |
| Exit | "Avoid lock-in" | "A documented export exists in an open format" |

**The elements may change. The relationships never do.** Once you gather data you can refine and
restate R1 and R2, and the relationship between the parts always prevails. Minto Ch. 8,
pp. 130–131. She calls the set a rough but recognizable scaffolding. It shows the gaps in your own
understanding, and you then wrap the introduction around it.

---

## 2. The Question falls out of the four elements

**The Question depends on how far the reader already progressed.** Minto Ch. 8, "Look for the
Question", p. 131. She names the failure directly. A big error is that the writer does not specify
to himself whether the reader already took action. Exhibit 34 maps seven problem situations to
Situation, Complication, and Question. Minto Ch. 8, pp. 131–132.

| # | Where the reader stands | S | C | Q | Engineering instance |
|---|---|---|---|---|---|
| 1 | They do not know how to get from R1 to R2 | Situation | R1, R2 | How do we get from R1 to R2? | "Our writes are too slow. What do we do?" |
| 2 | They think they know, and are not certain | Situation, R1, R2 | Solution | Is it the right solution? | "We plan to move to Postgres. Is that right?" |
| 3 | They know, and cannot implement it | Situation, R1, R2 | Solution | How do we implement the solution? | "We chose Postgres. How do we migrate?" |
| 4 | They implemented a solution and it failed | Situation, R1, R2, Solution | The solution did not work | What should we do? | "We added the cache and p99 did not move." |
| 5 | They hold several solutions and cannot pick one | Situation, R1, R2 | We have alternative ways | Which is the best alternative? | "Postgres or DynamoDB?" |
| 6 | They know R1 and cannot articulate R2 | Situation, R1 | We must change, and we lack a target | What should our objectives be? | "The platform feels wrong. Fix it." |
| 7 | They know R2 and doubt that they are at R1 | Situation, R2 | Not sure whether we are at R1 | Do we have a problem? | "Are we behind on our stack?" |

Minto groups them. Situations 1 to 3 are the most common circumstances. Situations 4 and 5 are
variations on them. Situations 6 and 7 are possible and not common. She names situation 7 as the
typical benchmarking study. Minto Ch. 8, pp. 131–132.

**Situation 5 is the framework-against-framework case, and this skill meets it most often.
Modern.** Situation 6 produces a different deliverable. The reader cannot state R2, so the R2
table is the work. Do not score options.

**One beginning Question per document.** The related rule sits in another chapter. Minto Ch. 4,
pp. 48–49. "Should we migrate, and if so to what?" collapses to "How should we migrate?" See
`06-the-scqa-introduction.md` and `advice-defect-catalog.md`, X-07.

---

## 3. Convert the framework into an introduction

**Read the diagram from left to right and down.** Minto Ch. 8, "Converting to an Introduction",
p. 124. **The last thing the reader knows is always the Complication.** Everything that the reader
already accepts moves into the Situation. That includes prior R1 values, prior R2 values, and
Solutions that failed. Minto Ch. 8, p. 124.

**Alternatives always belong in the Complication.** You should ordinarily not raise them unless
the reader knows them in advance. Minto Ch. 8, "Move to the Introduction", p. 135. **Never raise
an alternative in order to reject it.** Minto forbids the Key Line that reads "Way A is no good,
Way B is no good, therefore do Way C". The merit of C is that C solves the problem. Minto Ch. 8,
pp. 135–136. See `advice-defect-catalog.md`, X-11.

**Name the plural noun that governs the Key Line.** Minto separates two cases. Minto Ch. 8, "Move
to the Introduction", p. 133. You change a system already in operation, so the plural noun is
**changes**. You tell somebody how to do something new, so the plural noun is **steps**.

**Triple layers.** A problem can carry a history of two failed solutions. Minto subscripts them
R1-a, R2-a, Solution-a, R1-b, R2-b, Solution-b, R1-c. Exhibit 33, Minto Ch. 8, pp. 125–126. All
earlier layers collapse into the Situation. Only the newest R1 is the Complication.

**The five-step process for the framework.** Minto Ch. 8, "Converting to an Introduction", p. 127.
Steps 4 and 5 are the verification gate, and they are also the procedure for the review of a
document that somebody passed to you.

1. State the basic parts of the problem.
2. Identify where you stand in terms of the Solution.
3. Determine the appropriate Question.
4. Check that the introduction reflects the problem definition.
5. Check that the pyramid answers the Question.

---

## 4. Two instruments, and the work each one does

Chapter 9 separates two aids that people group together under "analytical techniques". You must
know the difference to use the right one in the right place. Minto Ch. 9, "Starting with the
Data", p. 142.

| Instrument | It produces | Sequential-analysis step | Source |
|---|---|---|---|
| Diagnostic framework | **Questions** about where the problem lies and why it exists | 2 and 3 | Minto Ch. 9, "Devising Diagnostic Frameworks", p. 143 |
| Logic tree | **Solutions**, as alternative ways to solve the problem | 4 and 5 | Minto Ch. 9, "Developing Logic Trees", p. 156 |
| Decision tree, PERT diagram | The **need for action** only. It is not a diagnostic | Neither | Minto Ch. 9, Exhibit 45, p. 150 |

**The five questions of Sequential Analysis.** Is there a problem? Where does it lie? Why does it
exist? What could we do about it? What should we do about it? Minto Ch. 8, opening, pp. 121–122,
footnoted to Holland, *Sequential Analysis*, McKinsey, 1972. Questions 1 and 2 become the
introduction. Questions 3 to 5 become the pyramid.

### 4.1 Diagnostic frameworks generate questions

**Derive the likely possible reasons from the Opening Scene.** You cannot take them from the air.
You get them when you examine the structure of the area where the problem occurred. Minto Ch. 9,
"Starting with the Data", p. 142. The framework you need is generally implied by the Opening
Scene. Minto Ch. 9, "Applying the Frameworks", p. 153. Six rules build it. Each **In engineering**
clause is a translation.

1. **Use one of three operations only.** Divide, trace cause and effect, or classify. Minto Ch. 9,
   p. 143.
2. **Create a MECE classification at the upper branch.** It guides the causes below it. Minto
   Ch. 9, "Classifying Possible Causes", p. 149. See `07-logical-order-and-mece.md`.
3. **Prefer a split that an identity forces.** Minto splits cigarette cost by Cost/Hour ×
   Hours/Million = Cost/Million. Exhibit 48, Minto Ch. 9, p. 158. **In engineering.** Prefer total
   latency = queue time + service time + network time over ad-hoc buckets.
4. **Mirror the real sequence in your bifurcations.** Minto calls this the secret of the choice
   structure. Minto Ch. 9, p. 150.
5. **Assess the causes in the order in which they are easiest to eliminate.** You do not book a
   brain-tumor test when a sinus may cause the headache. Minto Ch. 9, p. 143. **In engineering.**
   Run the cheap discriminating test first.
6. **Correct the weaknesses on the left before those on the right.** Minto Ch. 9, p. 150. A
   recommendation that reorders dependent fixes is wrong even when each item is right.

### 4.2 Logic trees generate solutions

**Procedure.** Minto Ch. 9, "Generating Possible Solutions", pp. 157–158.

1. Decompose the objective into its elements.
2. Split by an identity where one exists.
3. State the ways in which each factor can improve, then continue to the next level.
4. Calculate the benefit and estimate the risk of each action.
5. Be as collectively exhaustive as you can.

Step 4 gives the shape of a defensible recommendation. You enumerate the possibilities. You price
each one. Then you name the selected set. You do not select first and justify afterwards.

---

## 5. The issues derive from the structure, not from the question

This warning governs every request that this skill receives. Minto analyzes a UK retail bank. The
Opening Scene is the bank plus European retail banking. The Disturbing Event is a policy change
that permits cross-border activity. R1 is the opportunity to operate in other countries. R2 is a
profitable position in Europe. The client asked what its strategy in Europe should be. Minto
Ch. 9, "The Misconceptions", p. 166.

**The client's question usually reflects an R2, not a problem.** The issues cannot derive from the
client's question. "They must come out of the structure of the situation that gave rise to the
R1." Minto Ch. 9, p. 166. **In engineering.** "Should we use Postgres or DynamoDB?" is an R2 that
wears a question mark. Never accept the candidate set before you define the problem. See
`advice-defect-catalog.md`, X-06.

**An issue demands a yes or a no.** "Strictly speaking, an issue is a question so phrased as to
demand a yes or no answer." Minto Ch. 9, "Performing an Issue Analysis", p. 163. "How should we
reorganize?" is not an issue. "Should we reorganize functionally?" is one.

**Call everything else a concern.** Reserve "issues" for yes-no questions, and use "concerns" for
the topics that indicate what worries the reader. Minto Ch. 9, p. 163. Never write a section
called "Issues". State the process that you will follow and the end product instead. Minto Ch. 9,
p. 161.

**Test a written list against the tree it should have derived from.** Draw the tree of the
logically possible ways to reach the objective. Write each item number against a branch. An item
that maps nowhere is irrelevant. An item that maps to the root restates the objective. Two items
on one branch are one item. Minto Ch. 9, "Revealing Flaws in Grouped Ideas", pp. 161–162.

---

## 6. Structure the analysis before you gather data

**Structure the analysis of the problem before you gather any data.** Minto Ch. 9, "Starting with
the Data", p. 142. **Modern.** This is the one instruction that an eager agent skips most often.
**The failure it prevents.** People gather whatever data exist in an area, and they postpone
thought until the facts sit in one place. The result is extra work and a mass of facts that yield
no conclusion. Minto reports that a major consulting firm once estimated that 60% of its
fact-finding effort was wasted. Minto Ch. 9, p. 141.

**The scientific method, as Minto states it.** Minto Ch. 9, p. 142. Generate alternative
hypotheses. Devise a crucial experiment whose alternative outcomes each exclude one or more
hypotheses as nearly as possible. Perform the experiment so that you get a clean result. Plan
remedial action accordingly. **Procedure for planning the evidence.** Minto Ch. 9, "Applying the
Frameworks", pp. 155–156.

1. Gather only the data that you need to build the diagnostic framework.
2. Make an educated guess at where the weaknesses are likely to be.
3. Specify exactly what you expect to observe if a weakness exists.
4. Write the data-gathering questions as yes-no questions, and ask what settles each one.
5. Identify the source of each item, assign an owner, schedule it, and estimate its cost.

**Why yes-no questions matter.** Minto says that they tell you in advance when your research is
finished. Minto Ch. 9, p. 150. **In engineering.** "Does version 16 support transactional DDL?"
ends. "How good is the migration story?" does not.

**The limit Minto states.** "Good problem solving cannot be done in the abstract." Minto Ch. 9,
p. 153. There is no substitute for extensive and accessible knowledge of the subject area.
**Modern.** For this skill that area includes the repository. See
`10-research-method-and-sources.md` and `advice-defect-catalog.md`, X-04.

---

## 7. Abduction, from Appendix A

Appendix A covers the case where the trouble is not that you dislike the result. The trouble is
that you cannot explain it. Minto App. A, opening, p. 210. She names three causes. The structure
does not exist, because you try to invent something new. The structure is invisible, because you
hold only its results. The structure fails to explain the result, because the prevailing theory
predicts one thing and you observe another.

**Abduction.** Charles Sanders Peirce coined the name in 1890, and he chose it to show the
affinity with Deduction and Induction. Minto App. A, pp. 210–211. Peirce names three entities. The
**Rule** is a belief about the way the world is structured. The **Case** is an observed fact. The
**Result** is an expected occurrence when the Rule applies in that Case. Minto App. A, p. 211.
Where you start decides which form of reasoning you use. Exhibit A-1, p. 212. The examples below
are Minto's own, and they run one price-and-sales claim three ways.

| Form | Order | Minto's example |
|---|---|---|
| Deduction | Rule, Case, Result | If we price too high, sales fall. We priced too high. Therefore sales will fall |
| Induction | Case, Result, Rule | We raised the price. Sales fell. Probably the price is too high |
| Abduction | Result, Rule, Case | Sales fell. Sales often fall because the price is too high. Check the price |

**Analytical problem solving is abduction.** You notice an undesired Result. You search your
knowledge of the structure for the Rule. You then test whether you found it. Minto App. A, p. 211.
In business we know the structure that creates our result, so we hold two of the three entities
and reason to the third. The scientist must invent the second first. Minto App. A, p. 212.

**Hypotheses are not drawn out of the air.** The structural elements of the situation that
produced the problem suggest them directly. Minto App. A, "Generating Hypotheses", p. 212. She
says the move frequently requires an analogy between what you know of the problem and what you
know of the world. **The experiment template.** Four lines, in order. Minto App. A, "Devising
Experiments", p. 213.

1. **Result.** I observe the unexpected fact A.
2. **Rule.** A may be so because B is the case.
3. **Case.** If B were the case, then C would follow as a matter of course.
4. Check whether C does in fact follow.

**The yes-or-no gate.** The experiment must yield a clear-cut yes-or-no answer. The result must
let you state without doubt whether you keep or discard the hypothesis. It is not enough to see
what happens when you change a condition. Minto App. A, p. 214. **In engineering.** State which
number keeps the hypothesis and which number discards it before anybody runs the benchmark. The
consultant states the test. The owning skill runs it. See `advice-defect-catalog.md`, X-04 and
X-22.

---

## 8. The book's worked examples

**The industrial real estate seller.** A company sold industrial real estate for 30 years by one
method. Salesmen list the prospects, write a script, and deliver the message. Those three boxes
are the Opening Scene, and they produced 10% annual growth, which is R2. The Disturbing Event is a
projection of quarterly sales down 10% instead of up 10%. Because the Solution changes the Opening
Scene, the candidate causes are the same three boxes. Exhibit 32, Minto Ch. 8, pp. 122–124.

**The retail distributor of household goods.** Three distribution centers plus rented space serve
438 stores, and in theory could serve 490. Volume grows 4 to 5% a year, and 198 new stores open.
R1 is that capacity runs out in two years. R2 is sufficient capacity to cope. The R2 criteria are
lowest capital outlay, lowest operating cost, the same processing speeds, and the same full-line
strategy. The recommendation is a positive action. Add capacity incrementally, to avoid a fourth
warehouse as long as possible. Exhibits 35 and 36, Minto Ch. 8, pp. 137–138. The winner is not one
candidate. It is a course of action assembled from several of them.

**Galileo and the cannonball.** Aristotle's Rule says that force produces velocity, so a body
stops when the force stops. The cannonball keeps moving, which is the unexpected Result. Galileo
sees three structural elements in a falling ball. Weight, distance, and time. If force is
proportional to time, then distance is proportional to the square of the time. He rolls a ball
down an inclined plane to slow the fall. The New Rule is that force produces a change of velocity.
Minto App. A, pp. 213–214.

---

## 9. What this file commits you to

1. Specify all four elements before you accept any candidate set. Minto Ch. 8, p. 127.
2. Draw the Opening Scene from the repository, and never manufacture a Disturbing Event. Minto
   Ch. 8, p. 129, and Ch. 9, p. 153. **Modern** for the repository.
3. State R2 with a threshold or an end state, or make the R2 table the deliverable. Minto Ch. 8,
   p. 130.
4. Place the reader in one of the seven problem situations. Minto Ch. 8, pp. 131–132.
5. Keep one beginning Question per document. Minto Ch. 4, pp. 48–49.
6. Derive every issue from the structure that produced R1, and phrase it to demand a yes or a no.
   Minto Ch. 9, pp. 163, 166.
7. Structure the analysis before you gather any data, and define what a yes means before you run
   the test. Minto Ch. 9, p. 142, and App. A, p. 214.

**When this file does not apply.** The reader asks for the meaning of one term, and no decision
depends on it. Answer the term. Do not build a problem definition. "Not applicable" is a valid
result.

**Where to read next.** `06-the-scqa-introduction.md` turns the four elements into the opening of
the document. `07-logical-order-and-mece.md` holds the grouping tests that the frameworks need.
`09-the-tradeoff-evaluation.md` scores the candidate set against the R2 table, and
`10-research-method-and-sources.md` governs the evidence. `advice-defect-catalog.md` holds the
defects that this file prevents.
