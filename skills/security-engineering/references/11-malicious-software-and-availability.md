# Malicious Software, Untrusted Input, and Availability

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 19 (Malicious Software,
p. 598): §19.1 Viruses and Related Threats, §19.2 Virus Countermeasures, §19.3 Distributed Denial of
Service Attacks.

This chapter is about code you did not intend to run and requests you did not intend to serve. Three
of its ideas are load-bearing in a modern review: the **backdoor**, which is the security defect most
likely to be introduced deliberately and then forgotten; the **supply-chain argument** — that code
arrives from somewhere and its arrival path is a trust decision; and **availability as a security
service**, which Chapter 1 lists alongside confidentiality and integrity but which most reviews treat
as someone else's concern.

---

## 1. The taxonomy, and the two axes that matter (§19.1, p. 600)

> "The terminology in this area presents problems because of a lack of universal agreement on all of the
> terms and because some of the categories overlap." (§19.1, p. 599)

The definitions worth having exactly:

> "**Backdoor (trapdoor):** Program modification that allows unauthorized access to functionality"
> (§19.1, p. 600)

> "**Logic bomb:** Triggers action when condition occurs" (§19.1, p. 600)

> "**Trojan horse:** Program that contains **unexpected additional functionality**" (§19.1, p. 600)

> "**Virus:** Attaches itself to a program and propagates copies of itself to other programs" (§19.1,
> p. 600)

> "**Worm:** Program that propagates copies of itself to other computers" (§19.1, p. 600)

Two classification axes, both useful:

> "Malicious software can be divided into two categories: **those that need a host program, and those
> that are independent.** … Viruses, logic bombs, and backdoors are examples." (§19.1, p. 600)

> "We can also differentiate between those software threats that **do not replicate** and those that
> **do.**" (§19.1, p. 600)

**The axes tell you which control applies.** Host-dependent code is introduced by modifying something
you already run, so the control is integrity of your artifacts and their sources (§4). Independent code
arrives on its own, so the control is what you expose and what you accept (§5). Replication is what
turns a single exploitable instance into an availability event — which is the §6 argument.

---

## 2. Backdoors: the defect you introduce on purpose (§19.1, pp. 600–601)

> "A **backdoor**, also known as a trapdoor, is a **secret entry point into a program that allows
> someone that is aware of the backdoor to gain access without going through the usual security access
> procedures.**" (§19.1, p. 600)

Then the part that makes this a code-review finding rather than a malware topic — the motivation is
entirely legitimate:

> "**Programmers have used backdoors legitimately for many years to debug and test programs.** This
> usually is done when the programmer is developing an application that has an authentication procedure,
> or a long setup, requiring the user to enter many different values to run the application. **To debug
> the program, the developer may wish to gain special privileges or to avoid all the necessary setup and
> authentication.**" (§19.1, p. 600)

> "The backdoor is code that recognizes some **special sequence of input** or is triggered by being run
> from a **certain user ID** or by an **unlikely sequence of events.**" (§19.1, p. 600)

> "**Backdoors become threats when unscrupulous programmers use them to gain unauthorized access.**"
> (§19.1, p. 601)

That paragraph describes normal development. Every one of these is the same defect:

- A magic token, header, or query parameter that bypasses authentication.
- An `if (userId === 'test-user')` or `if (env !== 'production')` branch that skips a check.
- A debug endpoint that returns internal state, impersonates a user, or executes arbitrary queries.
- A seeded administrative account with a known password (see `09-authentication-and-access-control.md`
  §4, V-54).
- A feature flag that disables authorization, defaulting to permissive when the flag service is
  unreachable.
- Verification disabled for a local or test setup — `verify=False`, `InsecureSkipVerify` — where the
  configuration can reach production (V-32).

This is V-60, and its severity does not depend on intent. The three properties Stallings names —
triggered by a special input, or a particular identity, or an unlikely sequence — are exactly the
properties that make it survive review and testing: nothing in the normal path touches it.

