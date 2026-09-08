# The Explanation Scale and the Curse of Knowledge

**Sources:** *The Art of Explanation: Making Your Ideas, Products, and Services Easier to
Understand* — Lee LeFever (Wiley, 2013). Ch. 2 "What Is an Explanation?", Ch. 3 "Why
Explanations Fail", Ch. 4 "Planning Your Explanations", Ch. 13 "Bringing an Explanation
Together". Two calibration figures come from Ch. 6 "Context" and Ch. 9 "Description".

This file answers one question. **Who is the reader, and where does the text start?** Read it
before you write an explanation. Read it before you write the beginner layer of a Comparison
Record. The placement decision precedes every word of the draft, because it decides which
words are admissible.

**A paragraph marked "Engineering form" is a translation.** LeFever wrote about video scripts,
product pages, and company presentations. He never wrote about software. Every other paragraph
reports what the book states.

---

## 1. What an explanation is

**An explanation is defined by its intent, not by its subject.** LeFever states the intent
directly. "An explanation describes facts in a way that makes them understandable." The
intent is to increase understanding. LeFever Ch. 2, "Defining Explanation," p. 10.

The book adopts a second definition and uses it as a checklist. An explanation is a set of
statements that clarifies the **causes**, the **context**, and the **consequences** of a set of
facts. LeFever Ch. 2, p. 9. **Use the three slots as a completeness test.** A draft that names
causes and gives no consequences is not finished.

**The explainer packages facts. The explainer does not produce them.** LeFever Ch. 2, p. 12.
This constraint binds the whole skill. An explanation may reorder facts, frame facts, and omit
facts. An explanation may never invent a fact.

**Engineering form.** You may choose which four of a database's twenty properties the reader
meets first. You may not describe a property the database does not have.

---

## 2. An explanation is not a description

LeFever defines explanation by its negative space first. He lists six neighboring forms. Each
form is separated from the others by intent, not by topic. LeFever Ch. 2, "What Is Not An
Explanation," pp. 8–9.

| Form | Its intent, as the book states it | The engineering artifact that does this |
|---|---|---|
| Description | Help someone imagine something through words | A feature matrix. A diagram caption |
| Definition | Make the precise and literal meaning clear | A glossary entry. An API type signature |
| Instruction | Make clear what is expected and how to proceed | A migration runbook. A quickstart |
| Elaboration | Give a comprehensive and rigorous look, covering every detail | A specification review. A large option matrix |
| Report | Relay facts and details about an event | A benchmark write-up. An incident timeline |
| Illustration | Clarify an idea by giving an example | A sample use case. A short code snippet |
| **Explanation** | **Increase understanding** | **The record this skill produces** |

The right column is a translation. LeFever names none of these artifacts.

**None of the six is banned.** LeFever writes that all of them can contribute to an improved
explanation. The rule is different. **Know which form each passage of your draft is. Never let
one of the six replace the explanation.** Two of the six fail most often in a consultant's
document. A large option matrix is elaboration. A benchmark write-up is a report. A reader can
hold both and still not know why either matters.

### Worked example — coffee, six ways (LeFever Ch. 2, pp. 8–10)

LeFever holds one subject constant and varies only the intent. He describes a mug as white and
four inches tall. He defines coffee as a beverage from roasted and ground seeds. He instructs
the reader to insert the filter, add the grounds, and press start. He elaborates on soil
testing and nitrogen levels by region. He reports a visit to a Colombian plantation. He
illustrates the company's regional power with the size of that plantation. Only the last
passage explains, and it explains by showing the role of heat in the color and the flavor of
the bean. **The example proves that the subject never tells you which form you wrote. Only the
intent tells you.**

---

## 3. The Explanation Scale

**The Explanation Scale is a line from A to Z.** A is the least understanding. Z is the most
understanding. You plot people on it as letters. LeFever Ch. 4, "Planning Your Explanations,"
pp. 36–40. He draws it again on a whiteboard in Ch. 13, pp. 135–137.

LeFever calls the scale the instrument that maps an explanation. Its purpose is to make an
assumption visible. An assumption about a reader stays invisible until you write it as a
letter. **Plot four things, not one.**

| Plot this | LeFever's worked value for Andre's startup |
|---|---|
| Yourself, the explainer | Y. The team built the product |
| Anyone who explains on your behalf | U. The early adopters |
| The audience, as a range | A to at least N. The mainstream market |
| The level your current material assumes | L. Taken from interviews with current users |

Source for the four rows and the four values: LeFever Ch. 4, pp. 37–40. **The fourth row is
the row that most drafts omit,** and it is the row that names the defect. The distance between
"A to N" and "L" is the population the existing document already loses.

