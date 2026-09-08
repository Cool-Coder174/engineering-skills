# Logical Order, MECE, and Summarizing

**Sources:** *The Minto Pyramid Principle: Logic in Writing, Thinking and Problem Solving* —
Barbara Minto (Pearson, 3rd ed.), Ch. 5 "Deduction and Induction: The Difference", Ch. 6
"Imposing Logical Order", Ch. 7 "Summarizing Grouped Ideas". Supporting rules from Ch. 1, Ch. 10
"Reflecting the Pyramid on the Page", and App. C. Content the book predates carries **Modern**.

This file holds the tests that decide whether a group of ideas is valid. Minto names the
combined work **Hard-Headed Thinking** (Intro to Part 2, p. 74). Chapter 6 finds the framework
that holds the ideas together and fixes their order. Chapter 7 states the insight that framework
implies. Read this file when you build a criteria set, a Key Line, or a set of claims about one
concept. Read it again before you write the verdict.

**Minto writes about business memos. The reader of this file works on software.** Every
paragraph marked **In engineering** is a translation. It is not a report of what Minto wrote.

---

## 1. Deduction and induction

**Deduction states a point, comments on that point, then states the implication.** Minto Ch. 5,
"Deductive Reasoning", p. 62. The second point comments on the subject or on the predicate of
the first.

**Induction groups ideas of the same kind, then states what the sameness means.** Minto Ch. 5,
"INDUCTIVE REASONING", p. 68. The mind notices that several ideas are similar and comments on
the significance of that similarity. Minto calls induction harder to do well than deduction.

**One grouping is deductive or it is inductive. It is never both at once.** App. C, Ch. 2,
point 3, p. 235.

| Read the second point | The relation is | Source |
|---|---|---|
| It comments on the subject or the predicate of the first point | **Deductive** | Ch. 5, "How It Differs", p. 71 |
| It adds a member to the class the first point named | **Inductive** | Ch. 5, p. 71 |
| Nothing about it is the same as the first point | **Neither.** The points do not belong in one document | Ch. 5, p. 72 |

**Minto's own self-test (Ch. 5, p. 71).** Start from "Japanese businessmen are escalating their
drive for the Chinese market". The follow-up about arriving American businessmen stimulating
them further comments on the first point, so it is deduction. The follow-up "American
businessmen are escalating their drive for the Chinese market" adds a member to the class, so it
is induction. The test is mechanical and it is fast.

**Hold one element constant.** Minto Ch. 5, pp. 71–72. Fix the subject and vary the predicate,
or fix the predicate and vary the subject. Vary both and no inference follows.

**One piece of evidence forces deduction.** Minto Ch. 5, p. 71: "Whenever you have only one
piece of evidence for anything, you are forced to deal with it deductively."

**In engineering.** One benchmark run is one piece of evidence. **Modern.** State it deductively
and state its scope. "On workload W, under configuration C, we measured M." The constant-element
rule also sets the shape of a table. Three tools against one criterion, or one tool against
three criteria. Never both at once. See `advice-defect-catalog.md`, entry X-21.

---

## 2. Prefer induction at the Key Line

**On the Key Line, present the message inductively.** Minto Ch. 5, "When to Use It", p. 64. Her
stated reason is that induction is easier on the reader. A deductive Key Line makes the reader
hold what is going wrong, match it to the cause, and carry both to the action. The advice
applies to the Key Line only, and not to the levels below it (p. 67).

**Present the action before the argument.** Minto Ch. 5, pp. 65–66. The reader cares about the
action. Minto marks this as a rule of thumb, and she names the rare case where the reader cares
about the argument instead.

| Signal | The Key Line shape | Source |
|---|---|---|
| The verdict is what the reader expects | **Inductive.** Parallel reasons under the verdict | Ch. 5, p. 66, Situation 1 |
| The verdict is alien to what the reader expects | **Deductive.** Argue first, then act | Ch. 5, p. 66, Situation 2 |
| The reader cannot understand the action without the reasoning | **Deductive.** Reasoning first, procedure second | Ch. 5, pp. 66–67 |
| Every other case | **Inductive** | Ch. 5, p. 64 |

