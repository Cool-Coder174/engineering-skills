# Why Explanations Fail

**Sources:** *The Art of Explanation* — Lee LeFever (Wiley, 2013), Ch. 3 "Why Explanations
Fail," sections "All About Confidence," "Assumptions Cause Failure," "Words Can Hurt," "We Lack
Understanding," "We Want to Appear Smart," and "The Direct Approach—No Context," pp. 23–32.
*The Minto Pyramid Principle* — Barbara Minto (Pearson, 3rd ed.), Ch. 1 "Why a Pyramid
Structure?", sections "The Magical Number Seven," "The Need to State the Logic," and "Ordering
from the Top Down," pp. 1–7. Two secondary citations carry a marker where they are used:
LeFever Ch. 4 "Planning Your Explanations," p. 38, and Ch. 6 "Forest then Trees," pp. 52–55.

LeFever names six failures. Minto names three more. LeFever explains why a reader stops trying.
Minto explains why a reader who is still trying reaches the wrong conclusion.

**The books describe memos, presentations, and video scripts. The reader of this file works on
software.** Every paragraph marked *Engineering form* is a translation by this skill, and it
reports no book. Any software object that both books predate carries **Modern**. Each failure
below comes from a competent author, so this skill needs a catalog and not a warning.

---

## 1. Confidence is the variable that fails first

**An explanation moves one variable, and that variable is confidence.** LeFever states it in one
line: "At heart, explanations are about affecting confidence." (Ch. 3, "Summary," p. 32.) A good
explanation raises it. A poor one removes it.

**A blank stare reports lost confidence, not lost comprehension.** Blank stares arise when a
person has lost confidence that they can grasp the idea, or that they should care about it
(Ch. 3, pp. 23–24). The person is still in the room and has stopped trying. The erosion is
difficult to reverse in that session. The audience then works toward the end of the explanation
instead of toward the idea.

*Engineering form (translation).* A written document returns no stare, so the consultant must
plan against this failure instead of watching for it. **Modern.** Three substitutes exist.

| Substitute signal | What it usually reports | First check |
|---|---|---|
| A reviewer approves a design and asks no question | The reviewer stopped at some paragraph | Find the first unglossed term |
| A team adopts a tool and uses one feature of it | The explanation reached one stepping stone | Ask what they think the tool does |
| An engineer repeats a question a week later | The first answer was a fact, not an explanation | Read entry F6 below |

**The author's own confidence is not the measure.** For a machine author, fluency and knowledge
are separate. **Modern.** See `10-research-method-and-sources.md`.

---

## 2. The six failures LeFever names

Each row expands into an entry below. Entry F7 is derived, and its source line says so.

| # | The book's own name | Signature in one line | Catalog code |
|---|---|---|---|
| F1 | Assumptions Cause Failure | One explanation serves a group whose levels nobody recorded | `X-09` |
| F2 | The curse of knowledge | You predict comprehension far above what you get | `X-09` |
| F3 | Words Can Hurt | One unglossed term inside an otherwise plain paragraph | `X-39` |
| F4 | We Lack Understanding | You wrote the word *explain* and cannot answer question two | `X-18` |
| F5 | We Want to Appear Smart | The document is accurate, well made, and aimed at the expert | `X-09` |
| F6 | The Direct Approach—No Context | A correct, short, category-noun answer | `X-37` |
| F7 | Detail before context | The text begins with parameters and never states the world | `X-38` |

LeFever places the curse of knowledge inside his section "Assumptions Cause Failure." This file
separates F1 and F2, because the cause and the mechanism need different remedies.

### F1 — The assumption of prior knowledge

**Signature:** one explanation serves a group. Nothing in the text records who it serves. The
register moves between paragraphs.

**Consequence:** LeFever calls the mismatch between your assumption and the reality "probably
the biggest reason explanations fail" (Ch. 3, p. 25). With a group you cannot measure each
person, so you must assume, and he states that the assumption will not match reality.

**The book's example.** LeFever asks the reader to imagine an explanation of type 2 diabetes in
front of a large audience (Ch. 3, pp. 24–25). What is basic for one listener is completely new
for the next. No single starting point fits the room, and the speaker cannot see which listener
is which. The mismatch is structural, not careless.

*Engineering form (translation).* A design document serves a staff engineer, a new hire, and a
product manager at once. An unglossed acronym appears in paragraph two and a first-principles
definition in paragraph six. **Modern.**