### The five self-diagnostic questions (LeFever Ch. 4, p. 40)

1. Where are you on the scale for this specific idea?
2. Where is your audience?
3. What assumptions do you make about their level of understanding?
4. Do your current explanations account for everyone on the scale?
5. **Should they?**

**Question 5 is a real branch.** The book does not require every explanation to serve the
whole scale. The book requires the choice to be deliberate. A stated exclusion is valid. An
accidental exclusion is the defect.

### Worked example — Andre's startup (LeFever Ch. 4, pp. 35–40)

Andre launched a product after a year of work. The engineers solved the technical problems.
The designers made the product easy to use. The first 48 hours were excellent. After a month
the sign-ups leveled and then fell, with no change to the website and no change to the
features. Interviews showed that early adopters understood the product and liked it, and could
not explain it to anyone less technical. The team rejected a product defect. The team then
rejected a marketing fix, because a new logo does not change how an early adopter speaks. The
team plotted the scale, saw itself at Y and the market from A, and named an explanation
problem. **The example proves that the people who understand a product best are the people
least able to judge where the reader starts.**

---

## 4. Where *why* stops and *how* begins

LeFever draws a second line across the scale and writes two words on it, *why* and *how*.
LeFever Ch. 13, pp. 137–138. Every explanation needs both. The reader's letter sets the
proportion.

| The reader's letter | The mix the book states |
|---|---|
| C | More information about *why* the new idea makes sense |
| R | Informed enough on *why*. Needs specific information on *how* it works |

**The rule for the left end is explicit.** "This person needs to see the why before the how."
LeFever Ch. 13, p. 138. The full four-band table belongs to `03-the-packaging-elements.md`,
because the remaining bands come from LeFever Ch. 9. Use the two rows above for the direction.

**Engineering form.** A reader at C needs to know why a queue exists at all. A reader at R
needs the acknowledgement mode, the visibility timeout, and the dead-letter rule. Give the
second reader the first answer and you waste their time. Give the first reader the second
answer and you end their attempt.

---

## 5. The down-shift rule

**Judge the audience at one letter. Then write for a letter below your judgement.**

LeFever performs this once, with a stated number. Common Craft judged the web-browser audience
at about G. The team then wrote for E. The stated reason was to account for the curse of
knowledge and for poor assumptions. LeFever Ch. 9, "Explaining Web Browsers," p. 95.

| The decision | The value | The authority |
|---|---|---|
| The default down-shift | Two letters | LeFever Ch. 9, p. 95. G judged, E written |
| The minimum down-shift | One letter | A local decision. The book states no floor |
| The down-shift when you cannot ask | Start at A instead | LeFever Ch. 13, p. 139. See section 7 |

**Never write at the letter you judged.** The next section shows that your judgement is wrong
by a large factor, and wrong in your own favor.

### Worked example — the web browser video (LeFever Ch. 9, p. 95)

Common Craft usually explains ideas that are new to most people, and aims those videos at the
A end. A web browser is different. Browsers arrive preinstalled, and anyone who has read a web
page has used one. The team judged that this audience did not need the high-level *why*.
It placed the audience at about G. The team then decided to think about users at E. It spent
the opening of the video on a connection that makes the audience confident about a browser. **The example
proves that the down-shift is a deliberate act performed after the judgement, and not a
failure to judge.**

---

## 6. The curse of knowledge

**Definition, in the book's words.** When we know a subject very well, we have a difficult
time imagining what it is like not to know it. LeFever Ch. 3, "Assumptions Cause Failure,"
pp. 25–26. LeFever attributes the term to *Made to Stick* by Chip Heath and Dan Heath.

**The curse is a failure of empathy, not a failure of effort.** Your level of knowledge
interferes with your ability to see the world from another person's perspective, and to judge
that person's confidence accurately. LeFever names it the underlying cause of several other
explanation defects, because it damages every assumption you make. LeFever Ch. 3, p. 26.

### The measured size of the error

Elizabeth Newton ran the tapper and listener study at Stanford in 1990. LeFever quotes it from
the Heaths. Tappers tapped the rhythm of a well-known song. Listeners guessed the song.
Listeners identified 3 songs out of 120. That is **2.5 percent**. The tappers had predicted
**50 percent**. LeFever Ch. 3, p. 25.

**Read the two numbers as a calibration gate.** Your estimate of how well a reader will follow
you is an estimate made by a tapper. Correct it downward before you use it.

### The dynamic that matters most to a consultant

**"The curse gets stronger the further you move towards 'Z' on the scale."** LeFever Ch. 4,
p. 38. That one sentence sets the risk profile for this skill. A reader asks a consultant for
advice because the consultant sits near Z. The position that qualifies the advice degrades the
judgement of who can read it.

