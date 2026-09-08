# Comparing Frameworks and Tools

**Sources:** *The Minto Pyramid Principle: Logic in Writing, Thinking and Problem Solving* —
Barbara Minto (Pearson, 3rd ed.), Ch. 4 "Fine Points of Introductions", Ch. 6 "Imposing
Logical Order", Ch. 7 "Summarizing Grouped Ideas", Ch. 8 "Defining the Problem", Ch. 9
"Structuring the Analysis of the Problem", App. B "Examples of Introductory Structures".
Material the book predates carries **Modern**.

This file holds the procedure that turns "which framework should we use" into a
recommendation the reader can act on and can challenge.

**Minto compares warehouses, motors, and distribution centres. The reader of this file
compares software.** Every paragraph marked **In engineering** is a translation. It is not a
report of what Minto wrote.

**The signal that starts this procedure.** The reader names two or more options and cannot
choose. Minto calls this problem situation 5. Minto Ch. 8, "Look for the Question", pp.
131–132. **The signal that stops it.** The reader cannot say what a good result would be. The
deliverable is then the R2 table, and the comparison waits. Minto Ch. 8, "R2 (Desired
Result)", p. 130.

---

## The order of work

Seven steps. Each step names the signal that makes it due.

| # | Step | The signal | The artifact |
|---|---|---|---|
| 1 | State R2 with thresholds | The reader named options | The R2 table |
| 2 | Assemble the candidate set | R2 has thresholds | The candidate table |
| 3 | Derive the criteria from R2 | The candidate set is closed | The criteria table |
| 4 | Convert each criterion to an issue | The criteria set is MECE | The evidence plan |
| 5 | Score each option against the threshold | Each cell has a named source | The score matrix |
| 6 | Choose the Key Line shape | The matrix is complete or marked | A named shape |
| 7 | Test the verdict four ways | A verdict exists | The four test results |

**Steps 1 to 4 produce no evidence.** Minto forbids data gathering before the structure
exists. A major consulting firm estimated that 60% of its fact-finding effort was wasted.
Minto Ch. 9, "Starting with the Data", p. 141, and p. 142.

---

## 1. State R2 with thresholds before you read one vendor page

**R2 is the desired result, and it is the only thing the options are measured against.**
Minto states the reason directly. Without an end-product description of the Desired Result,
you cannot easily choose between the solutions you generate. Minto Ch. 8, "R2 (Desired
Result)", p. 130.

**Write R2 in end-product terms.** Minto requires a specific number or a specific end state.
Her own examples include "Reduce time to market by 1/3" and "Have sufficient capacity to cope
with projected demand". Minto Ch. 8, "R2 (Desired Result)", p. 130. **In engineering**, write
"p99 write latency stays under 50 ms at 10,000 writes per second", not "better performance".

**The triviality test.** If one instance satisfies the stated result, the result constrains
nothing. Minto's instance is sales. One additional sale improves sales, so "improve sales" is
not a result. Minto Ch. 7, "Summarize Directly", pp. 107–108.

**The "How?" test.** Ask "How?" of the statement and try to fill the boxes below it. Minto
uses "Develop a world consciousness". Nobody can answer, so the statement has no intellectual
value. Her two companion questions are these. What action does it require of us? How do we
recognize the result when it arrives? Minto Ch. 7, "Make the Wording Specific", p. 100.

**The gate.** If R2 cannot be stated, Minto's own instruction applies. Write the general end
state, and make the first work item the definition of the specific R2. Minto Ch. 8, p. 130.
The comparison waits. The R2 table holds five columns. The statement, the threshold or end
state, how the team observes it, who set it, and the date. The date column is **Modern**.

---

## 2. Put the candidate set in the Complication

**Alternatives always go in the Complication.** Minto's rule is that you should not raise
alternatives unless the reader knows them in advance. Write the alternatives memo only when
the options are already under discussion. Minto Ch. 8, "Move to the Introduction", structure
(5), p. 135, and Ch. 4, "Choosing Among Alternatives", p. 55. **In engineering**, list the
options the team named. Do not import a contender nobody proposed. An imported contender widens
the comparison and weakens the verdict.

**One addition is permitted, and it must be declared.** Build a one-level MECE decomposition
of the solution space. Check whether any branch holds no candidate. If a whole class of
solution is unrepresented, add one representative and write the reason in the document. Minto
Ch. 9, "Generating Possible Solutions", pp. 157–158, and Ch. 6, "Creating Proper Class
Groupings", p. 90.