The controls are stated, and they are the correct ones:

> "**It is difficult to implement operating system controls for backdoors.**" (§19.1, p. 601)

> "**Security measures must focus on the program development and software update activities.**" (§19.1,
> p. 601)

**You cannot detect this at runtime; you have to prevent it at authorship.** That means the diff is the
control point. The practical rule: a bypass may exist only if it is (a) impossible to enable in
production by configuration — compiled out, or gated on a build that production cannot be — and (b)
covered by a test that asserts it is off. "It's only for local dev" is a claim about intent, not about
reachability.

And the limit case, which is worth knowing because it bounds how much assurance code review can
provide:

> "An example of a Trojan horse program that would be difficult to detect is **a compiler that has been
> modified to insert additional code into certain programs as they are compiled**, such as a system
> login program [THOM84]. The code creates a backdoor in the login program that permits the author to log
> on to the system using a special password. **This Trojan horse can never be discovered by reading the
> source code of the login program.**" (§19.1, p. 601)

> "The threat was so well implemented that **the Multics developers could not find it, even after they
> were informed of its presence** [ENGE80]." (§19.1, p. 601)

Ken Thompson's trusting-trust attack. **Reviewing source is not sufficient assurance if the toolchain
is untrusted** — which is the argument for reproducible builds, pinned and verified toolchains, and
treating your CI system as a production system with production-grade access control. If an attacker can
modify the build, reviewing the code proves nothing.

---

## 3. Trojan horses and borrowed authority (§19.1, p. 601)

> "A **Trojan horse** is a useful, or apparently useful, program or command procedure containing hidden
> code that, when invoked, performs some unwanted or harmful function." (§19.1, p. 601)

> "**Trojan horse programs can be used to accomplish functions indirectly that an unauthorized user
> could not accomplish directly.**" (§19.1, p. 601)

> "For example, to gain access to the files of another user on a shared system, a user could create a
> Trojan horse program that, when executed, **changed the invoking user's file permissions** so that the
> files are readable by any user. The author could then induce users to run the program by placing it in
> a common directory and naming it such that it appears to be a useful utility." (§19.1, p. 601)

The middle sentence is the general principle and it is worth restating in modern terms: **the attacker
does not need the privilege, only something privileged that will act on their input.** That is the
confused-deputy problem, and it is the shape of a large fraction of real vulnerabilities:

- A server-side request made to a URL supplied by the caller — the server's network position is the
  privilege being borrowed (SSRF).
- A background job that runs with elevated rights over a payload a user submitted.
- A template, deserialiser, or expression evaluator that executes attacker-supplied content with the
  process's authority.
- A CI pipeline that runs a build script from a pull request, holding deploy credentials.
- An admin tool that performs an action on behalf of a user without re-checking that user's own
  permission.

The review question for every privileged component is: **whose input does it act on, and does it apply
that requester's authority or its own?** See `12-perimeter-and-trusted-systems.md` §5 for the worked
multilevel-security version of this attack, where the access control list permits the write and the
policy denies it anyway.

---

## 4. Propagation, and what it says about your inputs (§19.1, pp. 602–609)

The virus mechanism, in one sentence:

> "A virus can do anything that other programs do. **The only difference is that it attaches itself to
> another program and executes secretly when the host program is run.**" (§19.1, p. 602)

> "The key to its operation is that **the infected program, when invoked, will first execute the virus
> code and then execute the original code of the program.**" (§19.1, p. 603)

The four phases (§19.1, p. 602) — dormant, propagation, triggering, execution — are the reason a
compromise is not a single event. Note especially the dormant and triggering phases: "The virus will
eventually be activated by some event, such as a date, the presence of another program or file, or the
capacity of the disk exceeding some limit" (§19.1, p. 602). **Presence and activation are separate in
time**, which is why "nothing happened" is not evidence that nothing was installed, and why a logic
bomb ("triggers action when condition occurs", p. 600) can sit in a codebase across many releases.