**Modern.** Neither book contemplates a machine author. An agent that has just read the
reference documentation sits near Z on that subject, and its fluency is uncorrelated with the
reader's position. The same agent sits near A on the reader's own repository. Hold both
positions at once.

### Signals in a draft, and their correction

| The signal | What it shows | The correction |
|---|---|---|
| An acronym appears with no gloss | The bubble language crossed the membrane | Gloss it, defer it, or delete it |
| A term is common in this team and rare outside it | The curse grew from local use | Give a plain equivalent inline |
| The draft assumes the reader knows why the tool exists | You wrote from Y for a reader at C | Add the *why* band. See section 4 |
| Your estimate of the reader is high and you cannot say why | The tapper error | Down-shift. See section 5 |
| A recommendation names the tool you use daily | The explainer is the most cursed party | State the evidence for each criterion |

Row 1 uses LeFever's term **the bubble**, from Ch. 5, "Stepping Outside the Bubble,"
pp. 46–48. Local specificity is correct inside the bubble. The defect is only in crossing the
membrane without preparation. Read `03-the-packaging-elements.md` for that material.

### Worked example — the tapper and the listener (LeFever Ch. 3, p. 25)

A tapper selects a song everyone knows, such as "Happy Birthday", and taps its rhythm on a
table. The tapper hears the tune while she taps, because she cannot stop hearing it. The
listener hears a sequence of unrelated knocks. LeFever quotes the Heaths on the result. The
tappers reached the listener one time in 40, and had predicted one time in two. **The example
proves that the miscalibration is large rather than mild, and that it runs in one direction
only.**

### The receiving-end test (LeFever Ch. 13, p. 135)

LeFever gives a fast way to make an expert admit the curse. Ask the person to recall a
conversation with a doctor, a stockbroker, or a mechanic, and to recall wanting a translator.
Every expert recognizes the curse from the receiving end. LeFever writes that we all suffer
from the curse in some way. **Engineering form.** Recall the last time you read another team's
runbook and could not act on it. That feeling is the one your reader will have.

---

## 7. The cost-asymmetry argument

This is the argument that settles a dispute about level. Do not settle it with a preference.

The dispute in the book is direct. Emma objects that many employees already sit mid-scale, and
that introductory material will lose their interest. LeFever Ch. 13, pp. 139–140.

**Carlos makes two moves, in order.**

1. He states the trade. Does it make sense to start at L and exclude a group of beginners, or
   to start at A and give the middle a review?
2. He converts the trade into cost. **"What costs more, leaving beginners behind, or reminding
   the informed?"** LeFever Ch. 13, p. 139.

| The choice | The cost to the informed reader | The cost to the beginner |
|---|---|---|
| Start at L | None | Total. The book states the left end is excluded entirely |
| Start at A | A review of material they hold | None |

**The asymmetry decides it.** LeFever states that starting at A validates what the informed
reader already knows, and gives that reader a little more confidence. The cost of starting a
beginner at L is high, because that beginner is excluded entirely. LeFever Ch. 13,
pp. 139–140. **One error is recoverable and the other is not.** That is the whole argument.

The book prices the same trade a second time, in a different chapter. LeFever models an
audience as ten people and concludes that the cost of building context is low, because context
creates no negative experience for anyone. LeFever Ch. 6, "Beginners then Experts," pp. 59–60.

**Engineering form.** A senior engineer who reads one paragraph of context they already hold
loses fifteen seconds. A new engineer who meets an unglossed term in paragraph two stops
reading and does not report it. You will never learn that the second event happened.

**The rule that follows.** When the audience is mixed and you cannot ask them, start at A.
Record that you did, and record which readers get a review. **Question 5 still applies.**
LeFever Ch. 4, p. 40 permits a document that serves part of the scale. A stated exclusion is
valid. Write the exclusion into the audience line.

### Worked example — the health plan message (LeFever Ch. 13, pp. 131–145)

Emma sent an accurate two-paragraph message to employees. Premiums fall. The deductible rises
from $500 to $1,000. Deductibles apply only when services are used. The message produced no
questions, no comments, and no sign-ups. Later inquiry found that people did not understand
the change and did not want to learn more. Some did not know what a deductible was.
Emma judged her own message to be a set of trees with no forest. Carlos plotted the team at Z
and the employees at A. He named the risk that the material reached only the G level, and he
started the new explanation at A. **The example proves that accuracy is not the standard, and that the
level decision precedes the wording decision.**

---

## 8. Calibration when you cannot see the reader