**The two situations, in Minto's own dialogue (Ch. 5, p. 66).** In Situation 1 the reader asks
how to cut costs, you answer that cutting costs is easy, and you give A, B and C. A standard
inductive pyramid serves. In Situation 2 the reader asks the same question, and you answer that
he should forget cutting costs and consider selling the business. He now asks why. The Key Line
becomes deductive. The threat from abroad grows, the present structure cannot respond, a
different owner could respond, therefore sell. The second exception is the David Hertz
risk-analysis article (pp. 66–67). That reader needed the reasoning under the approach before he
could understand its steps.

**Push deduction as low in the pyramid as possible.** Minto Ch. 5, p. 67. "I am a bird /
Therefore, I fly" is easy to absorb, because the two points touch. The same reasoning loses its
clarity across ten or twelve pages. A deductive argument holds no more than four points, and no
more than two chained "therefore" points (p. 68). Minto states that you can break both limits,
and that the groupings then become too heavy to summarize. Chaining is permitted only when the
reader can supply the missing steps (p. 63).

**In engineering.** Your evaluation is deductive. You measured, you diagnosed, you chose. The
record is inductive. A recommendation against the option the team already selected is
Situation 2. Give the argument first. See `06-the-scqa-introduction.md`.

---

## 3. The three logical orders, and the order gate

**Ideas in any grouping must be in logical order.** Minto Ch. 6, opening, p. 75. This is her
second pyramid rule.

**The source of the grouping dictates its order.** Minto Ch. 6, Exhibit 23, p. 76. The mind
performs three analytical activities of this kind, and each one forces one order.

| The mind did this | The grouping is | The order is | Engineering instance |
|---|---|---|---|
| Determined the causes of an effect | A process or a system | **Time order** | Migration steps. A request path |
| Divided a whole into its parts | A structure | **Structural order** | Service topology. A module graph |
| Classified like things | A class | **Degree order** | Three risks. Four criteria |

Minto also calls degree order **comparative order** or **order of importance** (p. 77).

**Time order lists the steps in the order a person must take them.** Minto Ch. 6, "TIME ORDER",
p. 77. A process runs one step at a time, and the summary of the set is the effect (p. 76).

**Structural order reflects what you see after you visualize the thing.** Minto Ch. 6,
"STRUCTURAL ORDER", p. 82. Divide the thing, then describe the parts as they appear on the
diagram. Divide by functioning part, and show the parts in the order they perform that function
(p. 83). **The radar set (pp. 83–84)** runs modulator, oscillator, antenna, receiver, indicator.
The modulator takes in power that the oscillator then gives out, and each part hands the next
its input.

**Degree order ranks members by how strongly each holds the shared characteristic.** Minto
Ch. 6, "DEGREE ORDER", p. 89. State the strongest first. **The printed order asserts a ranking
even when you intended none** (p. 90). Her Telecom billing case lists customer needs, then
management requirements, then outside regulations, which tells the reader that the customer
matters more than the regulator. Minto permits the reverse order for drama, and she rules that
drama is style, not logic.

### The order gate

**One of the three orders must be present to justify a grouping.** Minto Ch. 6, p. 77. She
states the diagnostic in the same place: "If you don't find one, it tells you instantly that
there is something wrong with the grouping."

**A missing order means one of two things.** App. C, Ch. 6, point 3, p. 237. Either the ideas do
not relate logically, or your thinking about them is incomplete. Both require a regrouping.
Neither permits you to ship the list.

**The review checklist.** Steps 1 to 3 come from Ch. 6, p. 93. Step 4 comes from Ch. 5, p. 70.

1. Read down the list. Do you find time, structural, or degree order?
2. If you find none, identify the source. Is it a process, a structure, or a class? Impose the
   order that source dictates.
3. If the list is long, find similarities that let you build subgroups. Order those subgroups.
4. Question from the bottom up. Ask whether the grouped items compel the top point. If three or
   four other item sets fit that point, the point sits too high.

