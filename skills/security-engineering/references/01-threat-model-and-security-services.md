# Threat Model and Security Services

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 1 (p. 5).

This is the vocabulary chapter. Almost every muddled security discussion is a failure to keep
attacks, mechanisms, and services distinct, or a failure to say which service is being claimed.
Read this before writing a threat model.

---

## 1. Attack, mechanism, service (§1.2, p. 12)

X.800, *Security Architecture for OSI*, defines three things:

- **Security attack:** "Any action that compromises the security of information owned by an
  organization."
- **Security mechanism:** "A process (or a device incorporating such a process) that is
  designed to detect, prevent, or recover from a security attack."
- **Security service:** "A processing or communication service that enhances the security of
  the data processing systems and the information transfers of an organization. The services
  are intended to counter security attacks, and they make use of one or more security
  mechanisms to provide the service."

RFC 2828 adds the pair that reviews need (§1.2, p. 12, Table 1.1):

- **Threat:** "A potential for violation of security, which exists when there is a
  circumstance, capability, action, or event that could breach security and cause harm. That
  is, a threat is a possible danger that might exploit a vulnerability."
- **Attack:** "An assault on system security that derives from an intelligent threat; that is,
  an intelligent act that is a deliberate attempt (especially in the sense of a method or
  technique) to evade security services and violate the security policy of a system."

**Why this matters in review.** The chain is: *threat* exploits *vulnerability* to mount an
*attack* against a *service*, which is provided by a *mechanism*. A finding should name the
vulnerability and the service it breaks. "Uses a weak hash" names neither.

X.800 also distinguishes **reversible** encipherment (encryption, which can be undone) from
**irreversible** encipherment — "hash algorithms and message authentication codes, which are
used in digital signature and message authentication applications" (§1.5, p. 19). Both are
"encipherment" in the standard's vocabulary, which is exactly why "it's encrypted" is
ambiguous enough to be useless.

---

## 2. Attack taxonomy (§1.3, p. 13)

> "A passive attack attempts to learn or make use of information from the system but does not
> affect system resources. An active attack attempts to alter system resources or affect their
> operation."

### 2.1 Passive attacks

| Attack | Description |
|---|---|
| **Release of message contents** | The straightforward one — the opponent reads a conversation, message, or file (p. 13) |
| **Traffic analysis** | "Suppose that we had a way of masking the contents … an opponent might still be able to observe the pattern of these messages. The opponent could determine the location and identity of communicating hosts and could observe the frequency and length of messages being exchanged. This information might be useful in guessing the nature of the communication" (p. 13) |

The strategic consequence, and the reason this section is short but load-bearing:

> "Passive attacks are very difficult to detect because they do not involve any alteration of
> the data. Typically, the message traffic is sent and received in an apparently normal fashion
> and neither the sender nor receiver is aware that a third party has read the messages or
> observed the traffic pattern. However, it is feasible to prevent the success of these
> attacks, usually by means of encryption. Thus, the emphasis in dealing with passive attacks
> is on **prevention rather than detection**." (p. 13)

You will never get an alert for a confidentiality breach in transit. So "we'd notice" is not an
argument for leaving a channel unencrypted, and "we'll add TLS later, we monitor closely" is not
a mitigation.

**Traffic analysis is a distinct service to buy or decline.** If the existence, size, timing, or
endpoints of a message are themselves sensitive — a whistleblower endpoint, a health condition
implied by which API is called, a request size that reveals which document was fetched — then
encryption does not cover it. The mechanisms are padding and routing control (§1.5, p. 20), and
in IPsec, tunnel mode (§16.2, p. 490).

### 2.2 Active attacks

"Active attacks involve some modification of the data stream or the creation of a false stream
and can be subdivided into four categories" (p. 13):