The worm's replication loop (§19.1, p. 607):

> "1. **Search for other systems to infect** by examining host tables or similar repositories of remote
> system addresses. 2. **Establish a connection** with a remote system. 3. **Copy itself to the remote
> system and cause the copy to be run.**"

The Morris worm's three entry techniques are a compact catalogue of vulnerability classes, and all
three are still live:

> "The worm performed this task by examining a variety of lists and tables, including **system tables
> that declared which other machines were trusted by this host**, users' mail forwarding files, tables
> by which users gave themselves permission for access to remote accounts, and from a program that
> reported the status of network connections." (§19.1, p. 608)

> "It exploited **a bug in the finger protocol**, which reports the whereabouts of a remote user."
> (§19.1, p. 608)

> "It exploited **a trapdoor in the debug option of the remote process that receives and sends mail.**"
> (§19.1, p. 608)

> "If any of these attacks succeeded, **the worm achieved communication with the operating system command
> interpreter.**" (§19.1, p. 608)

Read them in order. **Trust relationships are a map for lateral movement** — the worm read the host's
own trust configuration to find its next targets, which is precisely what an attacker does with your
service account grants, peer allowlists, and mutual-trust configuration. **A parsing bug in an
information endpoint was remote code execution.** And **a debug trapdoor left in a mail daemon was a
production entry point** — §2, demonstrated at internet scale in 1988.

The last line is the one to keep: the goal of all three was to reach an interpreter. That is V-59,
stated as an objective rather than a category. Untrusted input reaching a command interpreter, a SQL
parser, a template engine, a deserialiser, an expression evaluator, or a memory operation is the same
finding with different syntax. The 2003 example makes the memory case explicit: "the SQL Slammer worm
appeared. **This worm exploited a buffer overflow vulnerability** in Microsoft SQL server" (§19.1,
p. 609).

And speed, which is the availability consequence:

> "The Slammer was extremely compact and spread rapidly, **infecting 90% of vulnerable hosts within 10
> minutes.**" (§19.1, p. 609)

> "In the second wave of attack, **Code Red infected nearly 360,000 servers in 14 hours.**" (§19.1,
> p. 608)

> "In addition to the havoc it causes at the targeted server, **Code Red can consume enormous amounts of
> Internet capacity, disrupting service.**" (§19.1, p. 608)

> "Mydoom replicated up to 1000 times per minute and reportedly flooded the Internet with 100 million
> infected messages in 36 hours." (§19.1, p. 609)

> "**Transport vehicles:** Because worms can rapidly compromise a large number of systems, they are
> ideal for spreading other distributed attack tools, such as **distributed denial of service zombies.**"
> (§19.1, p. 609)

**Ten minutes is shorter than your patch cycle.** Any control whose latency is measured in days —
manual patching, quarterly dependency review, a human approving an upgrade — does not engage against a
threat that saturates in minutes. That is the argument for automated dependency updates, for a known
inventory of what you run, and for the exposure surface being small enough to reason about.

---

## 5. Countermeasures, and the honest limits of each (§19.2, pp. 610–614)

The framing:

> "**The ideal solution to the threat of viruses is prevention:** Do not allow a virus to get into the
> system in the first place. **This goal is, in general, impossible to achieve**, although prevention can
> reduce the number of successful viral attacks." (§19.2, p. 610)

> "The next best approach is to be able to do the following: **Detection** … **Identification** …
> **Removal** … " (§19.2, p. 610)

> "**Advances in virus and antivirus technology go hand in hand.**" (§19.2, p. 610)

> "**The arms race continues.** With fourth-generation packages, a more comprehensive defense strategy is
> employed, **broadening the scope of defense to more general-purpose computer security measures.**"
> (§19.2, p. 611)

The generational progression (§19.2, pp. 610–611) — "First generation: simple scanners; Second
generation: heuristic scanners; Third generation: activity traps; Fourth generation: full-featured
protection" — maps directly onto how dependency and artifact security works now:

