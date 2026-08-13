# Intrusion Detection and Audit

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., §18.1 (Intruders, p. 567), §18.2
(Intrusion Detection, p. 570), and Appendix 18A (The Base-Rate Fallacy, p. 595).

Chapter 1 divides attacks into passive and active and says prevention is realistic for the former,
detection for the latter (`01-threat-model-and-security-services.md` §2). This is the chapter that
takes detection seriously. Two things in it are directly useful in code review: the structure of an
audit record — six fields, each with a stated reason — and the base-rate argument, which is the
mathematical reason most alerting is ignored.

The reason this belongs in a *code* review and not only an operations review is that **detection is
built at write time.** If the code does not emit the event, with the fields needed to answer a
question, no amount of downstream tooling recovers it.

---

## 1. Detection exists because prevention fails (§18.2, p. 570)

> "**Inevitably, the best intrusion prevention system will fail.** A system's second line of defense is
> intrusion detection." (§18.2, p. 570)

Four reasons, in Stallings' order:

> "If an intrusion is detected quickly enough, the intruder can be identified and **ejected from the
> system before any damage is done** or any data are compromised." (§18.2, p. 570)

> "Even if the detection is not sufficiently timely to preempt the intruder, **the sooner that the
> intrusion is detected, the less the amount of damage and the more quickly that recovery can be
> achieved.**" (§18.2, p. 570)

> "An effective intrusion detection system can **serve as a deterrent**, so acting to prevent
> intrusions." (§18.2, p. 570)

> "Intrusion detection enables the **collection of information about intrusion techniques** that can be
> used to strengthen the intrusion prevention facility." (§18.2, p. 570)

The first two are the ones that turn detection into a design requirement rather than a nice-to-have:
damage is a function of *dwell time*, and dwell time is a function of what you can see. The fourth
closes the loop — detection feeds prevention, which is why the incident review that produces no code
change is a wasted incident.

And the asymmetry that frames the whole problem:

> "**The difficulty stems from the fact that the defender must attempt to thwart all possible attacks,
> whereas the attacker is free to try to find the weakest link in the defense chain and attack at that
> point.**" (§18.1, p. 570)

---

## 2. Who you are detecting, and why it matters which (§18.1, p. 567)

> "**Masquerader:** An individual who is not authorized to use the computer and who penetrates a
> system's access controls to exploit a legitimate user's account" (§18.1, p. 567)

> "**Misfeasor:** A legitimate user who accesses data, programs, or resources for which such access is
> not authorized, **or who is authorized for such access but misuses his or her privileges**" (§18.1,
> p. 567)

> "**Clandestine user:** An individual who seizes supervisory control of the system and uses this
> control to **evade auditing and access controls or to suppress audit collection**" (§18.1, p. 567)

> "The masquerader is likely to be an outsider; the misfeasor generally is an insider; and the
> clandestine user can be either an outsider or an insider." (§18.1, p. 567)

These three are not a taxonomy for its own sake — each is detectable by different means, and two of
them are barely detectable at all:

> "Anderson suggests that the task of detecting a misfeasor (legitimate user performing in an
> unauthorized fashion) is more difficult, in that **the distinction between abnormal and normal
> behavior may be small.** Anderson concluded that such violations would be **undetectable solely
> through the search for anomalous behavior.**" (§18.2, p. 571)

> "Finally, the detection of the clandestine user was felt to be **beyond the scope of purely automated
> techniques.**" (§18.2, p. 571)

**The review consequences are concrete.** A masquerader using stolen credentials looks like the
legitimate user to any authorization check, so the only signal is behavioural — which requires that
behaviour be recorded in the first place. A misfeasor is *inside* the policy: every action is
authorized, so no access-denied event fires, and the only trace is the record of successful,
legitimate-looking actions. **This is the argument for logging successful privileged actions, not just
failures** — a log that records only denials is blind to the entire insider case.

The clandestine user is the reason for the next point. An attacker with administrative control
suppresses the audit trail. If the trail is a file on the host being attacked, writable by the account
being used, it is evidence the attacker can edit. That is V-56, and the fix is architectural: append
only, shipped off-host promptly, with the destination's credentials not derivable from the source. See
§5 below.

---

## 3. What to record: the six-field audit record (§18.2, pp. 572–573)

Stallings contrasts two sources of audit data, and the trade is the same one every team makes about
logging:

> "**Native audit records:** Virtually all multiuser operating systems include accounting software that
> collects information on user activity. The advantage of using this information is that no additional
> collection software is needed. **The disadvantage is that the native audit records may not contain the
> needed information or may not contain it in a convenient form.**" (§18.2, p. 572)

> "**Detection-specific audit records:** A collection facility can be implemented that generates audit
> records containing **only that information required by the intrusion detection system.**" (§18.2,
> p. 572)