**The order changes with the question you answer.** Minto Ch. 6, p. 92: "the order reflects the
process, and the process is dependent on the question being answered." She takes one grouping of
eight buyer complaints and orders it three ways, one way per question a reader might ask.

**In engineering.** A criteria list you cannot order is a criteria list you cannot summarize.
Name the Question first. See `08-defining-and-structuring-the-problem.md`.

---

## 4. MECE and the plural-noun test

**MECE means mutually exclusive and collectively exhaustive.** Minto Ch. 6, "Creating a
Structure", pp. 82–83. The abbreviation is hers. She glosses the two terms as "no overlaps" and
"nothing left out". The rule covers any division of a whole, physical or conceptual.

**You cannot summarize a grouping unless it is MECE.** App. C, Ch. 7, point 2, p. 238.

**Akron Tire and Rubber (Ch. 6, Exhibit 24, p. 82).** The chart shows a Tire Division, a
Housewares Division, and a Sports Equipment Division. What happens in Tire is not duplicated in
Housewares, so the units are mutually exclusive. What happens in all three is everything that
happens in the company, so the units are collectively exhaustive. Minto uses the case to show
that you already apply MECE when you draw an organization chart.

**Find one word that names the kind of idea in the grouping. It is always a plural noun.** Minto
Ch. 5, "How It Works", p. 69. Her own examples are warlike movements, schemes, steps, indicators,
and ways of hurting. **An idea that does not fit the plural noun is a misfit.** Move it, or
remove it.

| Test | The question you ask | The failure it catches | Source |
|---|---|---|---|
| **Plural noun** | Does one plural noun name every member? | Mixed categories | Ch. 5, p. 69 |
| **Misfit** | Does every member fit that noun? | One member does not belong | Ch. 5, p. 69 |
| **Mutually exclusive** | Would the same evidence settle two members? | One property counted twice | Ch. 6, pp. 82–83 |
| **Collectively exhaustive** | Does a known item hold the characteristic and sit outside the set? | A hole nobody can see | Ch. 6, pp. 82–83, 90 |

**The plural noun controls the inference.** Minto Ch. 5, p. 68, the Polish tanks case. The
events were defined as warlike movements against Poland, so the inference was an imminent
invasion. Defined instead as preparations by Poland's allies to attack the rest of Europe, the
same events support a different inference. The noun decides what the grouping can prove.

**In engineering.** "Performance" and "scalability" usually fail the exclusivity test, because
one load test settles both. Merge them and name the property underneath. Minto gives the
compact form of the same test at Ch. 10, p. 175. "Introduction" and "Background" overlap
because both hold introductory material. See `advice-defect-catalog.md`, X-14 and X-15.

---

## 5. Proper class groupings, and improper ones

**Degree order is where listing replaces thinking.** Minto Ch. 6, "DEGREE ORDER", p. 89: "it is
here that the tendency to list rather than to think becomes most acute." A line such as "the
company has three problems" is not literal truth. The company holds a universe of problems, and
you classified three of them as noteworthy.

**Create a proper class grouping in three steps.** Minto Ch. 6, "Creating Proper Class
Groupings", p. 90.

1. Define specifically the characteristic that the items share.
2. Search your knowledge for every known item that holds that same characteristic.
3. Rank the items by the degree to which each one holds it. State the strongest first.

Step 2 is the completeness test. **Modern:** it is also your answer to the reviewer who asks why
you never evaluated option X. See `advice-defect-catalog.md`, entry X-31.

**Repair an improper class grouping in three steps.** Minto Ch. 6, "Identifying Improper Class
Groupings", p. 92. She names this the only process she knows that reaches the real thinking
under a list.

1. Identify the type of point that each item makes.
2. Group together the items of the same type.
3. Find the order that the set of groups implies.