- **Signature scanning.** "A first-generation scanner **requires a virus signature to identify a
  virus**" (§19.2, p. 610). This is your CVE scanner: precise, and blind to anything not yet published.
  Necessary, insufficient, and its coverage is bounded by someone else's disclosure timeline.
- **Behaviour at runtime.** "Third-generation programs are memory-resident programs that identify a
  virus **by its actions rather than its structure**" (§19.2, p. 611).

The behaviour-blocking argument is the strongest general point in the chapter, and it generalises well
past malware:

> "While there are literally **trillions of different ways to obfuscate and rearrange the instructions**
> of a virus or worm, many of which will evade detection by a fingerprint scanner or heuristic,
> **eventually malicious code must make a well-defined request to the operating system.**" (§19.2,
> p. 613)

> "Given that the behavior blocker can intercept all such requests, it can **identify and block malicious
> actions regardless of how obfuscated the program logic appears to be.**" (§19.2, p. 613)

**Constrain the effects, not the inputs.** There are unbounded ways to write a malicious payload and a
small, enumerable set of things it must eventually do: open a network connection, write a file, spawn a
process, read a credential. Controls placed on that narrow set are far more robust than controls that
try to recognise bad input. In application terms that is egress restrictions, read-only filesystems,
dropped capabilities, seccomp profiles, and least-privileged credentials — the bastion host properties
of `09-authentication-and-access-control.md` §8. It is also the argument for allowlisting the small set
of legitimate destinations for a server-side fetch rather than blocklisting the bad ones.

With the stated limitation, which is equally important:

> "**Since the malicious code must actually run on the target machine before all its behaviors can be
> identified, it can cause a great deal of harm to the system before it has been detected and blocked**
> by the behavior blocking system." (§19.2, p. 614)

Runtime containment reduces blast radius; it does not prevent execution. So the artifact's *arrival*
path still needs its own control: signed and verified packages, pinned versions with lockfiles,
integrity hashes, verified base images, and update mechanisms that check a signature before installing.
An update channel that accepts an unverified artifact is a remote code execution path by design — V-61.
Note that the immune-system design Stallings describes depends on exactly this: it "**passes information
about that virus to systems** running IBM AntiVirus so that it can be detected before it is allowed to
run elsewhere" (§19.2, p. 612) — a distribution channel whose integrity is the whole basis of the
defence.

---

## 6. Availability is a security service (§19.3, pp. 614–619)

> "A **denial of service (DoS) attack** is an attempt to prevent legitimate users of a service from using
> that service." (§19.3, p. 614)

> "**A DDoS attack attempts to consume the target's resources so that it cannot provide service.**"
> (§19.3, p. 614)

> "Broadly speaking, **the resource consumed is either an internal host resource on the target system or
> data transmission capacity** in the local network to which the target is attacked." (§19.3, p. 614)

That two-way split is the useful part. **Bandwidth exhaustion is your provider's problem; resource
exhaustion is yours, and it is a code review finding.** The canonical example is stated in a way that
transfers directly:

> "A simple example of an internal resource attack is the **SYN flood attack.**" (§19.3, p. 614)

> "The slave hosts begin sending TCP/IP SYN packets, **with erroneous return IP address information**,
> to the target." (§19.3, p. 614)

> "**The Web server maintains a data structure for each SYN request waiting for a response back and
> becomes bogged down as more traffic floods in.**" (§19.3, p. 616)

> "The result is that **legitimate connections are denied while the victim machine is waiting to complete
> bogus 'half-open' connections.**" (§19.3, p. 616)