| Attack | Definition | Application analogue |
|---|---|---|
| **Masquerade** | "One entity pretends to be a different entity … authentication sequences can be captured and replayed after a valid authentication sequence has taken place, thus enabling an authorized entity with few privileges to obtain extra privileges by impersonating an entity that has those privileges" (p. 13) | Session hijacking, token theft, privilege escalation via a forged identity claim |
| **Replay** | "The passive capture of a data unit and its subsequent retransmission to produce an unauthorized effect" (p. 13) | Replayed webhook, resubmitted payment, reused one-time token |
| **Modification of messages** | "Some portion of a legitimate message is altered, or … messages are delayed or reordered, to produce an unauthorized effect." Stallings' example: "Allow John Smith to read confidential file accounts" becomes "Allow Fred Brown to read confidential file accounts" (p. 14) | Parameter tampering, unauthenticated IV bit-flip, reordered event stream |
| **Denial of service** | "Prevents or inhibits the normal use or management of communications facilities." Either targeted — "an entity may suppress all messages directed to a particular destination (e.g., the security audit service)" — or broad, "by disabling the network or by overloading it with messages so as to degrade performance" (p. 14) | Resource exhaustion, amplification, lock contention, log flooding |

Note the parenthetical in the DoS definition: suppressing traffic to the **audit service** is
listed as an attack in its own right. An attacker's first move against a monitored system is to
break the monitoring. That is the justification for V-56 (audit trail modifiable by its
subject).

And the mirror-image conclusion to §2.1:

> "Whereas passive attacks are difficult to detect, measures are available to prevent their
> success. On the other hand, it is quite difficult to prevent active attacks absolutely,
> because of the wide variety of potential physical, software, and network vulnerabilities.
> Instead, the goal is to **detect** active attacks and to recover from any disruption or
> delays caused by them. If the detection has a deterrent effect, it may also contribute to
> prevention." (p. 15)

**The design rule that falls out of §2:**

| Against | Primary posture | Because |
|---|---|---|
| Passive (disclosure, traffic analysis) | **Prevent** by construction | You will never detect it |
| Active (masquerade, replay, modification, DoS) | **Prevent where you can, and always detect** | You cannot prevent all of it |

This is why every security design owes an answer to "if this failed, would we know?" — and why
audit logging is not an optional extra in a security review.

---

## 3. The six security services (§1.4, pp. 16–19)

X.800 defines five categories and fourteen specific services (Table 1.2, p. 17). The categories,
with the distinctions that matter in practice:

### Authentication
"The assurance that the communicating entity is the one that it claims to be." Two specific
services:

- **Peer entity authentication:** "corroboration of the identity of a peer entity in an
  association … It attempts to provide confidence that an entity is not performing either a
  masquerade or an unauthorized replay of a previous connection" (p. 18).
- **Data origin authentication:** "corroboration of the source of a data unit. **It does not
  provide protection against the duplication or modification of data units.** This type of
  service supports applications like electronic mail where there are no prior interactions
  between the communicating entities" (p. 18).

**That bolded sentence is one of the most useful in the book for review work.** A valid signature
or MAC tells you who produced a message. It does not tell you that this is the first time you
have seen it. Freshness is separate work — a nonce, a timestamp, or a recorded message ID (see
`06-signatures-and-authentication-protocols.md`). Every "but it's signed" defense of a replayable
endpoint dies here.

For an ongoing interaction the service has two obligations: authenticate both entities at
connection setup, *and* "assure that the connection is not interfered with in such a way that a
third party can masquerade as one of the two legitimate parties for the purposes of unauthorized
transmission or reception" (p. 18). Authenticating only at login and then trusting the channel
satisfies the first and not the second.

### Access control
"The prevention of unauthorized use of a resource (i.e., this service controls who can have
access to a resource, under what conditions access can occur, and what those accessing the
resource are allowed to do)" (p. 17).

And the relationship to authentication: "each entity trying to gain access must first be
identified, or authenticated, so that access rights can be tailored to the individual" (p. 18).
Authentication is a *precondition* for access control, not a substitute. A logged-in user is not
an authorized user.