Framework access logs, cloud provider logs, and database logs are your native records: free, and
missing the domain facts that would let you answer a real question. Which record was read? On whose
behalf? Under which tenant? A detection-specific record is one you decide to emit, with the fields the
question needs.

The six fields (§18.2, p. 572), which are a remarkably good schema:

| Field | Stallings' definition | What it answers |
|---|---|---|
| **Subject** | "Initiators of actions. A subject is typically a terminal user but might also be a **process acting on behalf of users** or groups of users" | Who — including the real principal behind a service call |
| **Action** | "Operation performed by the subject on or with an object; for example, login, read, perform I/O, execute" | What was attempted |
| **Object** | "Receptors of actions. Examples include files, programs, messages, records, terminals, printers, and user- or program-created structures" | Which specific thing was touched |
| **Exception-Condition** | "Denotes which, if any, exception condition is raised on return" | Whether it succeeded, and how it failed |
| **Resource-Usage** | "A list of quantitative elements in which each element gives the amount used of some resource (e.g., number of lines printed or displayed, number of records read or written, processor time, I/O units used, session elapsed time)" | Scale — the difference between reading one row and reading the table |
| **Time-Stamp** | "**Unique** time-and-date stamp identifying when the action took place" | Ordering and correlation |

Two design rules follow, both stated:

> "**Because objects are the protectable entities in a system, the use of elementary actions enables an
> audit of all behavior affecting an object.**" (§18.2, p. 573)

> "**Single-object, single-action audit records simplify the model and the implementation.**" (§18.2,
> p. 573)

**Decompose to one subject, one action, one object per record.** A single log line covering "synced 400
records" cannot answer "who touched customer 12345, and when." One event per object is more volume and
vastly more answerable. This is the same normalisation instinct as data modelling, applied to evidence.

**Resource-Usage is the field everyone omits, and it is where exfiltration lives.** An attacker with
valid credentials reading every record generates no errors and no denials; the only anomaly is volume.
If the log records "list customers, 200 OK" without a row count, the bulk read and the single lookup
are indistinguishable. Record counts, byte sizes, and durations on any operation that can return
unbounded data.

The subject definition deserves emphasis too: "a process acting on behalf of users." In a service
architecture the acting identity is usually a service account, and logging only that identity destroys
the audit trail's value — every action is attributed to `api-service`. **Propagate and record the
originating principal, distinctly from the service identity that executed the call.**

Absence of these events for security-relevant actions is V-55. The list of what qualifies: authentication
outcomes (both), authorization denials, privilege and role changes, credential and key lifecycle
operations, changes to security configuration, access to sensitive records, bulk reads and exports,
administrative actions, and — from `08-transport-and-channel-security.md` §3 — rejected messages and
failed integrity checks.

---

## 4. Anomaly versus rule, and why both (§18.2, pp. 571–577)

> "**Statistical anomaly detection:** Involves the collection of data relating to the behavior of
> legitimate users over a period of time. Then statistical tests are applied to observed behavior to
> determine with a high level of confidence whether that behavior is not legitimate user behavior."
> (§18.2, p. 571)

> "**Rule-based detection:** Involves an attempt to define a set of rules that can be used to decide
> that a given behavior is that of an intruder." (§18.2, p. 571)

The distinction in one line:

> "In a nutshell, **statistical approaches attempt to define normal, or expected, behavior, whereas
> rule-based approaches attempt to define proper behavior.**" (§18.2, p. 572)

Matched to the adversaries of §2:

> "**Statistical anomaly detection is effective against masqueraders**, who are unlikely to mimic the
> behavior patterns of the accounts they appropriate." (§18.2, p. 572)

> "On the other hand, such techniques **may be unable to deal with misfeasors.** For such attacks,
> **rule-based approaches** may be able to recognize events and sequences that, in context, reveal
> penetration." (§18.2, p. 572)

> "In practice, a system may exhibit **a combination of both approaches** to be effective against a
> broad range of attacks." (§18.2, p. 572)

The costs are stated honestly on both sides. Statistical: "The main advantage of the use of statistical
profiles is that **a prior knowledge of security flaws is not required.** The detector program learns
what is 'normal' behavior and then looks for deviations" (§18.2, p. 576). Rule-based: "**the strength
of the approach depends on the skill of those involved in setting up the rules**" (§18.2, p. 577), and
"a weakness of this plan is **its lack of flexibility**" (§18.2, p. 577).

**What this means for a code reviewer** is narrower than it sounds. You are not building a detector.
But you decide whether the events you emit can support either approach: an anomaly detector needs a
*continuous, comparable* signal (counts, rates, volumes per principal over time), and a rule needs
*specific, categorical* facts (this role changed, this export ran, this credential was used from a new
place). Emitting only unstructured message strings supports neither. Emitting structured events with
stable field names and numeric measures supports both.