**Every excluded candidate carries a stated reason.** Nothing is dropped in silence. The
exclusion table holds the option, the one-line reason, and who proposed it.

**Score the incumbent.** If a candidate already exists in the repository, name its path and
score it with the others. **Modern.** Minto's nearest rule is that good problem solving
demands knowledge of the area where the problem occurred. Minto Ch. 9, p. 153.

---

## 3. Derive the criteria from the structure, never from a feature list

**The criteria come from the structure of the situation that produced R1.** Minto states the
rule against the opposite practice. The issues cannot derive from the client's question,
because the question usually reflects an R2. They must derive from the structure of the
situation that gave rise to the R1. Minto Ch. 9, "The Misconceptions", p. 166. **In
engineering**, a vendor landing page is that vendor's R2. Criteria copied from it run the
comparison on that vendor's home ground. Every individual claim can be accurate and the set
can still be rigged.

**Prefer a decomposition that arithmetic proves.** Minto splits cost per cigarette by the
identity Cost/Hour × Hours/Million = Cost/Million. Minto Ch. 9, "Generating Possible
Solutions", p. 158. **In engineering**, prefer total latency = queue time + service time +
network time over an ad-hoc bucket list.

### Is this criterion admissible?

| If the criterion… | Then… | Because |
|---|---|---|
| Cannot be phrased as a yes/no question | Reclassify it as a **concern**. Remove it from the matrix | Minto Ch. 9, "Performing an Issue Analysis", p. 163 |
| Has no named evidence source | It is not admissible until you name one | Minto Ch. 9, "Applying the Frameworks", pp. 155–156 |
| Fails the "How?" test | Rewrite it as an end product, or cut it | Minto Ch. 7, "Make the Wording Specific", p. 100 |
| One instance satisfies it | Rewrite it with a threshold | Minto Ch. 7, "Summarize Directly", pp. 107–108 |
| Derives from a vendor feature list | Cut it. It is the vendor's R2, not yours | Minto Ch. 9, "The Misconceptions", p. 166 |
| Restates the whole objective | It is the root of the tree, not a branch | Minto Ch. 9, "Revealing Flaws in Grouped Ideas", pp. 161–162 |
| Serves no R2 row | Cut it, or raise it as a missing R2 row | Minto Ch. 7, "Find the Structural Similarity", p. 112 |
| The same evidence settles it and its neighbour | Merge the two. Name the property underneath | Minto Ch. 6, "Creating a Structure", pp. 82–83 |

**The criteria set is MECE and it holds five criteria at most.** Four at most if the Key Line
is deductive. A set above the cap means you missed a grouping, so regroup rather than
truncate. Minto Ch. 6, "Distinguishing Cause from Effect", p. 78, and Ch. 10, "Underlined
Points", p. 177.

**Run the plural-noun test and the order gate.** One plural noun must label the whole set, and
a member that does not fit the noun is a misfit. The set must also show one of three orders.
Time, structural, or degree. A set with none is defective. Minto Ch. 5, p. 69, and Ch. 6,
opening, p. 77. See `07-logical-order-and-mece.md`.

---

## 4. Convert every criterion to a yes/no issue

**An issue is a question phrased to demand a yes or no answer.** Minto reserves "issue" for
that shape and uses **concern** for a topic that worries the reader. Her own pair shows the
difference. "What level of inventory investment is necessary?" is not an issue. "Is the
present level of inventory too high?" is an issue. Anything that resists a yes/no phrasing
leaves the matrix and becomes a listed concern. Concerns appear in the document. They do not
score. Minto Ch. 9, "Performing an Issue Analysis", p. 163, and "Revealing Flaws in Grouped
Ideas", p. 160.

**In engineering.** Write "Does it support transactional DDL?" Do not write "How good is its
migration story?"

**A yes/no issue tells you in advance when the research ends.** Minto Ch. 9, p. 150. **The
result must let you keep or discard the hypothesis.** Minto states that it is not enough
to see what happens when you change a condition. She quotes Darwin. All observations must be
for or against some view, if they are to be of any service. Minto App. A, "Devising
Experiments", p. 214.

**Order the issues by ease of elimination.** Minto's instance is a headache. You do not book a
brain-tumour test while a sinus may be the cause. Minto Ch. 9, "Devising Diagnostic
Frameworks", p. 143. **In engineering**, read the lock file before anybody plans a load test.
The consultant reads. It never runs the test.