**Remedy:** write the audience line before the draft. Record four letters on the Explanation
Scale: the reader's letter, your letter, the letter the current documents assume, and the letter
you write for. Then decide whether this document serves the whole scale. LeFever makes that last
question a real branch (Ch. 4, p. 40). The method is in `01-the-explanation-scale.md`.

### F2 — The curse of knowledge

**Signature:** you predict that the reader will follow. The prediction is confident, and it is
wrong by a large factor.

**Consequence:** LeFever defines the curse as the difficulty of imagining what it is like not to
know a subject you know well (Ch. 3, pp. 25–26). He credits the term to *Made to Stick* by Chip
Heath and Dan Heath, and names it a failure of empathy. The curse also gets stronger as you move
toward "Z" on the Explanation Scale (Ch. 4, p. 38). So the person best placed to explain a tool
is the person least able to judge what the explanation needs.

**The book's example.** LeFever quotes the tapper and listener experiment that Elizabeth Newton
ran at Stanford in 1990 (Ch. 3, p. 25). One person taps the rhythm of a well-known song on a
table, and a second person names the song. Listeners identified 3 songs out of 120, which is 2.5
percent. The tappers had predicted 50 percent. The tapper hears the tune during the taps. The
listener hears only the taps.

*Engineering form (translation).* The engineer who wrote the service also writes its
documentation from the far right of the scale. The 2.5 percent figure is not a software
measurement, and this skill does not present it as one. Use it for the direction and the size of
your own error. **Modern** for the transfer.

**Remedy:** estimate the reader's letter, then write two letters lower. LeFever records that
decision as his own team's practice (Ch. 9, p. 95). Treat your estimate as a number you already
know to be too high.

### F3 — Jargon used as a shortcut

**Signature:** one unglossed term inside an otherwise plain paragraph. The term is ordinary
inside your team and unknown outside it.

**Consequence:** LeFever writes that "A single word can make your explanation fail because it
lowers confidence." (Ch. 3, p. 27.) One word can move a person from interest to disinterest. The
reader reads the unknown term as a risk, then discards the whole item.

**The book's example.** LeFever describes a restaurant menu with three dishes (Ch. 3, p. 27).
The third is crab cakes with mushrooms and a French rémoulade. The diner likes crab cakes and
mushrooms. He has never met the word *rémoulade*, so the dish feels like a risk. It falls to
the bottom of his list. LeFever then states that rémoulade is a lot like tartar sauce. The
connection existed. The menu never made it.

**Jargon is not the defect. An unprepared crossing is the defect.** Specialist language lets you
communicate with peers without any adjustment for their confidence level (Ch. 3, pp. 26–27). The
longer you live inside a culture and use its language, the more the curse of knowledge grows.

*Engineering form (translation).* The rémoulade word in a comparison table is a term such as
*idempotent*, *backpressure*, or *eventual consistency*. The reader moves that option down the
list for a reason unrelated to its merits, and never says so. **Modern** for the terms.

**Remedy:** run a list of foreign words against the finished draft. Each hit takes a plain
equivalent inline, a deferral, or a deletion. The tartar-sauce move is the model. Put the
familiar equivalent in the same sentence as the term. Catalog code `X-39`.

### F4 — We lack understanding

**Signature:** you read one source. You then wrote the word *explain*.

**Consequence:** a claim of knowledge makes you an authority, and that authority carries a
responsibility to be clear and accurate (Ch. 3, pp. 27–28). The gap opens at the first follow-up
question. At that moment you must admit ignorance, or you must invent. LeFever forbids the
invention.

**The book's example.** LeFever describes a news story about a new cancer drug (Ch. 3, p. 28).
You read it, you understand the basics, and you want to share them. Sharing is harmless. The
sentence "I can explain it to you" is not. The news gave surface information only, and the next
question leaves you with nothing. His remedy is one sentence: "The key to avoiding this
situation is to set expectations." (Ch. 3, p. 28.)

*Engineering form (translation).* The consultant reads one vendor page and writes a capability
claim with no version and no date. A beginner cannot detect the invention, because that
inability defines a beginner. **Modern.**

**Remedy:** write what the source says. Then write "the documentation does not cover X" as its
own sentence, and name the test that would settle it. Silence means *undetermined*, never *no*.
Catalog code `X-18`. The source ladder is in `10-research-method-and-sources.md`.

### F5 — We want to appear smart

**Signature:** the document is accurate, well made, and full of good information. It is aimed at
the most senior expert who will read it.

**Consequence:** LeFever writes that the goal of Dipika's presentation "was to make her look
smart, instead of helping others feel smart" (Ch. 3, p. 30). The people who must act on the work
do not understand it, and the author never learns this, because the expert was satisfied.

