# Perimeter Controls, Trusted Systems, and Assurance

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 20 (Firewalls, p. 621):
§20.1 Firewall Design Principles, §20.2 Trusted Systems, §20.3 Common Criteria.

Chapter 20 is the most misread chapter in the book. Read as "how to configure a firewall" it is dated.
Read as **what a boundary can and cannot do, and what it takes to be able to claim a system is
secure**, it contains three of the most useful ideas available: the firewall's explicitly stated
*limitations*, the reference monitor's three properties, and the Trojan horse defence, which is the
book's one worked example of a policy overriding a permission.

The access matrix vocabulary (subject, object, access right) and the bastion host's least-privilege
properties are covered in `09-authentication-and-access-control.md` §6 and §8.

---

## 1. What a boundary is for, and the three conditions (§20.1, pp. 623–624)

> "The aim of this perimeter is to protect the premises network from Internet-based attacks and to
> provide **a single choke point where security and audit can be imposed.**" (§20.1, p. 623)

The three design goals, all of which have to hold or the boundary is decorative:

> "1. **All traffic from inside to outside, and vice versa, must pass through the firewall.**
> 2. **Only authorized traffic, as defined by the local security policy, will be allowed to pass.**
> 3. **The firewall itself is immune to penetration.**" (§20.1, p. 623)

> "A firewall defines a single choke point that keeps unauthorized users out of the protected network,
> prohibits potentially vulnerable services from entering or leaving the network, and provides protection
> from various kinds of IP spoofing and routing attacks." (§20.1, p. 624)

> "**The use of a single choke point simplifies security management because security capabilities are
> consolidated** on a single system or set of systems." (§20.1, p. 624)

Goal 1 is complete mediation (§4) at the network layer: a path that avoids the boundary makes the
boundary irrelevant, not weaker. Goal 3 says the enforcement point is itself an asset, with a higher
assurance requirement than what it protects — which is why an API gateway, service mesh control plane,
policy engine, or identity provider deserves stricter treatment than the services behind it. A
consolidated control is a consolidated target.

The four things such a control can decide (§20.1, pp. 623–624) are a better vocabulary for API-level
policy than most API policy documents use:

> "**Service control:** Determines the types of Internet services that can be accessed, inbound or
> outbound." (p. 623)

> "**Direction control:** Determines the direction in which particular service requests may be initiated
> and allowed to flow through the firewall." (p. 624)

> "**User control:** Controls access to a service according to which user is attempting to access it."
> (p. 624)

> "**Behavior control:** Controls how particular services are used." (p. 624)

**Direction control is the one that gets skipped.** Most reviews consider what may come in and never
what may go out. Egress restriction is what limits data exfiltration, blocks callbacks to
attacker-controlled hosts, and contains SSRF — and it is the control that makes the runtime containment
argument of `11-malicious-software-and-availability.md` §5 real. Behaviour control is the second most
neglected: allowing a service but constraining *how* it may be used (which methods, which paths, what
volume, what shape of payload) is where most real policy lives.

---

## 2. The limitations, quoted in full, because they are the point (§20.1, p. 624)

> "**The firewall cannot protect against attacks that bypass the firewall.** Internal systems may have
> dial-out capability to connect to an ISP. An internal LAN may support a modem pool that provides
> dial-in capability for traveling employees and telecommuters." (§20.1, p. 624)

> "**The firewall does not protect against internal threats, such as a disgruntled employee or an
> employee who unwittingly cooperates with an external attacker.**" (§20.1, p. 624)

> "**The firewall cannot protect against the transfer of virus-infected programs or files.** Because of
> the variety of operating systems and applications supported inside the perimeter, it would be
> impractical and perhaps impossible for the firewall to scan all incoming files, e-mail, and messages
> for viruses." (§20.1, p. 624)

Three limitations, written in 2005, that describe every perimeter breach since. The modem pool is now a
laptop on a home network, a SaaS integration with an API key, a developer's workstation, a CI runner, a
managed database with a public endpoint, or a personal device on the corporate VPN. The category is
unchanged: **a path exists that does not traverse the control.**

**This is the entire case against trusting network position** (V-46), and Stallings makes it without any
of the marketing vocabulary that came later. A boundary constrains what crosses it. It says nothing
about what is already inside, nothing about a legitimate insider misusing access, and nothing about
content it cannot inspect. Therefore:

- An internal service that skips authentication because "it's not exposed" is relying on limitation 1
  never occurring.
- An internal service that skips authorization because "only our own services call it" is relying on
  limitation 2 never occurring.
- A service that accepts a payload from a trusted peer without validating it is relying on limitation 3
  never occurring.

