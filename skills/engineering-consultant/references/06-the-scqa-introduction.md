# The SCQA Introduction

**Sources:** *The Minto Pyramid Principle: Logic in Writing, Thinking and Problem Solving* —
Barbara Minto (Pearson, 3rd ed.), Ch. 4 "Fine Points of Introductions", App. B "Examples of
Introductory Structures". Supporting rules from Ch. 3 "How to Build a Pyramid Structure",
section "Caveats for Beginners", and App. C. Content the book predates carries **Modern**.

Every record this skill produces opens with a story. The story states a Situation. Inside it a
Complication develops. The Complication raises the Question. The document is the Answer. Minto
abbreviates the four elements as S-C-Q-A. She states that writers botch more introductions than
any other part of a document (Ch. 4, p. 43). **Minto writes about business memos. The
reader of this file writes about software.** Every row and paragraph marked **Modern** is a
translation by this skill, not a report of Minto.

---

## 1. The four elements

| Element | What it holds | The test it must pass | Source |
|---|---|---|---|
| **Situation** | An established truth about the subject | The reader agrees without proof | Ch. 4, p. 36 |
| **Complication** | What happened inside that Situation to disturb it | It raises the Question you already wrote | Ch. 3, p. 23, step 6 |
| **Question** | What the disturbed reader now asks | There is exactly one | Ch. 4, pp. 48–49 |
| **Answer** | Your main point | It answers that Question and no other | App. C, Ch. 2, point 6 |

**The three story elements are never optional.** Minto permits you to reorder them. She does
not permit you to omit one (Ch. 4, theory point 2, p. 48). A long document adds a fourth
element that states what is to come, and in a long record that element is the Key Line.
**Engineering form. Modern.** The Situation is the repository as it stands today. The
Complication is the event that made today unacceptable. The Question is the one thing the
reader wants settled. The Answer is the recommendation, or the core idea.

---

## 2. Where to start the Situation