**The book's example.** Dipika, a new MBA, prepares a marketing strategy presentation for
engineers, designers, and executives (Ch. 3, pp. 29–30). She learns that the chief marketing
officer will attend, and her view of the meeting changes. She presents fluently, with charts and
the language of marketing, and the question period meets silence. Her boss calls the work
accurate and beautifully designed, and says many of her ideas were incomprehensible to most of
the room. His diagnostic question is the part to keep. Who were you thinking about?

*Engineering form (translation).* The consultant writes a comparison for the engineer who will
challenge it, not for the engineer who must implement it. Approval is not adoption. **Modern.**

**Remedy:** name the reader in the audience line, before the draft. Then aim to help every
reader feel smart. LeFever notes that this is also the durable way to impress the expert.

### F6 — Answering a question the person did not ask

**Signature:** a correct, short, category-noun answer to a question that asked for an
explanation.

**Consequence:** LeFever writes that "A statement of fact with no other context puts the onus on
the asker to take the next step." (Ch. 3, p. 30.) The asker does not take that step. He calls
the result a lost opportunity, and the answerer usually never learns that the answer failed.

**The book's example.** At a Silicon Valley conference in 2004 a chief executive speaks about
web trends and says "RSS" three times (Ch. 3, pp. 31–32). An older engineer raises his hand and
asks what RSS is. The answer names an XML-based content syndication format. The hand goes down
and the speaker continues. LeFever calls the answer 100 percent accurate, and says it made the
engineer feel less confident that RSS was something he could understand. LeFever supplies the
rewrite. RSS makes it easy to subscribe to websites so that new content comes directly to you.

**The rule under the rewrite.** "They answer questions like 'What is this?' as if the question
was, 'Why should I care about this?'" (Ch. 3, p. 31.) The fault is a preference for efficiency
over understanding. LeFever does not ban the direct answer. He states that direct answers are
often needed and well placed, and that they do not work universally (Ch. 3, pp. 30–31).

*Engineering form (translation).* An engineer asks what a message queue is, and the consultant
answers that it is a durable FIFO buffer with at-least-once delivery. Every word is true, and
the engineer can decide nothing. **Modern.**

**Remedy:** answer the intent first. Then give the fact. Section 6 holds the admission test.
Catalog code `X-37`. The packaging order is in `03-the-packaging-elements.md`.

### F7 — Detail before context

LeFever names the missing context in Ch. 3, p. 30. He develops the failure in Ch. 6, "Context,"
section "Forest then Trees," pp. 52–55. This entry cites Ch. 6 for the development.

**Signature:** the text begins with terms, parameters, or components. It never states the world
those details operate in.

**Consequence:** the reader memorizes, cannot apply, and decides that the fault is their own.

**The book's example.** Angela changes career and takes an accounting workshop (Ch. 6,
pp. 53–55). Mr. Tidwell opens with credits, debits, revenues, and expenses. Angela memorizes the
terms, cannot apply them, and panics at the financial statements. She concludes that she is not
smart enough. In a second workshop Ms. Stowe asks the class about their own business experience,
then teaches how a business runs, how money moves, and what makes a profit. Angela does not hear
the word *debit* for hours. The debits and the credits arrive later, inside the context of a
business, and they land. The content did not change. The order changed.

**Remedy:** state the world before the parts. Section 4 names three engineering instances, and
a README that opens with installation is the clearest one. Catalog code `X-38`.

---

## 3. Minto: the reader must receive the idea before the support

LeFever explains why an audience stops trying. Minto explains why an audience that is still
trying arrives somewhere else. Her three failures are cognitive, and they apply to a fully
engaged reader.

### M1 — More items than the mind holds

**Signature:** a list of six or more items with no stated category above them.

**The rule.** Minto cites the paper "The Magical Number Seven, Plus or Minus Two" by George A.
Miller (Ch. 1, "The Magical Number Seven," p. 3). The mind holds about seven items in short-term
memory at one time. Some minds hold nine. Some hold five. Minto says three is a convenient
number, and one is the easiest number. Above four or five items, the mind starts to build its
own categories. Read that threshold as a warning. The reader will group your items whether or
not you name the grouping, and the grouping the reader builds is the one you are judged on.

**The book's example.** A husband walks toward the door for a newspaper (Ch. 1, pp. 3–4). His
wife adds items one at a time as he crosses the room. Nine items in total: grapes, milk,
potatoes, eggs, carrots, oranges, butter, apples, and sour cream. Minto writes that most men
return with the newspaper and the grapes. She then gives the repair. Read the same list, and
assign each item to a section of the supermarket as you meet it. Three categories replace nine
items, and the reader keeps all nine.