The correct reading is not "perimeters are useless." It is that a perimeter is **one layer whose
failure must not be catastrophic** — which is exactly the argument §3 makes for defence in depth.

---

## 3. Default deny, and the attacks on a filter that only looks at headers (§20.1, pp. 626–634)

The mechanism, and then the single most transferable sentence in the chapter:

> "The packet filter is typically set up as a list of rules based on matches to fields in the IP or TCP
> header. If there is a match to one of the rules, that rule is invoked to determine whether to forward
> or discard the packet. **If there is no match to any rule, then a default action is taken.**" (§20.1,
> p. 626)

> "**Default = discard:** That which is not expressly permitted is prohibited." (§20.1, p. 626)

> "**Default = forward:** That which is not expressly prohibited is permitted." (§20.1, p. 626)

> "**The default discard policy is more conservative.**" (§20.1, p. 626)

**Apply this to everything with rules, not just networks.** Route authorization: does an endpoint with
no declared policy default to authenticated-and-authorized, or to open? Field serialisation: does a new
database column appear in the API response automatically, or only when explicitly exposed? Feature
flags, CORS origins, IAM policies, message-topic subscriptions, admin capability lists — each has a
default, and the default is the security policy for everything anyone forgets. Most frameworks default
to forward, which is why the safe pattern is a global deny with explicit opt-in per route.

The three attacks on a header-only filter (§20.1, pp. 628–629) each carry a general lesson:

> "**IP address spoofing:** The intruder transmits packets from the outside with a source IP address
> field containing an address of an internal host. The attacker hopes that the use of a spoofed address
> will allow penetration of systems that **employ simple source address security, in which packets from
> specific trusted internal hosts are accepted.** The countermeasure is to discard packets with an inside
> source address if the packet arrives on an external interface."

> "**Source routing attacks:** The source station specifies the route that a packet should take as it
> crosses the Internet, **in the hopes that this will bypass security measures that do not analyze the
> source routing information.** The countermeasure is to discard all packets that use this option."

> "**Tiny fragment attacks:** The intruder uses the IP fragmentation option to create extremely small
> fragments and force the TCP header information into a separate packet fragment. **This attack is
> designed to circumvent filtering rules that depend on TCP header information.**"

Generalised: **an identifier the attacker supplies is not an identity** (spoofing — the modern form is
a trusted `X-Forwarded-For`, `X-Real-IP`, or `X-User-Id` header); **an attacker-chosen routing or
processing option can steer a request around a check** (source routing — the modern form is a parameter
that selects which handler, parser, or validator runs); and **a decision made on a fragment of a
message is a decision made on incomplete information** (tiny fragments — the modern form is any
inspection that happens before reassembly, normalisation, or decoding, which is why filters that match
on raw request text are routinely bypassed by encoding). The fragmentation countermeasure states the
right principle: "**If the first fragment is rejected, the filter can remember the packet and discard
all subsequent fragments**" (§20.1, p. 629) — decide on the whole thing, in canonical form, once.

The stated weakness of the header-only approach is why proxies exist:

> "**Because packet filter firewalls do not examine upper-layer data, they cannot prevent attacks that
> employ application-specific vulnerabilities or functions.**" (§20.1, p. 628)

> "**Application-level gateways tend to be more secure than packet filters.** Rather than trying to deal
> with the numerous possible combinations that are to be allowed and forbidden at the TCP and IP level,
> the application-level gateway **need only scrutinize a few allowable applications.**" (§20.1, p. 630)

> "If the gateway does not implement the proxy code for a specific application, **the service is not
> supported and cannot be forwarded** across the firewall." (§20.1, p. 630)

> "In addition, **it is easy to log and audit all incoming traffic at the application level.**" (§20.1,
> p. 630)

Two reusable ideas: **enumerate a small set of allowed operations rather than trying to enumerate the
infinite set of forbidden ones**, and **a control that understands the semantics of what it inspects
can enforce properties a syntactic filter cannot.** This is the argument for schema validation at the
edge over regex-based request filtering, and it is why a web application firewall is a mitigation rather
than a fix.

Defence in depth is argued explicitly, in terms of attacker work rather than of prevention:

> "This configuration has greater security than simply a packet-filtering router or an application-level
> gateway alone, for two reasons. First, this configuration implements **both packet-level and
> application-level filtering**, allowing for considerable flexibility in defining security policy.
> Second, **an intruder must generally penetrate two separate systems before the security of the internal
> network is compromised.**" (§20.1, p. 632)

> "The screened subnet firewall configuration … is the most secure of those we have considered. **There
> are now three levels of defense to thwart intruders.**" (§20.1, p. 634)