---

## 5. Score each option against the threshold, never against another option

**This is the rule the whole file rests on.** Minto states it once and repeats it. It is
irrelevant how the alternatives compare to each other. What matters is how they compare to the
R2. She also names the cause of a rigged comparison. So-called alternatives arise when the R2
is ambiguously stated, so that you cannot judge that you have a solution when you see it.
Minto App. B, "Dealing with Alternative Solutions", p. 225.

**The matrix shape is Minto's own.** Alternatives run down the side. The criteria by which you
made your judgment run across the top. Marks show where each alternative did or did not match.
Minto App. B, p. 226. **The header row holds the threshold, not the option name.**

| Option | p99 under 50 ms at 10k writes/s? | Licence permits binary distribution? | Documented export path? |
|---|---|---|---|
| A | met — measured, config C, 2026-09-01 | met — LICENSE, v4.2, 2026-09-01 | undetermined — no export page |
| B | not met — 180 ms, config C, 2026-09-01 | met — LICENSE, v2.9, 2026-09-01 | met — docs, v2.9, 2026-09-01 |
| incumbent | undetermined — never measured here | met — LICENSE, v1.7, 2026-09-01 | met — repository path |

**No cell is filled from memory. Every filled cell carries a source, a version, and a date.**
**Modern.** Neither book requires a claim to carry its source into the document. LeFever
supplies the nearest rule. Never invent an answer to appear smart. LeFever Ch. 3, "We Lack
Understanding", pp. 27–28.

**Silence in a source reads *undetermined*, never *no*.** Minto App. A, p. 214. **Report ties
as ties.** More than one option can clear R2. That is a real outcome, not a failure. **One
measurement supports one scoped claim**, because a single piece of evidence forces you to deal
with it deductively. Minto Ch. 5, "How It Differs", p. 71. **In engineering**, write "under
configuration C on workload W we measured M". Do not write "X is three times faster". See
`10-research-method-and-sources.md`.

---

## 6. The Modern axes the books do not know

Minto compares motors and warehouses. She has no concept of a version, a licence, or a
maintainer. **Every axis in this section is Modern.** Never attribute one to either book.

**The completeness discipline is Minto's.** Define specifically the characteristic the items
share, then search your knowledge for every known item with that characteristic. Minto Ch. 6,
p. 90. This section is that search, performed once, for software.

| Axis (**Modern**) | The yes/no issue | Where the answer lives | What a miss costs |
|---|---|---|---|
| Version and release cadence | Is the pinned version still receiving releases? | The release history and the lock file | A recommendation for a product that stopped moving |
| Deprecation and end of life | Is a capability we depend on scheduled for removal? | The deprecation policy, then the changelog | A migration that starts on the day it is due |
| Licence | Does the licence permit our distribution? | The LICENSE file at the pinned version | A copyleft term inside a shipped binary |
| Maintainer health | Do at least two maintainers merge changes? | Commit cadence, release cadence, issue response | A dependency with one person behind it |
| Benchmark methodology | Does the number state workload, hardware, and author? | The benchmark method, not the headline | A number that does not hold for our workload |
| Total cost, operation included | Does the cost stay inside budget at the projected load? | The dated pricing page, then a stated model | A price that scales on a metric nobody modelled |
| Lock-in and exit cost | Is there a documented export in an open format? | The export section of the reference docs | A one-way door that nobody named as one |
| Security and supply chain | Are known vulnerabilities fixed in the pinned version? | The advisory feed and the dependency tree | A defect the comparison never scored |
| Documentation quality | Do the reference docs answer our issues at our version? | The reference docs at the pinned version | Research cost that lands on the whole team |

**These nine axes are candidates, not a checklist to copy.** An axis enters the criteria
table only when an R2 row asks for it. An axis with no R2 row is cut, or raised as a missing
R2 row. Minto Ch. 7, p. 112.

**A dimension you exclude is stated as out of scope, with the reason.** An R2 table that
covers only the axes the team enjoys discussing is not collectively exhaustive. **A security
criterion is scored elsewhere.** State the yes/no issue here and route it. See
`11-routing-to-the-specialist-skills.md`.

---

## 7. Choose the Key Line shape

Minto gives three shapes and one condition per shape.