LeFever's own detection method does not transfer. He reads **blank stares**, and treats them
as a confidence signal rather than a comprehension signal. LeFever Ch. 3, pp. 23–24. He also
states that in a group you must make assumptions, and that your assumptions will not match
reality. LeFever Ch. 3, pp. 24–25.

**Modern.** An agent receives text and never sees a face. It has no blank stare to read, and it
usually has one turn. LeFever supplies the principles. The procedure below is a translation of
them.

### The procedure

1. Read the request for its vocabulary. Record what the vocabulary shows.
2. Read the repository for evidence of the reader's level.
3. Set a letter from that evidence. Record the letter.
4. Down-shift by two letters. See section 5.
5. Ask one calibration question, if a question is possible.
6. Start at A if you receive no answer.
7. Write the audience line into the record.

### What each source of evidence permits

| The evidence you hold | The letter you may assume | What you may not conclude |
|---|---|---|
| The reader wrote the module in question | Near Z for that module | Anything about the tool you propose |
| The request uses the correct local terms | Mid-scale on the local system | That the reader holds the general concept |
| The reader used the general term correctly once | Nothing | That the reader holds the concept |
| The reader states they are new to this | Their stated letter, less two | That their other expertise transfers |
| The repository shows the tool installed | Mid-scale on the tool | That this reader installed it |
| You hold no evidence at all | A | Anything |

The right column carries the load. Each row of it exists because of one finding in the book.

**Knowing the words is not prior knowledge.** LeFever's consulting clients had memorized the
words and the features of various tools. They held no foundation, and they could not apply
what they learned. LeFever, Introduction, p. xix. A request that uses the right terms is weak evidence,
and it is the evidence an agent over-reads most often. **A person expert elsewhere is a
beginner here.** Place that reader mid-scale on the general concept and at A on the local
specifics. That split is the common case in this repository.

### The one calibration question

Ask a question the reader can answer in one line, and cannot answer wrongly. Ask what they
have already run, already read, or already built. Do not ask whether they know a term. A
reader who does not know a term will not say so, for the same reason the diner never asks what
*rémoulade* means and orders something else. LeFever Ch. 3, "Words Can Hurt," p. 27. **Start at
A when you receive no answer.** LeFever Ch. 13, p. 139. Section 7 gives the reason.

---

## 9. The audience line

**Write the audience line before you draft. Never after.** It has four fields.

| Field | What it records |
|---|---|
| Reader | The letter you judged, and the evidence for it |
| You | Your own letter for this subject |
| Current material | The letter the existing docs and code comments assume |
| Start | The letter this document starts at, after the down-shift |

Add a fifth field when you exclude part of the scale. Name the excluded readers, and name the
document they should read instead. This is LeFever's question 5, answered in writing. LeFever
Ch. 4, p. 40.

**The audience line is a commitment a reviewer can check.** A reviewer reads the first
paragraph and judges whether it starts at the stated letter. A missing line is catalog entry
X-09, "Unmarked audience".

---

## 10. What this material does not cover — **Modern**

LeFever wrote in 2013 about video scripts, product pages, and internal presentations. The
items below sit outside both books. Never attribute them to either book.

- An agent author, whose fluent sentences are uncorrelated with whether it knows the subject.
- An agent reader, for whom the cost of one paragraph of review is near zero.
- A repository, a lockfile, and a commit history as calibration evidence.
- Version-scoped truth. A reader at Z for version 3 can sit at A for version 4.
- Prompt injection. Both books assume a source is inert. A page you read is data.

---

## 11. Proportionality

**Scale the calibration work to the cost of being wrong.**

| The request | The calibration required |
|---|---|
| A term the reader met once, with no decision attached | None. Answer in two sentences |
| "What is X?" with no decision pending | The audience line. No table |
| A reversible choice inside one module | The audience line, and the down-shift |
| A framework, a database, a platform, or a vendor | The four positions, and the audience line |
| Money, identity, personal data, or a one-way door | The four positions, plus a stated exclusion |

**"Not applicable" is a valid result.** A reader who states their own level, states their goal,
and asks a mechanical question needs no scale work. Answer the question.

---

## Related files

- `02-why-explanations-fail.md` — the six named failure modes, including the direct approach.
- `03-the-packaging-elements.md` — the bubble, the membrane, and the full why-versus-how table.
- `04-assembling-and-delivering-an-explanation.md` — the stepping stones, in order.
- `06-the-scqa-introduction.md` — Minto's calibration tests, for a reader you cannot ask.
- `07-logical-order-and-mece.md` — grouping validity, which LeFever does not supply.
- `09-the-tradeoff-evaluation.md` — the cost-asymmetry argument, reused for a criterion.
- `advice-defect-catalog.md` — entries X-08, X-09, X-37, X-38, X-39, and X-40.