> "The outside router advertises only the existence of the screened subnet to the Internet; **therefore,
> the internal network is invisible to the Internet.** Similarly, the inside router advertises only the
> existence of the screened subnet to the internal network; **therefore, the systems on the inside
> network cannot construct direct routes to the Internet.**" (§20.1, p. 634)

The last sentence is bidirectional containment — the inside cannot reach out directly either. That is
egress control as an architectural property rather than a rule list, and it is the single most effective
structural control against data exfiltration and command-and-control.

---

## 4. The reference monitor: three properties, and why they generalise (§20.2, pp. 637–638)

This is the most valuable half-page in the chapter.

> "**Complete mediation:** The security rules are enforced on **every access**, not just, for example,
> when a file is opened." (§20.2, p. 637)

> "**Isolation:** The reference monitor and database are **protected from unauthorized modification.**"
> (§20.2, p. 637)

> "**Verifiability:** The reference monitor's correctness **must be provable.** That is, it must be
> possible to demonstrate mathematically that the reference monitor enforces the security rules and
> provides complete mediation and isolation." (§20.2, p. 637)

> "A system that can provide such verification is referred to as a **trusted system.**" (§20.2, p. 637)

Its inputs and outputs:

> "The reference monitor has access to a file, known as the **security kernel database**, that lists the
> access privileges (security clearance) of each subject and the protection attributes (classification
> level) of each object." (§20.2, p. 637)

> "A final element … is an **audit file. Important security events, such as detected security violations
> and authorized changes to the security kernel database, are stored in the audit file.**" (§20.2, p. 638)

**These three properties are a complete review framework for any authorization mechanism**, at any
scale — an operating system kernel, a policy service, a middleware, a `can_user_do_this()` function:

- **Complete mediation.** Is the check on every access, or once per session/batch/flow? Is it on the
  enforcement path, or only on the paths you happened to think about — the UI, the gateway, the list
  query, but not the direct mutation by id? Every bypass route is a violation of property 1. This is
  V-44.
- **Isolation.** Can the decision or the data it relies on be modified by the subject being evaluated?
  A role claim in a client-supplied token that is not verified, a tenant id read from the request body,
  a permission cached in a client-writable session, an admin flag settable through a mass-assignment
  bug — all are isolation failures. The policy data is as sensitive as the enforcement code.
- **Verifiability.** Can you *demonstrate* the rules hold, or only assert it? In practice: is the policy
  expressed in one place where it can be read and reasoned about, or scattered across handlers as
  ad-hoc conditionals? Is it tested — specifically with negative tests, asserting that the wrong
  principal is refused? A policy nobody can enumerate cannot be verified, and a policy with only
  happy-path tests has not been checked at all.

Note that the audit file records both violations **and authorized changes to the policy database**. A
permission grant is a security event in its own right — see `10-intrusion-detection-and-audit.md` §3.

---

## 5. Policy beats permission: the Trojan horse defence (§20.2, pp. 636–638)

The multilevel rules first:

> "When multiple categories or levels of data are defined, the requirement is referred to as
> **multilevel security.**" (§20.2, p. 636)

> "**No read up:** A subject can only read an object of less or equal security level. This is referred to
> in the literature as the **Simple Security Property.**" (§20.2, p. 636)

> "**No write down:** A subject can only write into an object of greater or equal security level. This is
> referred to in the literature as the ***-Property** (pronounced star property)." (§20.2, p. 636)

> "**These two rules, if properly enforced, provide multilevel security.**" (§20.2, p. 636)

> "The general statement of the requirement for multilevel security is that **a subject at a high level
> may not convey information to a subject at a lower or noncomparable level unless that flow accurately
> reflects the will of an authorized user.**" (§20.2, p. 636)

*No write down* is the counter-intuitive one and the one that matters. It is not about protecting the
low-level object from corruption; it is about preventing information from **flowing downward** —
including via a process the high-level user was tricked into running.

The worked example (§20.2, p. 638). Bob has a sensitive file; Alice writes a Trojan horse and a
"back-pocket" file:

> "User Bob has created the file with read/write permission provided only to programs executing on his
> own behalf: that is, **only processes that are owned by Bob may access the file.**" (§20.2, p. 638)

> "Alice gives read/write permission to herself for this file and **gives Bob write-only permission**…
> Alice now induces Bob to invoke the Trojan horse program, perhaps by advertising it as a useful utility.
> **When the program detects that it is being executed by Bob, it reads the sensitive character string
> from Bob's file and copies it into Alice's back-pocket file.**" (§20.2, p. 638)