The pattern, abstracted: **an unauthenticated request causes the server to allocate state and hold it,
and the attacker never has to complete the transaction.** Every instance in application code has this
shape — a connection pool slot held for the duration of a slow client, an unbounded in-memory buffer
sized by a client-supplied length, a session or upload record created before authentication, a
websocket accepted and never idle-timed out, a queue consumer that reads an entire message of arbitrary
size. That is V-63; the Oakley cookie in `08-transport-and-channel-security.md` §8 is the fix pattern:
require a cheap, requester-bound round trip before committing real resources.

Its sibling is work performed rather than state held — V-62. Anything whose cost is controlled by the
requester and not bounded by the server: an unbounded page size, a regex over attacker-supplied input
with catastrophic backtracking, a nested query or deeply nested JSON/XML expanded server-side, an image
or archive decompressed without a size limit, a report generated synchronously, a password hash
computed at attacker-chosen cost. **Every request must have a bound on the work it can cause, and the
bound must be the server's decision.**

Amplification makes both worse, and the mechanism is worth understanding because you can be either the
victim or the amplifier:

> "The attacker takes control of multiple hosts over the Internet, instructing them to send ICMP ECHO
> packets **with the target's spoofed IP address** to a group of hosts that act as **reflectors.**"
> (§19.3, p. 616)

> "Nodes at the bounce site receive multiple spoofed requests and respond by sending echo reply packets
> to the target site. **The target's router is flooded with packets from the bounce site, leaving no data
> transmission capacity for legitimate traffic.**" (§19.3, p. 616)

> "**A reflector DDoS attack can easily involve more machines and more traffic than a direct DDoS attack
> and hence be more damaging.** Further, **tracing back the attack or filtering out the attack packets is
> more difficult** because the attack comes from widely dispersed uninfected machines." (§19.3, p. 617)

> "In this type of attack, the slave zombies construct packets requiring a response that contain the
> target's IP address as the source IP address… These packets are sent to **uninfected machines known as
> reflectors.** The uninfected machines respond with packets directed at the target machine." (§19.3,
> p. 617)

**The reflectors are not compromised.** They are correctly implemented services doing what they were
asked. Any endpoint of yours that takes a small request and produces a large response, or that sends a
request somewhere the caller names, can be used against a third party — an email or SMS sender, a
webhook registration, a URL preview generator, a "notify this address" feature, a password reset that
sends mail to any submitted address. **Being the amplifier is your finding too**, and the controls are
the same: authenticate, rate limit per principal *and* per target, and cap the amplification factor.

The attack network's prerequisites, which are a decent summary of why patch latency matters:

> "**A vulnerability in a large number of systems.** The attacker must become aware of a vulnerability
> that **many system administrators and individual users have failed to patch** and that enables the
> attacker to install the zombie software." (§19.3, p. 618)

> "In the scanning process, the attacker first seeks out a number of vulnerable machines and infects
> them. **Then, typically, the zombie software that is installed in the infected machines repeats the
> same scanning process**, until a large distributed network of infected machines is created." (§19.3,
> p. 618)

> "**Local subnet:** If a host can be infected behind a firewall, that host then looks for targets in
> **its own local network.**" (§19.3, p. 618)

That last one is lateral movement in a sentence, and it is the answer to "this service is not exposed
to the internet." A host inside the boundary scans the boundary's inside. See V-46.

The defensive picture is deliberately modest:

> "In general, there are **three lines of defense** against DDoS attacks: **Attack prevention and
> preemption** (before the attack): These mechanisms enable the victim to **endure attack attempts
> without denying service to legitimate clients.** … **Attack detection and filtering** (during the
> attack) … **Attack source traceback and identification** (during and after the attack)." (§19.3,
> pp. 618–619)

> "However, this method **typically does not yield results fast enough, if at all, to mitigate an ongoing
> attack.**" (§19.3, p. 619)

> "**The challenge in coping with DDoS attacks is the sheer number of ways in which they can operate.**
> Thus DDoS countermeasures must **evolve with the threat.**" (§19.3, p. 619)

> "defense technologies have been **unable to withstand large-scale attacks.**" (§19.3, p. 614)