*Engineering form (translation).* A comparison matrix with nine criteria fails this way, and so
does a design document with seven numbered risks and no grouping. The reader keeps two. This
skill caps a grouping at three to five criteria. **Modern.** See `07-logical-order-and-mece.md`.

**Remedy:** group the items, then name the group.

### M2 — Grouping without a summary

**Signature:** the items carry headings, and every heading is a category noun. Nothing states
what the group means.

**The rule.** Minto states the arithmetic (Ch. 1, "The Need to State the Logic," p. 4). Split
nine items into sets of four, two, and three, and the reader still faces nine items. You gain
nothing until you name the three categories. The name moves you one level of abstraction higher,
and the higher thought then suggests the items below it.

*Engineering form (translation).* A section titled "Trade-offs" holds four paragraphs. A
section titled "Postgres meets every threshold except cost at 10k writes per second" holds the
same four paragraphs and delivers an idea. Minto calls the first an intellectually blank assertion.
**Modern** for the example. Catalog code `X-28`.

**Remedy:** state what the group means. An unstatable grouping is an unfinished grouping.

### M3 — Support delivered before the idea

**Signature:** the document presents evidence in the order the author found it. The verdict
arrives at the end.

**The rule.** "Controlling the sequence in which you present your ideas is the single most
important act necessary to clear writing." (Ch. 1, "Ordering from the Top Down," p. 5.) The
clearest sequence gives the summarizing idea before the ideas it summarizes. The reader searches
for a structure that connects the ideas as they arrive (Ch. 1, p. 6). Without a supplied
structure the reader builds one, and readers rarely build the author's. Her second example is
the worse one. She reproduces the five opening points of an article about equal pay (Ch. 1,
pp. 6–7). The connecting relationship is unclear, so the mind searches for a relationship,
concludes there is none, and stops.

**The book's example.** A woman describes a trip over a beer (Ch. 1, pp. 5–6). In Zurich she
counted fifteen men with a beard or a moustache in fifteen minutes. In New York offices she saw
sideburns and moustaches. Facial hair has been part of the London scene for years. The listener
guesses at every step. First, that Zurich is becoming unconservative. Then, that she will
compare cities. Then, that she is interested in beards. His conclusion is that London leads the
other cities. Minto calls that conclusion perfectly logical and wrong. Her actual point was the
degree to which facial hair has become accepted in business life.

**The reader's budget.** A reader, however intelligent, holds a limited amount of mental energy
(Ch. 1, p. 7). Some energy interprets the words. More energy sees the relationships between the
ideas. Whatever remains comprehends their significance.

*Engineering form (translation).* A benchmark table with no verdict above it invites the Zurich
conclusion. Every reader builds a different recommendation from the same cells, and every one of
them is perfectly logical. **Modern** for the example.

**Remedy:** state the Answer first. The structure is in `05-the-pyramid-principle.md`. The
opening that carries it is in `06-the-scqa-introduction.md`.

---

## 4. The engineering forms, named

Six software artifacts fail in the exact shapes above. This section is a translation. Neither
book names a README, an API reference, or a pull request. **Modern** throughout.

| The artifact | The failure it instantiates | Why it happens | The repair |
|---|---|---|---|
| A README that opens with installation | F7, detail before context (LeFever Ch. 6, pp. 52–55) | The maintainer sits at "Z", and installation is their own first step | One paragraph above it. What problem this solves, and for whom |
| An answer that starts at the implementation | F6, the direct approach (LeFever Ch. 3, pp. 30–32) | The answerer prefers efficiency, which LeFever names as the fault | Answer "why should I care", then give the mechanism |
| A term used before it is defined | F3, the rémoulade word (LeFever Ch. 3, p. 27) | The term is ordinary inside the team, so the author cannot see it | A plain equivalent inline, a deferral, or a deletion |
| A comparison matrix with nine criteria | M1, the Magical Number Seven (Minto Ch. 1, p. 3) | Every criterion was easy to test, so the author cut none | Group into three to five. Name each group |
| A results table with no verdict | M3, support before the idea (Minto Ch. 1, pp. 5–7) | The author presents the work in the order of the work | State the Answer above the table |
| A heading that reads "Options" or "Analysis" | M2, grouping without a summary (Minto Ch. 1, p. 4) | The author grouped and then stopped | Every heading states its claim |