| If… | Then the Key Line is… | Because |
|---|---|---|
| One option clears every R2 threshold | **Criteria-structured.** "Choose C. It meets criterion 1, 2, and 3" | Minto Ch. 4, "Choosing Among Alternatives", p. 55 |
| One option wins overall and loses one criterion | **Alternative by alternative.** The main reason for C, then against A, then against B | Minto Ch. 4, pp. 55–56 |
| No option clears every threshold, and the thresholds conflict | **Alternative objectives.** "Choose A if what you want is X. Choose B if what you want is Y" | Minto Ch. 4, p. 56, and App. B, pp. 225–226 |
| Several options clear R2 at different cost | **Criteria-structured on the one criterion that discriminates.** Declare the rest as ties | Minto Ch. 5, p. 72 — a criterion every option ties on carries no inference. It is news |
| The shape needs more than five points, or four if deductive | **Regroup.** You missed a grouping | Minto Ch. 10, "Underlined Points", p. 177 |

**The criteria shape is the preferred one, and it has a precondition.** Minto's own wording is
"Select C — it is faster than A or B, it is cheaper than A or B, it is easier to implement".
Forcing that shape when the winner loses a criterion makes the grouping untrue, so switch to
the alternative-by-alternative shape. Minto Ch. 4, pp. 55–56.

**Alternative objectives is the disciplined form of "it depends".** Minto insists on the
label. You are not structuring around alternative ways to solve the problem. You are
structuring around alternative objectives, which is quite a different thing. Minto Ch. 4, p.
56. Appendix B calls the same structure **alternative R2s**. Minto App. B, pp. 225–226. Each
branch of that shape carries its own threshold. A branch with no threshold is a hedge wearing
a structure.

**Prefer induction at the Key Line**, because it is easier on the reader. **Two cases require
deduction.** The first is a verdict that is alien to what the reader expects you to say. The
second is a reader who cannot understand the action without the reasoning first. Minto Ch. 5,
"When to Use It", pp. 64, 66–67. **In engineering**, "Replace it, do not tune it" is alien, so
argue before you recommend.

---

## 8. Never argue by elimination

**The forbidden Key Line.** Way A is no good because… Way B is no good because… Therefore do
way C. **Minto's reason.** The reason for doing C is not that A and B are no good. The reason
for doing C is that it solves the problem. Minto Ch. 8, "Move to the Introduction", structure
(5), p. 136. She restates it in App. B, p. 225.

**The repair.** Move the candidate set into the Complication. Restate the Key Line as "C,
because it meets R2 rows 1, 2, and 3". The defects of A and B may appear as supporting detail
underneath. They never appear on the Key Line.

**The cost of the defect.** A recommendation built on the demerits of rejected options
collapses when one reader defends a rejected option. That reader wins without touching the
real question.

---

## 9. Test the verdict four ways

Run all four before you write the document.

1. **Is the verdict a genuine summary of the Key Line, or a label?** Minto's first rule is
   that ideas at each level must be summaries of the ideas below, because they were derived
   from them. Minto Ch. 7, opening, p. 94.
2. **Does it pass the abstraction test?** If three or four other criteria sets would fit the
   same headline, the headline sits too high. Minto Ch. 5, "How It Works", p. 70, Exhibit 22.
3. **Does it pass the triviality test?** Minto Ch. 7, "Summarize Directly", pp. 107–108.
4. **Read from the bottom up. Do these criteria compel this verdict?** Minto Ch. 5, p. 70.

**No intellectually blank assertion at any node.** "There are three viable options" states a
category and a count. It does not summarize the essence of what sits below. Minto Ch. 7,
"Avoid Intellectually Blank Assertions", pp. 94–95.

**The closing universal test.** Why have I brought together these particular ideas and no
others? Minto allows exactly two answers. They share a characteristic and no other ideas are
linked that way. Or they are all the actions needed to achieve one effect. Minto Ch. 7, "Make
the Inductive Leap", p. 118.

**State the reversal condition.** **Modern** as a document section. Minto supplies the
standard it serves. The reader must be able to determine whether he disagrees with the
reasoning, and to raise logical questions about it. Minto Ch. 2, p. 16. Name the finding that
would flip the verdict, and name how to obtain it.

---

## 10. The books' worked examples

**The motor alternatives memo.** A ruling makes a small motor the efficient choice for cold
drilling, so the largest customer announces a switch to a competitor's model. The company
holds three responses. Cut the price, reengineer the motor, or design a new one. Minto uses it
to show that the candidate set belongs in the Complication, and only because the reader
already holds it. Minto Ch. 4, p. 55.