**The opening statement must be self-sufficient and noncontroversial** (Ch. 4, "Where Do You
Start the Situation?", p. 36). Self-sufficient means no earlier statement is needed to make its
meaning clear. Noncontroversial means the reader agrees automatically. It also anchors him in a
specific time and place (Ch. 4, p. 37).

**Two tests decide what counts as known.** The objective-observer test: an observer can check
the point and call it true (Ch. 3, Caveat 5, p. 32). The magazine test: the information could
have appeared in *Business Week* or *Fortune* (Ch. 4, p. 37). **Engineering form. Modern.**
Software has no equivalent magazine. Use the artifacts instead.

| Start the Situation from… | Because… |
|---|---|
| The manifest, the lockfile, or the container file | The pinned version is checkable and undisputed |
| A path in this repository | The reader can open it |
| A dated metric the team already reviewed | The team accepted it once already |
| A published deprecation notice, with its date | The vendor states it, and the date is fixed |

**Never start the Situation from a claim you must prove.** **If you do not want to state
something about the subject, the subject is wrong.** Either you chose the wrong subject, or you
started in the wrong place (Ch. 4, p. 36).

---

## 3. What a Complication is

**A Complication is not always a problem.** It is the turn in the story you tell, and it creates
the tension that produces the Question (Ch. 4, "What's a Complication?", p. 37). Exhibit 10
lists the four shapes, and the last column translates them. **Modern.**

| Situation | Complication | Question | Engineering instance |
|---|---|---|---|
| Have a task to perform | Something stops us | What should we do? | The runtime we target drops the library we use |
| Have a problem | Know the solution | How do we implement the solution? | We chose the queue, and the cutover is unplanned |
| Have a problem | A solution has been suggested | Is it the right solution? | One engineer proposes a rewrite |
| Took an action | The action did not work | Why not? | We added the cache, and p99 did not move |

**The candidate options belong in the Complication.** Minto places alternatives there, and only
when the reader already knows them (Ch. 8, p. 135, and Ch. 4, p. 55). A comparison that
introduces options nobody proposed is a straw-man bake-off, entry X-11.

---

## 4. The Question, and why there is only one

**One document answers one beginning Question.** Minto calls it the pivot on which the whole
document depends (Ch. 4, p. 48). Two questions collapse into one. "Should we enter the market,
and if so, how?" is really "How should we enter the market?" (Ch. 4, pp. 48–49). The decision
to act sits inside the "How?". **Engineering form. Modern.** "Should we migrate, and if so, to
what?" collapses to "How should we migrate?" The decision to migrate becomes the Answer, and
the choice of target becomes the Key Line. See entry X-07.

**The Complication checks the Question.** State the Complication. It must raise the Question
you already wrote. If it raises a different one, change the Question. If the Question is right,
change the Complication (Ch. 3, p. 23, step 6). A directive, a spend approval, and a proposal
all imply the Question and never print it. Minto still requires you to write it for yourself
(Ch. 4, p. 50).

**Signal: the Question will not come.** Then derive it backward (Ch. 4, p. 49). Read the
material you intend to place in the body. Ask why the reader should know those points, because
they answer a question. Ask why that question would arise, because his situation raised it.
Then write the introduction that gives your Question a logical provenance. **The Question form
fixes the plural noun of the Key Line.** "How?" answers with *steps* (Ch. 4, p. 51 and p. 53).
"Why?" answers with *reasons* (Ch. 3, p. 25).

---

## 5. Why the introduction must tell a story the reader already agrees with

**Introductions remind. They do not inform** (Ch. 4, theory point 1, p. 48). No exhibit belongs
in an introduction. Nothing may appear that the reader must be convinced of before he accepts
your points. **The rule runs in both directions** (Ch. 3, Caveat 5, p. 32). Do not place in the
introduction anything he does not know, because it distorts his Question. Do not place in the
body anything he already knows, because that implies you omitted something above.

**Agreement first makes the reader receptive.** Minto contrasts easy reading of agreeable
points against confused plodding through detail (Ch. 4, p. 36). Read
`03-the-packaging-elements.md` for the "we can all agree" opening, which is the same move.
**For a wide audience you plant the Question rather than recall it.** A named reader holds it
already. A report for wide circulation does not, so you arrange familiar material into a story
until the reader asks (Ch. 4, pp. 36–37).

**Minto names the framing effect and does not hide it.** The narrative flow lends plausibility
to a selection of facts that is necessarily biased. She compares the effect to a trial
lawyer's opening statement (Ch. 4, p. 59). **This skill treats the warning as a duty. Modern.**
Write the introduction so a dissenter can attack it. State the excluded candidates and the
reversal condition. **Historical chronology belongs in the introduction, never in the body**
(Ch. 3, Caveat 4, p. 32). Events belong in the body only inside a cause and its effect. **The
content is fixed. The order changes only the tone** (Ch. 4, "Why that order?", p. 40).

| Order | Sequence | Use it when… |
|---|---|---|
| **Standard** | Situation, Complication, Solution | The reader holds no strong prior position |
| **Direct** | Solution, Situation, Complication | The reader asked for the answer and wants it first |
| **Concerned** | Complication, Situation, Solution | The reader underrates the problem |
| **Aggressive** | Question, Situation, Complication | The reader disputes that the question matters |

**Worked example — the diversification memo** (Ch. 4, p. 40). Diversification work grew 40% in
five years, and no study shows a demonstrable client benefit. Minto prints that one memo four
ways, and the content never moves. **Always think from the Situation, whatever order you write**
(Ch. 3, Caveat 2, p. 31).

---

## 6. The Key Line

**The Key Line is the level of points directly under the Answer.** It answers the New Question
that the Answer raises, and it states the plan of the document (Ch. 3, p. 25, and Ch. 4,
p. 41). State the Key Line points inside the total introduction. The reader then holds your
whole argument in the first 30 seconds, and he can scan when time is short. Thinking that is
unclear in those 30 seconds means you rewrite (Ch. 3, p. 29). **Key Line points state ideas,
never categories.** Never write about categories, only about ideas (Ch. 4, p. 42).

**Worked example — the six-section setout** (Ch. 4, p. 42). A memo announces six sections:
Background, Principles of project team approach, What project work is, How the program is
organized, Unique benefits and specific results, and Prerequisites for success. Minto rejects
the list. It gives the reader words he cannot place, and it conceals that two of the six
sections belong under two others. A heading set of Overview, Options, Analysis, and Conclusion
carries the same defect. See entry X-28.

### The three shapes for a choice

| Shape | Form | Use it when… | Source |
|---|---|---|---|
| **Criteria-structured** | "Choose C. It meets criterion 1, 2, and 3." | C clears every criterion | Ch. 4, p. 55 |
| **Alternative by alternative** | The main reason for C, then against A, then against B | C wins overall and loses one criterion | Ch. 4, pp. 55–56 |
| **Alternative R2s** | "Choose A if you want X. Choose B if you want Y." | No option delivers the whole desired result | Ch. 4, p. 56, App. B, p. 225 |

**The criteria shape is preferred, and it carries a validity test.** If the winner loses on one
criterion, the criteria grouping is untrue, so change the shape (Ch. 4, pp. 55–56). **Label the
third shape as alternative objectives.** Minto insists it is not a set of alternative ways to
solve one problem (Ch. 4, p. 56). An unlabelled "it depends" is a defect, entry X-30. **Never
build a Key Line by elimination.** The reason for choosing C is that C solves the problem
(App. B, p. 225).

**Worked example — the motor alternatives memo** (Ch. 4, p. 55). A ruling names a different
motor size as the efficient one for cold-weather drilling, and the largest customer announces a
switch to a competitor. The company holds three responses: cut the price of the current motor,
reengineer a smaller motor to match, or design a new motor for the ruling. The Question is
which response makes the most sense. All three sit in the Complication, because the company
already discussed all three. The horsepower figures carry no part of the lesson, so this file
omits them. Read `09-the-tradeoff-evaluation.md` for the scoring matrix.

### Mini-introductions at each Key Line point

**Every Key Line point gets its own S-C-Q, much shorter than the first** (Ch. 4, p. 45). Each
position performs a different job (Ch. 4, p. 48).

1. The initial introduction reminds the reader what he knows about the subject.
2. The first Key Line point reminds him why this subject serves the overall point.
3. Each later point shows how its subject relates to the point he just read.

**The procedure is one question.** Ask what has just entered the reader's head, then ask what
else he must be told (Ch. 4, p. 48). The heading states the essence of the point, not the
topic. Minto rejects "BENCHMARKING" and uses "BENCHMARKING PROCESS EFFICIENCY" (Ch. 4, p. 47).

**Worked example — "Management Tools for the Nineties"** (Ch. 4, pp. 46–48, Exhibit 13). Total
Quality Management was the tool of the 1980s, adopted to cut cost and raise quality. Most large
companies adopted it without the expected benefit, while the leaders still hold or gain share.
The Question is why, and the Answer is that the leaders added Benchmarking and Activity-Based
Management to the tool kit. Each Key Line point then opens with its own small story. The first
supposes that the reader already cut a loan application from two days to two hours. It notes
that he probably assumes the cut is enough. Then it asks whether it is. This is the book's model for
correcting a plausible belief without contradicting the reader who holds it.

---

## 7. How long an introduction should be

**Length tracks the needs of the reader, not the length of the document** (Ch. 4, p. 43 and
p. 48). Minto's gallery of eight introductions runs from a letter to Gibbon's history, and a
book takes about as many paragraphs as a memo.

| The reader… | Length |
|---|---|
| Wrote you the question last week | One sentence is enough |
| Works with you daily on this system | Two or three paragraphs |
| Knows the subject and not this decision | Two or three paragraphs, plus the Key Line points |
| Knows neither the subject nor the problem | Situation and Complication of three or four paragraphs each |
| Anyone | Never more than four paragraphs for either element |

Source for the whole table: Ch. 4, "How Long a Story?", p. 42. **Signal that an introduction is
too long: it carries exhibits.** That means you are overstating the obvious. However short the
introduction becomes, it must still remind the reader of his Question.

---

## 8. The common patterns

Minto names four business patterns and maps each to a question (Ch. 4, "Some Common Patterns",
pp. 49–56). The last column translates them into software work. **Modern.**

| Pattern | Skeleton | Engineering instance |
|---|---|---|
| **Giving direction** | S = We want to do X. C = We need you to do Y. Q = How do I do Y? | A migration task assigned to another team |
| **Seeking approval to spend money** | S = We have a problem. C = We have a solution that costs an amount. Q = Should I approve? | A vendor contract, a paid tier, a cluster |
| **Explaining "how to"** | S = You have system X. C = It does not work correctly. Q = How do I make it work? | A runbook rewrite, an upgrade guide |
| **Choosing among alternatives** | S = We want to do X. C = We have alternative ways. Q = Which one makes the most sense? | The framework comparison. The default for Duty 1 |

**A directive plants the Question. It does not recall it** (Ch. 4, p. 50). In a directive the
Complication and the Answer reverse each other, because the Answer is the effect of performing
the actions that solve the problem (Ch. 4, p. 51). Use the reversal as a check. If your actions
do not produce the stated result, one of the two is wrong.

**Worked example — the field sales meeting directive** (Ch. 4, pp. 50–51, Exhibit 14). The
meeting will teach the reader to present a new program, and the writer needs information about
a problem chain in his area. The implied Question is how to supply it, so the Key Line holds
four dated steps: select a chain, prepare a profile, collect the data, return the data.

**The spend approval carries three standard reasons, sometimes four** (Ch. 4, p. 52). The
problem cannot wait. This action solves it, or it is the best of the alternatives examined. The
cost is more than offset. There are other benefits. Minto permits the fourth reason only where
the facts support it, and never as the reason to act. In her equipment funding request the
Answer is an explicit request for approval, not a summary of findings.

**A "how to" document compares the present process against the recommended one.** Draw both.
The differences dictate the Key Line steps (Ch. 4, pp. 53–54, Exhibit 15). An expert who omits
the drawing has a high chance of leaving something important out. Exhibit B-1 adds two
skeletons (App. B, p. 217). *Tell how to do something new* is `S = Must do X. C = Not set up to
do it. Q = How do we get set up?`. *Tell how it works* is `S = Have an objective. C = Installing
a system to reach it. Q = How does it work?`.

### The consulting patterns

Minto treats consulting documents separately, because they are longer and written to inspire
action (Ch. 4, "Some Common Patterns — Consulting", p. 57).

| Pattern | Skeleton | Body structure | Source |
|---|---|---|---|
| **Proposal, new client** | S = You have a problem. C = You decided to bring in an outsider. Q = Are you the outsider we should hire? | Four reasons | Ch. 4, pp. 57–58 |
| **Proposal, known client** | S = You have a problem. C = You want consulting help. Q = How will you help us solve it? | The steps of the approach | Ch. 4, p. 58 |
| **Progress review, first** | S = We said we would do X. C = We have now done X. Q = What did you find? | The findings | Ch. 4, p. 58 |
| **Progress review, later** | S = We told you X. C = You asked us to investigate Y, and we did. Q = What did you find? | The findings | Ch. 4, pp. 58–59 |

**Choose the shape by whether the situation is competitive.** Structure around reasons when you
must win the argument. Structure around steps when the reader already agreed to the work
(App. B, p. 222). The four reasons of the new-client proposal are fixed. We understand the
problem. We have a sound approach. We have experience in applying it. Our business arrangements
make sense (Ch. 4, p. 57). **Engineering form. Modern.** A build-or-buy case is competitive, so
structure it around reasons. A migration plan for a decision already taken goes to a known
client, so structure it around steps. A spike report is a progress review, and its Question is
always "What did you find?".

**Keep the cost and the staffing outside the structure of the thinking** (App. B, p. 222). They
belong in the document. They are not Key Line points. **Do not write to the standard proposal
heading set**, which Minto names and rejects: Introduction, Background, Objectives and Scope,
Issues, Technical Approach, Work Plan and Deliverables, Benefits, Firm Qualifications, and
Timing, Staffing, and Fees. Those lists overlap, and the overlap obscures the thinking (App. B,
p. 221). Read `07-logical-order-and-mece.md` for the exclusivity test.

---

## 9. Applied forms

The four forms below are this skill's translations. **Modern.** The skeletons are Minto's. The
settings are not.

### 9.1 The answer to a technical question

Use the Direct order. The reader asked, so he already holds the Question.

```
S = You run <component> at <pinned version>, at <path>.
C = You asked what <term> means for that component.
Q = (What is it, and why does it matter here?)
A = <The core idea. One sentence. Subject and predicate.>
```

The Situation cites a path, so the reader can check it. His own question needs no proof. Keep
the introduction to one or two sentences when he works beside you (Ch. 4, p. 42). Answer the
intent, not the literal words. A reader who asks "what is X" usually asks "why should I care
about X". Read `02-why-explanations-fail.md`, entry X-37.

### 9.2 The framework recommendation

Use the Choosing Among Alternatives pattern, in the Standard order.

```
S = We <do X today>, at <paths>, on <pinned versions>.
C = <The disturbing event>. We named <A>, <B>, and <the incumbent> as ways forward.
Q = Which one meets R2?
A = <Option>, because it meets every R2 threshold.
```

Four rules bind this form. The options sit in the Complication, and only options the reader
already named (Ch. 8, p. 135). An option that a MECE decomposition forced carries the label
"added by the consultant", per `07-logical-order-and-mece.md`. The Key Line takes one of the
three shapes above, never elimination, and its points sit under the Answer (Ch. 4, p. 41).

**Worked example — Colefax Supermarkets** (App. B, p. 220). A sales-based replenishment system
was conceived as a central mainframe system. All data entry and all major use happen at branch
level, so a question arose about a branch-based design, and a committee formed to settle it.
The introduction closes with the recommendation in one sentence. Minto uses the case as her
model of a legitimate two-option comparison, because the reader already held both options.

### 9.3 The incident summary

Use the fourth Exhibit 10 pattern. We took an action. The action did not work. Why not?

```
S = We <deployed / configured / changed> <thing> on <date> to reach <result>.
C = <The observed result>, from <the named signal>.
Q = (Why?)
A = <The cause, stated as a cause.>
```

Three rules apply. The Question "Why?" answers with *reasons*, so every Key Line point is a
reason (Ch. 3, p. 25). Describe only the process steps where a problem occurs (App. B, p. 234).
Never present a problem without a solution (App. B, p. 230).

**Worked example — the Period Graph Books memo** (App. B, pp. 230–234, Exhibits B-5 and B-6).
Two systems produced monthly graph books. The original memo narrates every step of both, and
Minto labels it over-described. She draws both process chains instead. She attaches each
problem to the step that causes it. The finding then collapses to one sentence. The system
produces unreliable graphs because errors occur at data entry, at the calculation of the graph
points, and when presenters change the graphs. Those causes become the Key Line.

**Route a live incident to another skill. Modern.** This skill is read-only. An outage belongs
to `incident-response`, and a fault that needs a cause belongs to `production-troubleshooting`
(see `11-routing-to-the-specialist-skills.md`).

### 9.4 The README opening

Use the "tell how it works" skeleton from Exhibit B-1 (App. B, p. 217), in the Standard order.
The audience is wide, so plant the Question rather than recall it (Ch. 4, pp. 36–37).

```
S = <The job this software exists to do, stated so any reader agrees.>
C = <What made that job hard before this software.>
Q = (How does this software do it?)
A = <One sentence. It names what the software does and the effect it produces.>
```

Three rules apply. Never open with a section called Background or Introduction, because Minto
prohibits both by name (Ch. 4, p. 42). The opening holds nothing the reader must accept on
faith, so no benchmark and no claim of maturity. Headings state ideas, so "Install it in one
command" beats "Installation". Read `04-assembling-and-delivering-an-explanation.md` next.

---

## 10. Faults, repairs, and where to read next

| Symptom | Diagnosis | Repair | Source |
|---|---|---|---|
| The opening argues, and it carries an exhibit | The introduction informs instead of reminds | Move the proof into the body | Ch. 4, p. 48 |
| The document answers a question nobody asked | The Complication does not raise the stated Question | Change the Question, or change the Complication | Ch. 3, p. 23 |
| "Should we do it, and if so, how?" | Two beginning Questions | Collapse them. The second becomes the Key Line | Ch. 4, pp. 48–49 |
| A heading names a topic, then the point arrives cold | The section holds no mini-introduction | Add the S-C-Q, and rewrite the heading | Ch. 4, p. 47 |
| The Key Line reads "A is bad, B is bad, so C" | Argument by elimination | "C, because it meets R2" | App. B, p. 225 |
| The record ends at "let us discuss how to proceed" | A problem with no solution | State the solution, or state R2 as the deliverable | App. B, p. 230 |
| The reader cannot see the argument in 30 seconds | The Key Line points are missing from the opening | State them under the Answer | Ch. 3, p. 29 |
| The winner loses one criterion under a criteria Key Line | The grouping is untrue | Change to alternative by alternative | Ch. 4, pp. 55–56 |

Each row also appears in `advice-defect-catalog.md`, in group B, C, or E.

| Read next | For |
|---|---|
| `05-the-pyramid-principle.md` | The structure below the Answer, and the question-answer dialogue |
| `07-logical-order-and-mece.md` | Whether the Key Line groups hold, and in what order |
| `08-defining-and-structuring-the-problem.md` | R1, R2, and the problem definition behind the Situation |
| `09-the-tradeoff-evaluation.md` | The R2 table, the criteria, and the scoring matrix |
| `01-the-explanation-scale.md` | How much the reader knows before you write the Situation |

**"Not applicable" is a valid result.** A two-sentence answer to a term with no decision
attached needs no S-C-Q. Read the proportionality section of the SKILL.md first.