**The causes of New York's decline (Ch. 6, p. 93).** Eight causes resist ranking. Wage rates,
energy and rent and land costs, traffic congestion, lack of modern factory space, high taxes,
technological change, new centers in the Southwest and West, and the move of American life to
the suburbs. Sorted by type they become three groups. High costs, an unsuitable area, and
attractive alternatives. Three groups rank easily, and the top line becomes "The causes of New
York's decline are easy to trace."

**Grouping by a process or a structure is legitimate when you declare it.** Minto Ch. 6, p. 90.
Say which source you used, and accept the order that source imposes.

**In engineering.** A list of eleven findings against a candidate tool is not evidence. Ask of
each finding why it is a bad thing, group the answers, then impose the order. Minto's plastic
bottle case (Ch. 6, pp. 87–89) shows the payoff and the risk. She maps eleven scattered risks
onto a return tree and the message appears. She then names the omission that overturned the
memo. Nobody had assessed the favorable effect on product sales, and the company entered the
business and succeeded. Ch. 6, p. 89: "It is the imposition of the structure that permits you to
see flaws and omissions." See `advice-defect-catalog.md`, entry X-27.

---

## 6. The size cap on a grouping

**Limit a grouping to four or five points.** Minto Ch. 6, "Distinguishing Cause from Effect",
p. 78. She invokes the Magic Number Seven first, after George A. Miller (Ch. 1, p. 3), then
tightens it herself. In a grouping above five members, it is remote that no two ideas are more
closely related to each other than to the rest. You hide your own thinking when you leave that
relation unstated.

**The Ten Commandments (Ch. 6, p. 78).** Ten items sit in one flat list. Note that some are sins
against God and some are sins against man, and the reader gains an insight that the flat list
never delivered. Subgrouping adds meaning. It does not only tidy.

| The grouping | Maximum members | Source |
|---|---|---|
| A chained deductive argument | **Four** | Ch. 5, p. 68. Ch. 10, "Underlined Points", p. 177 |
| An inductive grouping | **Five** | Ch. 6, p. 78. Ch. 10, p. 177 |
| Chained "therefore" points inside one argument | **Two** | Ch. 5, p. 68 |

**Above the cap, regroup. Never truncate.** Minto Ch. 10, p. 177 states that a longer list means
you missed a chance to group. Four rows removed from a nine-row matrix are four criteria that
nobody evaluated.

**In engineering.** A nine-row criteria matrix is a missed grouping, not a sign of rigor. Sort
the rows by type. Name each group. Cite the original row numbers under each group. A reviewer
can then prove that nothing left the set. Minto uses that citation method herself at
Ch. 7, pp. 106–107. The cap is gate 8 of the SKILL.md, so it carries no catalog code.

---

## 7. Intellectually blank assertions

**An intellectually blank assertion states the kind of idea below it and no more.** Minto
Ch. 7, "Avoid Intellectually Blank Assertions", pp. 94–95. The term is Minto's own. Her examples
are "The company should have three objectives", "There are two problems in the organization",
and "We recommend five changes". The form is a category plus a count.

**The blank assertion carries two separate costs. They are not the same cost.**

| Who pays | The cost | Source |
|---|---|---|
| **The reader** | The line does not anchor his mind and is not stimulating to read. He seizes the first supporting point and answers that point instead | Ch. 7, p. 95 |
| **The writer** | The line conceals thinking that never finished. You cannot comment further on it and you cannot find others like it, so your own thinking stops | Ch. 7, pp. 94–96 |

The writer's cost is the one engineers miss. A blank top line is not a style defect. It is proof
that the derivation never finished. Minto Ch. 7, p. 94: "the act of summarizing the grouping is
the act of completing the thinking."

**John Wain and Samuel Johnson (Ch. 7, p. 95).** A radio speaker says that Wain is well placed
to write the Johnson biography "for three reasons". Same poor Staffordshire background, same
Oxford education, same literary preferences. The second speaker, given no idea to hold, replies
to the first reason alone. Everyone laughs and the subject dies. The derived summary is that
Wain and Johnson are essentially the same kind of people. Under that line the three reasons land
as support.