**The README case needs one more sentence.** Installation is a *description* in LeFever's
vocabulary. It answers *how*, and it belongs toward the "Z" end of the scale. First position is
the defect, because the reader who needs the forest receives a tree.

---

## 5. Diagnosis: read the signal, then choose the entry

Read down the table. Stop at the first row that matches what you observe.

| The signal you observe | The likely failure | First repair | Read |
|---|---|---|---|
| The reader approves and asks nothing | Lost confidence, section 1 | Find the first unglossed term | `01-the-explanation-scale.md` |
| The reader repeats a question you answered | F6, the direct approach | Answer the intent, not the words | This file, entry F6 |
| The reader repeats your words and cannot apply them | F7, detail before context | Give the world before the parts | `03-the-packaging-elements.md` |
| The reader follows every point and cannot decide | M3, support before the idea | State the Answer first | `05-the-pyramid-principle.md` |
| Two reviewers reach different conclusions from one table | M3, the Zurich failure | State the Answer first | `06-the-scqa-introduction.md` |
| The reader keeps two of your nine findings | M1, the Magical Number Seven | Group into three to five | `07-logical-order-and-mece.md` |
| A section heading names a category | M2, grouping without a summary | Rewrite the heading as a claim | `07-logical-order-and-mece.md` |
| Your draft cannot answer an obvious next question | F4, we lack understanding | Write **undetermined**. Name the test | `10-research-method-and-sources.md` |
| The expert is satisfied and the implementer is not | F5, we want to appear smart | Rewrite the audience line | `01-the-explanation-scale.md` |
| A comparison is requested, and the incumbent meets the requirement | An explanation problem, not a selection problem | Run the explanation duty. Say so in paragraph one | `09-the-tradeoff-evaluation.md` |

**The last row is the most expensive one to miss.** The team pays for a migration, and the new
tool arrives with the old misunderstanding attached. Catalog code `X-08`.

---

## 6. When these are not failures

**A short answer is not always the direct approach.** LeFever states that direct answers are
often needed and well placed (Ch. 3, pp. 30–31). Use this table before you expand an answer.

| The situation | Give the direct answer? | Because |
|---|---|---|
| The asker already runs the thing and wants a parameter | Yes | The reader sits near "Z" and needs *how*, not *why* |
| The asker is inside the team and inside its language | Yes | LeFever defends in-bubble specificity as correct (Ch. 3, pp. 26–27) |
| The asker used the term correctly in their own question | Yes | Vocabulary is evidence of position, not proof of it. Verify once |
| The asker asked "what is X" and has never used X | No | Answer the intent behind the question. LeFever Ch. 3, p. 31 |
| The answer holds a term the asker has not used | No | One unknown word lowers confidence. LeFever Ch. 3, p. 27 |
| A decision depends on the answer | No | Without the reason, the reader cannot check your verdict |

**A review of the basics costs almost nothing.** LeFever prices it as a negative impression, and
concludes that the cost of context is low and the benefit is high (Ch. 6, p. 60). Repetition
confirms what an informed reader holds. Omission excludes a beginner completely.

**"No failure found" is a valid result.** Do not invent a defect to fill a scan. An explanation
that names its audience, states its idea first, glosses its terms, and cites its claims passes
this file. Report that it passes.

---

## 7. What this file does not cover

| Omitted here | Where it lives |
|---|---|
| The Explanation Scale, the two-letter down-shift, and the audience line | `01-the-explanation-scale.md` |
| The six packaging elements, from Agreement to Conclusion | `03-the-packaging-elements.md` |
| The writing process, the constraints, and the visual choice | `04-assembling-and-delivering-an-explanation.md` |
| The Key Line, the vertical dialogue, and the three rules of a grouping | `05-the-pyramid-principle.md` |
| Situation, Complication, Question, Answer | `06-the-scqa-introduction.md` |
| MECE, logical order, and the intellectually blank assertion | `07-logical-order-and-mece.md` |
| R1, R2, the issue analysis, criteria, and thresholds | `08-defining-and-structuring-the-problem.md`, `09-the-tradeoff-evaluation.md` |
| Source precedence, versions, dates, and **undetermined** | `10-research-method-and-sources.md` |
| The catalog entries, their severities, and the routing table | `advice-defect-catalog.md`, `11-routing-to-the-specialist-skills.md` |

**One boundary the consultant defends.** This file repairs a failure of understanding. The work
that follows the repair routes to the owning skill. The consultant never writes the code.
**Modern.** Neither book contemplates a read-only charter.