### Data confidentiality
"The protection of data from unauthorized disclosure" (p. 17), at four granularities: connection,
connectionless, selective-field, and **traffic flow**. Stallings notes the narrower forms "are
less useful than the broad approach and may even be more complex and expensive to implement"
(p. 18) — an argument for encrypting the whole channel rather than hand-picking fields.

### Data integrity
"The assurance that data received are exactly as sent by an authorized entity (i.e., contain no
modification, insertion, deletion, or **replay**)" (p. 17). Note that replay is inside the
definition of integrity.

Two axes to be explicit about:

- **Connection-oriented** integrity covers a stream: it "assures that messages are received as
  sent, with no duplication, insertion, modification, reordering, or replays," and therefore
  "addresses both message stream modification and denial of service." **Connectionless**
  integrity, dealing with individual messages, "generally provides protection against message
  modification only" (p. 19). If your protocol is a series of independent signed messages, you
  have the weaker service and must add ordering and replay defenses yourself.
- **With or without recovery.** "Because the integrity service relates to active attacks, we are
  concerned with detection rather than prevention. If a violation of integrity is detected, then
  the service may simply report this violation, and some other portion of software or human
  intervention is required to recover" (p. 19). Detection with an unhandled exception is not a
  control — decide what happens on failure.

### Nonrepudiation
"Provides protection against denial by one of the entities involved in a communication of having
participated in all or part of the communication" (p. 17), in both directions — origin and
destination (p. 18).

**Only a signature can do this.** A MAC cannot, "because both sender and receiver share the same
key" (§11.2, p. 327), so either party could have produced the tag. If a requirement says
"provably from the customer," HMAC is the wrong mechanism.

### Availability
"The property of a system or a system resource being accessible and usable upon demand by an
authorized system entity, according to performance specifications for the system" (p. 19).

X.800 treats it as a property, but Stallings argues for calling it out as a service, noting it
"depends on proper management and control of system resources and thus depends on access control
service and other security services" (p. 19). In practice this is the link between security review
and capacity review: an unauthenticated expensive endpoint is an availability vulnerability
(V-62), not a performance nit.

---

## 4. Mechanisms (§1.5, pp. 19–21)

X.800's specific mechanisms — the building blocks a design should be able to name:
encipherment, digital signature, access control, data integrity, authentication exchange,
**traffic padding**, **routing control**, notarization (Table 1.3, p. 20).

Its pervasive mechanisms are the ones software teams most often skip because they belong to no
single feature: trusted functionality, **security label**, **event detection**, **security audit
trail**, security recovery (Table 1.3, p. 20).

Table 1.4 (p. 21) maps services to mechanisms. Two rows worth memorizing:

- **Data integrity** is provided by encipherment *and* digital signature *and* a data-integrity
  mechanism — not by encipherment alone.
- **Nonrepudiation** is provided by digital signature, data integrity, and notarization — and
  conspicuously *not* by encipherment.

---

## 5. The model, and the four design tasks (§1.6, pp. 22–23)

Every security technique has two components (p. 22):

1. "A security-related transformation on the information to be sent" — encryption, or "the
   addition of a code based on the contents of the message."
2. "Some secret information shared by the two principals and, it is hoped, unknown to the
   opponent."

The parenthetical *it is hoped* is the whole reason secret management dominates real incidents.
Component 1 is a library call; component 2 is your problem.

From the model, "there are four basic tasks in designing a particular security service" (p. 23):

1. **Design an algorithm** for the security transformation, such that "an opponent cannot defeat
   its purpose."
2. **Generate the secret information** to be used with the algorithm.
3. **Develop methods for the distribution and sharing** of the secret information.
4. **Specify a protocol** used by the two principals that employs the algorithm and the secret
   to achieve the service.