**The two organization problems (Ch. 7, Exhibit 27, p. 96).** A writer knew his top line was
blank and could not repair it, because no order fitted the two items below. Pressed on where the
ideas came from and how they were alike, he found that he was not discussing organization
problems at all. He was discussing areas of the organization that need more delegation. There
were four such areas, not two, and he had identified only one of them properly. The real summary
is that the major problem is an inability to delegate authority. The blank line had concealed
both the wrong category and the missing members.

**The derived summary takes one of two forms.** Minto Ch. 7, p. 97, Exhibit 28. Every idea is an
action statement or a situation statement.

| The grouping holds | Summarize it by | Source |
|---|---|---|
| **Action ideas** — steps, recommendations, changes | The effect the actions produce together | Ch. 7, p. 97 |
| **Situation ideas** — reasons, problems, conclusions | What their similarity implies | Ch. 7, p. 97 |

**In engineering.** "We evaluated four databases on five criteria" is a blank assertion. The
derived line reads "Postgres is the only candidate that meets all four thresholds." See
`advice-defect-catalog.md`, entry X-28.

---

## 8. State the effect of actions

**The summary of a set of actions is always the effect of carrying out the actions.** Minto
Ch. 6, p. 76, and Ch. 7, "State the Effect of Actions", p. 98. **A MECE set of actions plus its
effect is a unique closed system** (p. 98, Exhibit 29). Take that set of actions and you can be
certain of the stated effect.

**Word every action as an end product.** Minto Ch. 7, "Make the Wording Specific", p. 99: "the
effect must be so specifically stated that it implies an end product you can hold in your hand."
Visualize a person who performs the action. State what he holds when he finishes. Where no
number exists, name a tangible cutoff that proves the step is complete.

| Vague wording | End-product wording | Source |
|---|---|---|
| Strengthen regional effectiveness | Assign planning responsibility to the regions | Ch. 7, Exhibit 30, p. 101 |
| Reduce accounts receivable | Establish a system that follows overdue accounts | Ch. 7, Exhibit 30, p. 101 |
| Review management processes | Determine whether management processes need revision | Ch. 7, Exhibit 30, p. 101 |
| Improve financial reporting | Install a system that gives early notice of change | Ch. 7, Exhibit 30, p. 101 |

**The "How?" test.** Minto Ch. 7, p. 100. Ask "How?" of the statement and try to fill the boxes
below it. "Develop a world consciousness" fails. Minto adds two matching questions. How will we
know when we have done it? Can you tell a person who has done it from a person who has not?

**The triviality test.** Minto Ch. 7, pp. 107–108. "Improve Equity sales" fails, because one
extra sale improves sales. "Reduce accounts receivable" fails, because one paid bill reduces
receivables. A summary that one instance satisfies constrains nothing.

**Distinguish the levels of action.** Minto Ch. 7, "Distinguish the Levels of Action", p. 104.
An idea sits at the same level when the reader performs it **before** the next action. It sits
at a lower level when he performs it **so that** he can produce the next action. In her
telecommunications case (pp. 104–105) that one sort turns ten flat steps into three level-one
steps with sub-steps. It exposes a gap at once. The original ten hold no step that appoints
a central manager.

**Do not classify action ideas.** Minto Ch. 7, p. 106: actions unite only through their power to
bring about one specific effect. Labels such as Tasks, Objectives and Benefits slice the pyramid
the wrong way and always produce repetition. In her worked case, fourteen labelled items
collapse to about four distinct ideas.

**Summarize directly, under two conditions.** Minto Ch. 7, "Summarize Directly", p. 107. The
grouping must be MECE, and the summary must state the direct effect, worded as an end product.
Minto states plainly that these two conditions do not guarantee the right summary.

**In engineering.** The Pros, Cons, Risks and Nice-to-haves bucket is the same anti-pattern.
Pare each item to a few words, list them, find the repetitions, and rebuild around end-product
actions. Write "cut p99 write latency below 50 ms by the second quarter", not "improve
performance". **Modern.** The threshold belongs to the R2 table in
`08-defining-and-structuring-the-problem.md`. See `advice-defect-catalog.md`, entry X-16.