---

## 5. Protecting the audit trail (§18.2, pp. 578–579)

Once audit data moves across a network — and in any distributed system it must — it becomes an asset in
transit:

> "Although it is possible to mount a defense by using stand-alone intrusion detection systems on each
> host, **a more effective defense can be achieved by coordination and cooperation among intrusion
> detection systems across the network.**" (§18.2, p. 578)

> "One or more nodes in the network will serve as collection and analysis points for the data from the
> systems on the network. Thus, either raw audit data or summary data must be transmitted across the
> network." (§18.2, p. 579)

> "**Therefore, there is a requirement to assure the integrity and confidentiality of these data.**"
> (§18.2, p. 579)

> "**Integrity is required to prevent an intruder from masking his or her activities by altering the
> transmitted audit information.**" (§18.2, p. 579)

> "**Confidentiality is required because the transmitted audit information could be valuable.**" (§18.2,
> p. 579)

Both halves are review findings. Integrity: an attacker who can modify or delete log entries erases the
only record of what happened, and the clandestine user of §2 will try exactly this. Confidentiality:
the audit stream is a high-value target in its own right, because it describes the system's structure,
its principals, and its activity — and because developers put things in it that do not belong there.

That second point is V-58 and it is the most common logging finding by a wide margin. Credentials,
tokens, session identifiers, API keys, full request bodies, personal data, card numbers, and
`Authorization` headers all end up in logs, from which they reach a third-party aggregator with a
different access model, a longer retention period, and a wider audience than the system that generated
them. **A log is a data store.** It inherits every classification obligation of the data in it, and
§20.2's information-flow rules apply to it (see `12-perimeter-and-trusted-systems.md` §3): writing
sensitive data to a lower-classification sink is a policy violation regardless of the access control on
the sink.

Stallings also names the centralisation trade, which is worth stating because the answer is not obvious:

> "With a centralized architecture, there is a single central point of collection and analysis of all
> audit data. **This eases the task of correlating incoming reports but creates a potential bottleneck
> and single point of failure.**" (§18.2, p. 579)

And the mundane problem that eats most of the effort: "A distributed intrusion detection system may
need to **deal with different audit record formats**" (§18.2, p. 578). Twenty years later this is still
where log pipelines go to die. Agree on a schema — the six fields of §3 are a good starting point — and
enforce it at emission rather than repairing it at ingestion.

**The practical requirements.** Append-only from the perspective of the code that writes it. Shipped
off the host promptly, so that compromising the host does not retroactively erase evidence. Write
credentials that do not permit deletion or modification. Retention long enough to cover realistic
discovery delay, which is months and not days. And the trail itself needs access control and an audit
of who read it.

---

## 6. The base-rate fallacy: why your alerts are ignored (§18.2, p. 577; App. 18A, p. 596)

The overlap problem first:

> "Intrusion detection is based on the assumption that **the behavior of the intruder differs from that
> of a legitimate user in ways that can be quantified.**" (§18.2, p. 570)

> "Of course, we cannot expect that there will be a crisp, exact distinction between an attack by an
> intruder and the normal use of resources by an authorized user. Rather, **we must expect that there
> will be some overlap.**" (§18.2, p. 570)

> "Thus, **a loose interpretation of intruder behavior, which will catch more intruders, will also lead
> to a number of 'false positives'**, or authorized users identified as intruders. On the other hand,
> **an attempt to limit false positives by a tight interpretation of intruder behavior will lead to an
> increase in false negatives**, or intruders not identified as intruders." (§18.2, p. 570)

> "Thus, there is an element of **compromise and art** in the practice of intrusion detection." (§18.2,
> p. 570)

Then the argument that makes this worse than a simple trade-off. Appendix 18A uses a medical analogy —
a test with "the accuracy of the test is 87%" and where "the incidence of the disease in the population
is 1%" (App. 18A, p. 596):

> "**Most subjects gave the answer 13%.** The vast majority, including many physicians, gave a number
> below 50%." (App. 18A, p. 596)

> "**Thus, in the vast majority of cases, when a disease condition is detected, it is a false alarm.**"
> (App. 18A, p. 596)

> "The reason most people get it wrong is that they do not take into account the basic rate of incidence
> (**the base rate**) when intuitively solving the problem. This error is known as the **base-rate
> fallacy.**" (App. 18A, p. 596)

Then the numbers that should be pinned above every alerting dashboard:

> "Suppose we have Pr[positive/disease] = 0.999 and Pr[negative/well] = 0.999… we get
> Pr[well/positive] = 0.09." (App. 18A, p. 596)

> "Thus, if we can accurately detect disease and accurately detect lack of disease at a level of
> **99.9%**, then the rate of false alarms will be **9%**." (App. 18A, p. 596)