**Use this as a review checklist.** Task 1 is almost always fine — it is AES from a real library.
Tasks 2, 3, and 4 are where findings live: a key from `Math.random()` (task 2, V-09), a public
key fetched over HTTP (task 3, V-33), a protocol with no freshness (task 4, V-25). When a
security design document only discusses task 1, it is a quarter finished.

### 5.1 The second model: unwanted access (§1.6, p. 23)

A separate model covers threats that are not about a message in transit (Figure 1.6, p. 23).
Programs present two kinds of threat:

- **Information access threats:** "intercept or modify data on behalf of users who should not
  have access to that data."
- **Service threats:** "exploit service flaws in computers to inhibit use by legitimate users."

And the defenses come in two layers: a **gatekeeper function** — "password-based login procedures
that are designed to deny access to all but authorized users and screening logic that is designed
to detect and reject worms, viruses, and other similar attacks" — and, "once either an unwanted
user or unwanted software gains access," a second line of "internal controls that monitor activity
and analyze stored information in an attempt to detect the presence of unwanted intruders" (p. 24).

The assumption that the gatekeeper *will* be bypassed is baked into the model. That is the
architectural argument for defense in depth, for authorizing internal calls (V-46), and for audit
logging (V-55).

---

## 6. Why this is hard (§1, p. 9)

Stallings lists five reasons, and three of them are direct instructions for how to review:

> "In developing a particular security mechanism or algorithm, one must always consider potential
> attacks on those security features. In many cases, successful attacks are designed by looking at
> the problem in a completely different way, therefore exploiting an unexpected weakness in the
> mechanism." (p. 9)

This is the mandate for the adversarial stance in `/review-security`: reading the code as the
author intended it to be read will not find the bug.

> "Because of point 2, the procedures used to provide particular services are often
> counterintuitive: It is not obvious from the statement of a particular requirement that such
> elaborate measures are needed." (p. 9)

Which is why "this seems like overkill" is not a valid objection to a standard construction, and
why homemade simplifications (V-08) are so reliably broken.

> "Security mechanisms usually involve more than a particular algorithm or protocol. They usually
> also require that participants be in possession of some secret information … which raises
> questions about the creation, distribution, and protection of that secret information. There is
> also a reliance on communications protocols whose behavior may complicate the task … For
> example, if the proper functioning of the security mechanism requires setting time limits on
> the transit time of a message from sender to receiver, then any protocol or network that
> introduces variable, unpredictable delays may render such time limits meaningless." (p. 9)

The last sentence is the seed of the entire timestamp-versus-nonce argument in Chapter 13 (see
`06-signatures-and-authentication-protocols.md`), and of the suppress-replay attack.

Finally, a placement question that is easy to get wrong: "Having designed various security
mechanisms, it is necessary to decide where to use them. This is true both in terms of physical
placement … and in a logical sense [e.g., at what layer or layers of an architecture such as
TCP/IP should mechanisms be placed]" (p. 9). Chapter 17 answers it — see
`08-transport-and-channel-security.md`.

---

## 7. Review checklist from this chapter

- [ ] Which of the six services does this change claim to provide? Name them.
- [ ] For each claimed service, what is the mechanism, and where is it enforced?
- [ ] Is any service being claimed from a mechanism that cannot provide it? (Encryption ⇏
      integrity; MAC ⇏ nonrepudiation; signature ⇏ freshness; authentication ⇏ authorization.)
- [ ] Passive threats: is the data prevented from disclosure, given that detection is impossible?
- [ ] Is message existence, size, or timing itself sensitive? If so, is traffic-flow
      confidentiality addressed or explicitly declined?
- [ ] Active threats: for masquerade, replay, modification, and DoS — what prevents each, and
      what detects each?
- [ ] Of the four design tasks, are 2 (generation), 3 (distribution), and 4 (protocol) actually
      specified — or only 1 (algorithm)?
- [ ] Can the audit trail itself be suppressed by the attacker?
- [ ] If prevention fails, what is the recovery path, and who is notified?