---

## 9. The inductive leap

**Situation ideas group by similarity, and the summary states what the similarity implies.**
Minto Ch. 7, "Look for the Similarity in Conclusions", p. 110.

**Most writers stop at step one of three.** Minto Ch. 7, p. 111. Step one lists points worth
thinking about. Step two proves that the points belong together, by naming the common link that
separates them from all others. Step three states the wider significance of that link, which
creates a new idea. The thinking is complete only after step three.

**Finding the structural similarity has a decision rule.** Minto Ch. 7, "Find the Structural
Similarity", p. 111.

| What you observe | Search for the similarity in |
|---|---|
| The subjects are all the same | The predicates |
| The actions or the objects are all the same | The subjects |
| Neither the subjects nor the predicates match | The judgment each statement implies |

**Get behind the language.** Minto Ch. 7, p. 111: "The trick is to get behind the language to
see the bare structure of what is being said." She names the Five Forces, the Seven Ss, the Four
Ps and the Seven Habits as phrasings that soothe the reader and block the check.

**Truth is not relevance.** Minto Ch. 7, p. 112: "The fact that the points are true is not
sufficient to make them relevant." A table of accurate cells supports no message until you state
what holding those characteristics signifies. Minto Ch. 5, p. 72 names the related failure.
Unrelated true facts are **news**, and a document that communicates your thinking has no place
for news. Failure to find a clear relation among ideas grouped as problems, reasons or
conclusions is always a sign that the ideas need more thought (Ch. 7, p. 112). The fault sits in
your ideas, not in your reader.

**Look for closer links.** Minto Ch. 7, p. 113. Subdivide the list into tighter groups, then ask
why those groups and no others. Her five complaints split into information that does not exist
and information that exists and is not adequate. Order then supplies the third member nobody
wrote. Information that exists, is adequate, and is presented badly.

**The inductive leap is the move to the insight when the implication is hard to see.** Minto
Ch. 7, "Make the Inductive Leap", pp. 114–115. Her springboard is a visualization of the source
of the relation that the grouping reflects. Draw the situation, then read the conclusion off the
drawing.

**The automotive aftermarket (Ch. 7, pp. 115–116).** Five conclusions split into positives and
negatives. The positives summarize as an attractive market, which Minto draws as a circle.
Fragmentation puts segments inside the circle. Uncertainty makes some segments look different.
The barriers become a line that stops entry. She reads two points off the picture. Only some
parts of the market are attractive, and those parts are hard to enter. Nothing about the two is
the same, so they cannot relate inductively. They relate deductively and demand a "Therefore"
that the deck never supplied. Therefore abandon it? Therefore buy your way in?

**The closing universal test.** Minto Ch. 7, p. 118. Ask of every grouping: "Why have I brought
together these particular ideas and no others?" Exactly two answers are valid.

1. The ideas share one characteristic, and no other ideas are linked in that way. The summary
   states the insight the similarity carries.
2. The ideas are all the actions needed together to achieve one effect. The summary states the
   direct effect of those actions.

**The gestalt allowance, with its condition.** Minto Ch. 7, p. 117. You will not enforce this
discipline with absolute rigidity everywhere, because a reader imposes a gestalt where he must.
Her condition is narrow and explicit. You can accept a less precise summary point only when you
know your reasoning is valid. You simplify **after** you derive, never instead of deriving.

**In engineering.** A comparison that ends at "some options are strong and some are risky"
stopped where the automotive deck stopped. Supply the "Therefore". See
`advice-defect-catalog.md`, entries X-25 and X-30.

---

## 10. The cause-and-effect masquerade

**The most common problem is failing to distinguish cause from effect.** Minto Ch. 6,
"Distinguishing Cause from Effect", p. 78. **A causal chain presented as parallel points is
deduction masquerading as induction** (Ch. 5, p. 71).