> "Moreover, again assume 99.9% accuracy, but now suppose that the incidence of the disease in the
> population is only 1/10000 = 0.0001. We then end up with **a rate of false alarms of 91%.**"
> (App. 18A, p. 596)

Applied to detection:

> "**In general, if the actual numbers of intrusions is low compared to the number of legitimate uses of
> a system, then the false alarm rate will be high unless the test is extremely discriminating.**"
> (§18.2, p. 577)

> "A study of existing intrusion detection systems, reported in [AXEL00], indicated that **current
> systems have not overcome the problem of the base-rate fallacy.**" (§18.2, p. 577)

**A 99.9%-accurate detector produces 91% false alarms when the thing it detects is rare — and
intrusions are rare.** This is not a tuning problem, it is arithmetic. Which yields the practical
conclusions:

- **An alert nobody can act on trains people to ignore alerts.** A noisy detector is worse than none,
  because it consumes the attention that a real signal would need. This is V-57.
- **Raise the base rate instead of the accuracy.** Alert on narrow, high-prior conditions — a service
  account used from a new network, a production credential used outside deploy, an export of more than
  N records by a non-admin — rather than broad "unusual activity." Narrowing the population is far more
  effective than improving the classifier.
- **Separate detection from alerting.** Record everything (§3); alert on very little. Recorded events
  are for investigation after you have a reason, and their value does not depend on precision.
- **Every alert needs a stated response.** If the answer to "what do we do when this fires" is "look at
  it," it is a dashboard, not an alert.

Honeypots are the clean illustration of the base-rate fix, and Stallings says so almost in those terms:

> "**Honeypots** are decoy systems that are designed to lure a potential attacker away from critical
> systems." (§18.2, p. 581)

> "These systems are filled with fabricated information designed to appear valuable but that a
> legitimate user of the system wouldn't access. **Thus, any access to the honeypot is suspect.**"
> (§18.2, p. 581)

**Any access is suspect** — a signal whose base rate of legitimate use is zero, and therefore whose
false alarm rate is zero. The application analogue is a canary: a decoy record, a fake admin route, an
unused API key that nothing legitimate should ever touch. It is one of the few detections that is cheap
to build, cheap to run, and essentially never wrong.

---

## 7. Review checklist

**Event coverage**

- [ ] Are authentication outcomes recorded — successes as well as failures?
- [ ] Are authorization denials recorded?
- [ ] Are **successful** privileged and administrative actions recorded, not just failures? (The
      misfeasor case produces no denials.)
- [ ] Are privilege, role, and permission changes recorded?
- [ ] Are credential and key lifecycle events recorded — issue, rotate, revoke, use of a long-lived key?
- [ ] Are changes to security configuration recorded?
- [ ] Are access to sensitive records, bulk reads, and exports recorded?
- [ ] Are rejected messages, failed integrity checks, and replay rejections recorded rather than
      silently dropped? (V-55)

**Event content**

- [ ] Does each event carry subject, action, object, outcome, and timestamp?
- [ ] Is the *originating* principal recorded distinctly from the service identity that executed the
      call?
- [ ] Is the object identified specifically enough to answer "who touched this record?"
- [ ] Is a quantitative measure recorded for anything that can return unbounded data — row counts, byte
      sizes, durations?
- [ ] Is the granularity one subject / one action / one object, rather than a summary line?
- [ ] Are events structured with stable field names, so they can support both rule-based and statistical
      analysis?
- [ ] Are timestamps unique and ordered well enough to correlate across services?

**Trail protection**

- [ ] Is the audit trail append-only from the writer's perspective, and can the audited subject modify
      or delete it? (V-56)
- [ ] Is it shipped off-host promptly, with write credentials that cannot delete or rewrite history?
- [ ] Is it protected in transit and at rest, given that it is a high-value target?
- [ ] Is retention long enough to cover realistic discovery delay?
- [ ] Is read access to the trail itself controlled and audited?

**Sensitive data in logs**

- [ ] Are credentials, tokens, session identifiers, keys, and `Authorization` headers excluded? (V-58)
- [ ] Are full request/response bodies excluded, or filtered by an allowlist rather than a denylist?
- [ ] Do error paths and stack traces avoid emitting secrets and personal data?
- [ ] Does the log destination's classification, retention, and access model match the data being sent
      to it — including third-party aggregators?

**Alerting**

- [ ] For each alert: what is the expected base rate of the condition, and what fraction of firings will
      be false? (V-57)
- [ ] Is the condition narrow and high-prior, rather than a broad "anomaly"?
- [ ] Is there a stated response for each alert, distinct from "have a look"?
- [ ] Are recording and alerting separated, so that broad recording does not create noise?
- [ ] Is there at least one canary — a decoy record, route, or credential whose legitimate access rate is
      zero?