Every access control check passes. Bob's process may read Bob's file; Bob has been granted write access
to Alice's file. The ACL is satisfied at every step, and the data leaks. Now with levels:

> "In this example, there are two security levels, sensitive and public… **Processes owned by Bob and
> Bob's data file are assigned the security level sensitive. Alice's file and processes are restricted to
> public.**" (§20.2, p. 638)

> "When the program attempts to store the string in a public file (the back-pocket file), however, the
> [*-property] is violated and **the attempt is disallowed by the reference monitor.**" (§20.2, p. 638)

> "**Thus, the attempt to write into the back-pocket file is denied even though the access control list
> permits it: The security policy takes precedence over the access control list mechanism.**" (§20.2,
> p. 638)

**That final sentence is the most important line in the chapter.** Discretionary permissions describe
what a subject *may* do; an information-flow policy describes where data *may go*. They are different
questions, and a system that answers only the first is exploitable by any component that can be induced
to act on someone else's behalf — the confused deputy of
`11-malicious-software-and-availability.md` §3.

**The modern instances are everywhere, and they are all "the ACL permitted it":**

- A service authorised to read customer records and also to call an external analytics API. Both grants
  are individually reasonable; together they are a write-down path.
- Production data copied into a staging or analytics environment with a weaker access model. Every
  individual permission checks out.
- Sensitive fields written to logs, error trackers, or metrics labels — a write from a high-classification
  process to a low-classification sink (V-58).
- A user-supplied template, formula, or webhook URL evaluated by a process holding privileged data.
- An LLM or automation agent with read access to sensitive context and the ability to emit text to an
  untrusted destination — the write-down channel is the output itself.

Not considering information-flow rules for classified data is V-47. The review question is not only "is
this principal allowed to read this?" but **"where can this data end up once this component has it, and
is every one of those destinations cleared for it?"**

---

## 6. Assurance: the difference between claiming and demonstrating (§20.3, pp. 640–642)

The Common Criteria vocabulary, which is worth borrowing even if you never touch a formal evaluation:

> "The term **target of evaluation (TOE)** refers to that part of the product or system that is subject
> to evaluation." (§20.3, p. 640)

> "**Assurance requirements: The basis for gaining confidence that the claimed security measures are
> effective and implemented correctly.**" (§20.3, p. 640)

> "**Protection profiles (PPs):** Define an **implementation-independent** set of security requirements
> and objectives for a category of products or systems that meet similar consumer needs for IT security."
> (§20.3, p. 641)

> "**Security targets (STs):** Contain the IT security objectives and requirements of **a specific
> identified TOE** and defines the functional and assurance measures offered by that TOE to meet stated
> requirements." (§20.3, p. 641)

> "**The ST may claim conformance to one or more PPs, and forms the basis for an evaluation.**" (§20.3,
> p. 641)

The structure is: a reusable statement of *what any system of this kind must provide* (the profile), a
specific statement of *what this system provides and how* (the target), and *evidence* that the second
satisfies the first. That is exactly the structure of a good security record — a threat model plus a
mapping of required services to mechanisms plus evidence, as
`01-threat-model-and-security-services.md` §5 sets out.

The functional requirement classes (§20.3, p. 641) make a serviceable coverage checklist, because they
are the categories a formal evaluation insists you address:

| Class | Stallings' description |
|---|---|
| **Audit** | "Involves recognizing, recording, storing and analyzing information related to security activities." |
| **Communications** | "Provides two families concerned with **non-repudiation** by the originator and by the recipient of data." |
| **Cryptographic support** | "Used when the TOE implements cryptographic functions." |
| **User data protection** | "requirements relating to the protection of user data within the TOE during **import, export and storage**, in addition to security attributes related to user data." |
| **Identification and authentication** | "Ensure the **unambiguous identification** of authorized users and the correct association of security attributes with users and subjects." |
| **Security management** | "Specifies the management of security attributes, data and functions." |
| **Privacy** | "Provides a user with protection against discovery and misuse of his or her identity by other users." |
| **Protection of the TOE security functions** | "Focused on protection of TSF data, **rather than of user data.**" |
| **Resource utilization** | "Supports the **availability** of required resources, such as processing capability and storage capacity." |
| **TOE access** | "requirements … for **controlling the establishment of a user's session.**" |
| **Trusted path/channels** | "Concerned with **trusted communications paths** between the users and the TSF, and between TSFs." |

Two entries deserve a second look. "Protection of the TOE security functions … rather than of user
data" is the isolation property again, promoted to a first-class requirement: **the security machinery
needs its own protection, separate from the data it protects.** And "Resource utilization … supports the
availability" confirms that availability is a named security requirement, not an operational
afterthought — see `11-malicious-software-and-availability.md` §6.