**The distribution centres.** Three centres plus rented space serve 438 stores, and the
company plans 198 more. R1 is a capacity shortfall in two years. R2 carries four criteria.
Lowest capital outlay, lowest operating cost, unchanged processing speed, unchanged full-line
strategy. The recommendation is "Add capacity incrementally, to avoid building a fourth
warehouse as long as possible". Minto Ch. 8, "Real-Life Example", pp. 137–138. **Note the
shape.** The verdict draws on several candidate options rather than crowning one.

**Colefax Supermarkets.** A sales-based replenishment system was conceived as a central
mainframe system. All data entry and major use sit at branch level, so the question arose
whether a branch-based architecture is more practical. The conclusion is branch-based. This is
the book's own model of a legitimate two-option architecture comparison, and it qualifies
because the reader already held both options. Minto App. B, p. 220.

**The plastic bottle memo.** Eleven risks and constraints argue against in-house plastic
bottle manufacture. Minto maps them onto the standard ROI tree, which produces one message
about profitability. The memo omits any assessment of the favourable effect of plastic
containers on product sales. The company entered the business and made an immense success of
it. Minto Ch. 6, "Using the Concept to Clarify Thinking", pp. 87–89. **The recommendation was
reversed by the thing nobody evaluated.**

**The energy "Major Issues" list.** Ten issues map onto a tree of the logically possible ways
to cut energy cost. Three map to no branch, so they are irrelevant. One maps to the root, so
it restates the objective. One maps to two branches, so it is a duplicate. Minto Ch. 9,
"Revealing Flaws in Grouped Ideas", pp. 161–162. **In engineering**, run this as a validity
test on a criteria list somebody already wrote.

---

## Proportionality

| The question | Required output |
|---|---|
| A term with no decision attached | Two sentences. No record |
| A reversible library choice inside one module | R2 table, three criteria, the matrix, the reversal condition |
| A framework, a database, a platform, or a vendor | The full record, plus the Modern axes that R2 asks for |
| Money, identity, personal data, or a one-way door | The full record, plus a routing line to the owning skill |
| The reader has decided and wants agreement | Validate the one option against R2, or say R2 is missing |
| The incumbent meets R2 and is not understood | No comparison. Write the explanation |

**"Not applicable" is a valid result.** It is better than an invented comparison.

---

## Catalog codes this file supports

Run these entries of `advice-defect-catalog.md` against any comparison record.

| Code | The defect this file prevents |
|---|---|
| X-05 | No R2 |
| X-11 | Straw-man bake-off |
| X-12 | Options scored against each other |
| X-13 | A missing non-functional dimension |
| X-14 | Overlapping criteria |
| X-15 | A criteria set with a hole |
| X-16 | An unfalsifiable criterion |
| X-17 | Vendor framing adopted |
| X-25 | A verdict that is not a summary of the criteria |
| X-26 | A causal chain shown as parallel points |
| X-27 | A decisive omission |
| X-28 | An intellectually blank top line |
| X-29 | No reversal condition |
| X-30 | An unbranched "it depends" |
| X-31 | A silent exclusion |

---

## One place Minto is too strong for engineering

Minto writes that strictly speaking there is no such thing as an alternative solution to a
problem, provided the problem has been properly defined. Minto Ch. 4, p. 54, and App. B, p. 225.
**This rule is too strong for software.** Several options routinely clear the same R2 at
different costs along different axes. The decision is then a real trade.

**Keep the rules that follow from her doctrine.** Compare each option to R2. Never argue by
elimination. Put alternatives in the Complication. **Use her own escape hatch when more than
one option qualifies.** Structure the document around alternative R2s, and label it as
alternative objectives. Minto App. B, pp. 225–226.

---

## Related files

| You need | Read |
|---|---|
| The four problem elements, R1, R2, and the beginning Question | `08-defining-and-structuring-the-problem.md` |
| MECE, the plural noun, the three orders, and the summary rules | `07-logical-order-and-mece.md` |
| The Situation, Complication, Question, and Answer of the opening | `06-the-scqa-introduction.md` |
| The source ladder, the version check, and the claim ledger | `10-research-method-and-sources.md` |
| The reader level that decides how much *why* the record carries | `01-the-explanation-scale.md` |
| The skill that owns a criterion this consultant cannot score | `11-routing-to-the-specialist-skills.md` |
| The full catalog, all 45 entries | `advice-defect-catalog.md` |