Note how the first line of defence is phrased: not "stop the attack" but **"endure attack attempts
without denying service to legitimate clients."** That is a design property, and it is what a review can
actually assess. Does the system degrade or collapse? Is there a bound on concurrent work, a queue with
a rejection policy, a timeout on every external call, a circuit breaker, backpressure rather than
unbounded buffering, and load shedding that preserves the highest-value traffic? A system with no
defined behaviour at saturation has chosen collapse — and if a security dependency (the auth service,
the policy service, the key service) is what saturates, an undefined degraded mode is V-64. See
`07-identity-certificates-and-pki.md` §1: "lack of availability of the Kerberos service means lack of
availability of the supported services."

---

## 7. Review checklist

**Untrusted input**

- [ ] Does any untrusted value reach a command interpreter, SQL parser, template engine, expression
      evaluator, deserialiser, XML/YAML parser, or memory operation? (V-59)
- [ ] Is the boundary enforced by construction — parameterised queries, safe deserialisation, an
      allowlisted template context — rather than by escaping or validation alone?
- [ ] Is input validated at the point where it can still be refused, and against an allowlist rather
      than a denylist?
- [ ] For any privileged component: whose input does it act on, and does it apply the requester's
      authority or its own? (Confused deputy — SSRF, background jobs, admin tooling, CI running
      PR-supplied code.)

**Backdoors and development shortcuts**

- [ ] Any magic token, header, parameter, or user id that bypasses authentication or authorization?
      (V-60)
- [ ] Any environment- or flag-conditional check that can be disabled in production by configuration?
- [ ] Any debug, diagnostic, or impersonation endpoint reachable in production?
- [ ] Any seeded, fixture, or default credential? (V-54)
- [ ] Does a flag or policy service failing open disable a security control? (V-64)
- [ ] Is every intentional bypass impossible to enable in production *and* covered by a test asserting
      it is off?

**Supply chain and artifact integrity**

- [ ] Are dependencies pinned with a lockfile and integrity hashes, from a source whose authenticity is
      verified?
- [ ] Are updates and executable content signature-verified before installation? (V-61)
- [ ] Are base images and build tooling pinned and verified — and is CI treated as a production system
      with production access controls?
- [ ] Is there an inventory of what is actually deployed, and can an urgent upgrade be shipped in hours
      rather than weeks?
- [ ] Is dependency scanning present but understood as necessary-and-insufficient, given that it only
      finds published issues?

**Runtime containment**

- [ ] Does the workload run unprivileged, with a read-only filesystem where possible and unnecessary
      capabilities dropped?
- [ ] Is outbound network access restricted to an allowlist of destinations it actually needs?
- [ ] Are credentials scoped so a compromise of this component does not grant reach across the system?
- [ ] Are the *effects* constrained (what it can connect to, write, or execute), not only the inputs
      filtered?

**Availability**

- [ ] Does any unauthenticated request cause the server to allocate and hold state — a connection, a
      buffer sized by the client, a session, a pending record? (V-63)
- [ ] Is there a bound on the work a single request can cause: page size, result set, recursion and
      nesting depth, decompressed size, regex complexity, external calls fanned out? (V-62)
- [ ] Does every external call have a timeout, and is every queue and buffer bounded with an explicit
      rejection policy?
- [ ] Is expensive work gated behind a cheap, requester-bound proof of round-trip?
- [ ] Are rate limits keyed per principal *and* per target, not only per source address?
- [ ] Can this service be used as an amplifier or reflector against a third party — small request, large
      or caller-directed response (mail, SMS, webhooks, URL previews, notifications)?
- [ ] Is there defined behaviour at saturation — load shedding, backpressure, degraded mode — rather
      than collapse?
- [ ] When a security dependency (auth, policy, key management) is slow or unavailable, is the behaviour
      defined, and does it fail closed unless a different choice is explicitly justified? (V-64)
- [ ] Is the availability target of a security dependency at least as high as that of everything relying
      on it?