The assurance classes name what evidence looks like:

> "**Tests:** Concerned with **demonstrating that the TOE meets its functional requirements.**" (§20.3,
> p. 642)

> "**Vulnerability assessment:** Defines requirements directed at the identification of exploitable
> vulnerabilities, which could be introduced by **construction, operation, misuse or incorrect
> configuration** of the TOE." (§20.3, p. 642)

> "**Configuration management:** Requires that **the integrity of the TOE is adequately preserved.**
> Specifically, configuration management provides confidence that **the TOE and documentation used for
> evaluation are the ones prepared for distribution.**" (§20.3, p. 642)

Configuration management is the supply-chain requirement, stated in 2005 terms: **what you evaluated
must be what you shipped.** That is reproducible builds, artifact signing, and pinned dependencies —
see `11-malicious-software-and-availability.md` §5, V-61. And note that vulnerability assessment
explicitly includes *incorrect configuration* and *misuse* alongside construction defects: a securely
written system deployed with verification disabled has a vulnerability, and the fact that the code is
correct is no defence.

**The through-line of the whole chapter, and arguably the book:** verifiability requires that
correctness "must be provable" (§20.2, p. 637), and assurance is "the basis for gaining confidence that
the claimed security measures are **effective and implemented correctly**" (§20.3, p. 640). Both say
the same thing. A security claim without evidence is not a security property. In review terms, that is
the difference between "we validate input" and a named validator on a named path with a negative test
proving the invalid case is refused. Ask for the second, always.

---

## 7. Review checklist

**Boundaries and trust**

- [ ] Does any path reach this component without passing the control that is supposed to protect it —
      internal callers, admin tooling, scheduled jobs, message consumers, replicas, debug routes?
- [ ] Would this component still be safe if an adversary were already inside the network boundary?
      (V-46)
- [ ] Is authentication or authorization skipped anywhere because the caller is "internal" or "trusted"?
- [ ] Is any identity or trust decision derived from an attacker-suppliable header, source address, or
      request field?
- [ ] Is the enforcement point (gateway, mesh, policy service, identity provider) held to a higher
      assurance standard than the services behind it?
- [ ] Is a single control's failure survivable, or is it the only thing standing between an attacker and
      the asset?

**Default posture**

- [ ] Is the default deny? For new routes, new fields in responses, new topics, new IAM statements, new
      CORS origins, new flags — what happens when someone adds one and declares no policy?
- [ ] Is egress restricted to destinations that are actually needed, not just ingress? (Direction
      control.)
- [ ] Are allowed operations enumerated (allowlist), rather than forbidden ones (denylist)?
- [ ] Are security decisions made on fully reassembled, canonicalised, decoded input — once — rather
      than on raw or partial data?
- [ ] Can an attacker-chosen option, parameter, or content type select which parser, handler, or
      validator runs?

**The authorization mechanism itself (reference monitor properties)**

- [ ] **Complete mediation:** is the check on every access, on the enforcement path, with no route around
      it? (V-44)
- [ ] **Isolation:** can the subject being evaluated modify the decision logic, the policy data, the role
      claims, or the cached result?
- [ ] **Verifiability:** is the policy expressed somewhere it can be read and enumerated, and are there
      negative tests asserting the wrong principal is refused?
- [ ] Are policy changes — grants, role assignments, permission edits — recorded as security events?
      (V-55)

**Information flow**

- [ ] For each component holding sensitive data: what are all the places that data can go, and is every
      destination cleared for it? (V-47)
- [ ] Does any component combine read access to sensitive data with a write path to a
      lower-classification sink — logs, metrics labels, error trackers, analytics, external APIs,
      caller-supplied URLs, model outputs? (V-58)
- [ ] Does production data flow into environments with a weaker access model, and is it masked or
      synthetic if so?
- [ ] Is any privileged process acting on content supplied by a less-privileged party — templates,
      formulas, webhook targets, deserialised objects, prompts?
- [ ] Is the question being asked "may this principal read it?" *and* "where may it go afterwards?"

**Assurance**

- [ ] Which security requirements does this change claim to meet, and what is the evidence — a test, a
      named enforcement point, a verified configuration — rather than an assertion?
- [ ] Are there negative tests for each security control, not only happy-path tests?
- [ ] Is the artifact that was reviewed the artifact that ships: pinned dependencies, verified
      toolchain, signed build?
- [ ] Is deployed configuration covered, given that misconfiguration is a vulnerability in its own right?
- [ ] Does the security machinery have protection distinct from the data it protects — its own
      credentials, its own audit trail, its own availability target?