**Composing room costs (Ch. 5, Exhibit 22, pp. 70–71).** Three findings sit under one line that
reads "Composing room costs may represent a profit-improvement opportunity". Productivity is
low. Overtime is high. Prices are uncompetitive for simple jobs. Questioned from the bottom up,
the line fails twice. Three or four other finding sets fit the same label, so the line sits too
high. Worse, the three findings are one chain. Low productivity produced the overtime, and the
overtime produced the prices. The honest top point is causal. Prices are high because
productivity is low.

**Promote the effect and demote the causes.** Minto Ch. 6, p. 81. Her investment-evaluation list
holds three points, and the third one, "results in misleading prescriptions", is the effect of
the other two. It moves above them. The underlying process then appears. Establish a concept,
develop a technique from it, apply the technique. The author commented on the first two stages
and never on the third, so the reviewer knows the question to ask.

**The repair test.** Minto Ch. 6, p. 79. Visualize yourself performing each action and state what
you hold at the end. Then judge whether you must perform one action **before** the next, or **so
that** the next becomes possible. The second case makes it a sub-step, not a peer.

**In engineering.** "Slow builds", "long CI queue" and "low deploy frequency" are one chain, not
three findings. **Modern.** Scored as three cells, one defect costs an option three times. The
single fix that would resolve all three stays invisible, because nobody named the cause. Ask
of every finding group whether it is parallel or a chain. See `advice-defect-catalog.md`,
entry X-26.

---

## 11. Faults and repairs

| Symptom | Diagnosis | Repair | Source |
|---|---|---|---|
| No time, structural or degree order is visible | The grouping is defective | Identify the source, then impose its order | Ch. 6, pp. 77, 93 |
| Two criteria always score together | An overlap | Merge them. Name the property underneath | Ch. 6, pp. 82–83 |
| Nine rows under one heading | A missed grouping | Sort by type, name each group, rank the groups | Ch. 6, p. 78. Ch. 10, p. 177 |
| The top line gives a category and a count | An intellectually blank assertion | State the effect, or state the implication | Ch. 7, pp. 94–97 |
| Three or four other finding sets fit the top line | The inference sits too high | Lower the line until it speaks only about these points | Ch. 5, p. 70 |
| The findings form a chain | Deduction masquerading as induction | Promote the effect. Demote the causes under it | Ch. 5, p. 71. Ch. 6, p. 78 |
| A step exists only to produce another step | Mixed levels of action | Sort on "done before" against "done so that" | Ch. 7, p. 104 |
| One instance satisfies the summary | The triviality failure | Word the effect as the end product the steps produce | Ch. 7, pp. 107–108 |
| The set closes on two points with nothing in common | The reasoning stopped early | Supply the "Therefore" | Ch. 7, p. 116 |
| True facts appear that relate to nothing | News, not thinking | Remove them | Ch. 5, p. 72 |

---

## 12. Proportionality and cross-references

**Not every answer needs a grouping.** A single fact with a citation is a fact. It carries no
plural noun, no order, and no summary. Two sentences that define a term for one engineer carry
no criteria set, so sections 3 to 6 do not apply. A comparison that decides a database, a
platform, or a vendor carries all of them. **"Not applicable" is a valid result.** It is better
than an invented grouping.

| Read next | For |
|---|---|
| `05-the-pyramid-principle.md` | The three pyramid rules and the question-answer dialogue |
| `06-the-scqa-introduction.md` | The Key Line shapes, and where the Answer sits |
| `08-defining-and-structuring-the-problem.md` | R1, R2, and the thresholds end-product wording needs |
| `09-the-tradeoff-evaluation.md` | The criteria table, the scoring matrix, and the verdict |
| `10-research-method-and-sources.md` | The evidence a cell may carry, and the citation rules |
| `03-the-packaging-elements.md` | The connection, and the clause where an analogy stops |
| `01-the-explanation-scale.md` | How much the reader knows before you choose a grouping |
| `11-routing-to-the-specialist-skills.md` | Where an incident, a fault, or a build goes instead |
| `advice-defect-catalog.md` | Entries X-14, X-15, X-16, X-21, X-25, X-26, X-27, X-28, X-30 and X-31 |
