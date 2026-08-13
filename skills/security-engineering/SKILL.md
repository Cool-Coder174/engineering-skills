---
name: security-engineering
description: Security decision engine for software that handles secrets, identity, or untrusted input, grounded in Cryptography and Network Security (Stallings). Use when a task touches encryption, hashing, MACs, signatures, random number generation, key management, passwords, sessions, tokens, authentication or authorization, certificates and TLS, access control, audit logging, untrusted input, or availability under attack. Provides a threat model, decision tables, a named-vulnerability catalog, and deep-dive references per topic.
---

# SECURITY ENGINEERING ENGINE

**ROLE:** Principal security engineer.
**CORE FUNCTION:** Convert vague security intentions ("it's encrypted", "we validate
input", "it's behind the VPN") into explicit, named, defensible decisions — and reject
designs that claim a security service the chosen mechanism does not actually provide.

This skill is a **knowledge and gate skill**. It does not run a workflow. It is consulted by
`planner`, `detail-planning`, `implement`, `verify`, and `code-review` (which invokes it
directly in `/review-security` mode), and can be invoked on its own for design questions.

---

# 1. ACTIVATION

Activate whenever the work involves any of:

| Trigger area | Examples |
|---|---|
| Cryptography | encrypt, decrypt, cipher, AES, mode of operation, IV, nonce, KDF |
| Integrity | hash, digest, checksum, MAC, HMAC, signature, tamper-proof |
| Randomness | token, salt, nonce, session ID, `random`, `uuid`, invite code |
| Key material | API key, private key, secret, credential, `.env`, KMS, rotation |
| Identity | login, signup, session, JWT, OAuth, SSO, MFA, password reset |
| Authorization | roles, permissions, tenant isolation, admin action, ownership check |
| Transport | TLS, certificate, pinning, webhook, service-to-service call, proxy |
| Untrusted input | user input, upload, deserialization, query building, subprocess, parser |
| Detection | audit log, security event, alerting, anomaly, incident response |
| Availability | rate limit, quota, timeout, expensive endpoint, amplification |

If none of the above apply, this skill stays silent. **Do not apply cryptographic ceremony
to a change with no secrets, no identity, and no untrusted input.** See Section 8
(Proportionality Rule).

---

# 2. THE FRAME: ATTACKS, SERVICES, MECHANISMS (Ch. 1, p. 12)

X.800 separates three things that are constantly conflated in practice. Keeping them
separate is most of the discipline.

| Term | Definition (§1.2, p. 12) | In review terms |
|---|---|---|
| **Security attack** | "Any action that compromises the security of information owned by an organization" | What the adversary does |
| **Security mechanism** | "A process … designed to detect, prevent, or recover from a security attack" | The code you wrote |
| **Security service** | "A processing or communication service that enhances the security of the data processing systems and the information transfers" | The guarantee you are claiming |

A **threat** is "a possible danger that might exploit a vulnerability"; an **attack** is a
deliberate attempt to "evade security services and violate the security policy" (§1.2,
p. 12, from RFC 2828).

## 2.1 The six security services (§1.4, p. 17)

Every security requirement must name which of these it is buying. A requirement that names
none is not a requirement.

| Service | What it guarantees | Typical mechanism |
|---|---|---|
| **Authentication** | "The assurance that the communicating entity is the one that it claims to be" — split into *peer entity* authentication (a connection) and *data origin* authentication (a single message) | Signature, MAC, authentication exchange |
| **Access control** | "The prevention of unauthorized use of a resource … who can have access, under what conditions, and what those accessing are allowed to do" | Reference monitor, ACL, capability |
| **Data confidentiality** | "The protection of data from unauthorized disclosure", including traffic-flow confidentiality | Encipherment, traffic padding |
| **Data integrity** | "The assurance that data received are exactly as sent … contain no modification, insertion, deletion, or replay" | MAC, signature, sequence numbers |
| **Nonrepudiation** | Protection against a party denying having participated | Digital signature, notarization |
| **Availability** | The property of "being accessible and usable upon demand by an authorized system entity" | Rate limiting, access control, redundancy |

Two consequences worth stating in reviews:

- **Data origin authentication "does not provide protection against the duplication or
  modification of data units"** (§1.4, p. 18). A valid signature does not make a message
  fresh. Replay protection is separate work — see §3.5.
- Integrity "relates to active attacks, [so] we are concerned with detection rather than
  prevention" (§1.4, p. 19). Design the detection *and* the response; a detected violation
  with no handler is a crash, not a control.

## 2.2 Passive vs active attacks (§1.3, p. 13)

This distinction sets your whole defensive posture, and it is the reason "we'll monitor for
it" is sometimes a valid answer and sometimes not.

| Class | Members | Posture |
|---|---|---|
| **Passive** — "attempts to learn or make use of information from the system but does not affect system resources" | Release of message contents; **traffic analysis** | "Very difficult to detect because they do not involve any alteration of the data," so "the emphasis in dealing with passive attacks is on **prevention** rather than detection" (p. 13) |
| **Active** — "attempts to alter system resources or affect their operation" | **Masquerade**, **replay**, **modification of messages**, **denial of service** | "Quite difficult to prevent … absolutely, because of the wide variety of potential physical, software, and network vulnerabilities. Instead, the goal is to **detect** … and to recover" (p. 15) |

So: confidentiality must be *prevented* by construction (encryption), because you will never
see the breach. Integrity, authenticity, and availability must additionally be *detectable*,
because prevention will eventually fail.

**Traffic analysis is a real service, not a footnote.** Even with perfect encryption, an
opponent "could determine the location and identity of communicating hosts and could observe
the frequency and length of messages" (§1.3, p. 13). If message existence, size, or timing is
itself sensitive, encryption alone does not cover it.

Deep dive: `references/01-threat-model-and-security-services.md`

## 2.3 The four design tasks (§1.6, p. 23)

Any security service requires all four. Reviews routinely find the first done and the rest
assumed.

1. Design the algorithm performing the security transformation.
2. **Generate** the secret information.
3. **Develop methods for the distribution and sharing** of the secret information.
4. **Specify the protocol** that uses the algorithm and the secret to achieve the service.

Most real defects live in 2, 3, and 4 — not in 1. The algorithm is a library call; the key
lifecycle and the protocol are yours.

---

# 3. DECISION TABLES

Use these to decide quickly, then read the linked reference before committing to anything
expensive or hard to reverse. Entries marked **[Modern]** name mechanisms that postdate
Stallings (4th ed., 2005); the reasoning above them is his.

## 3.1 What am I actually protecting? (Ch. 1)

| If the requirement is… | The service is… | Mechanism | Not sufficient |
|---|---|---|---|
| "Nobody should be able to read this" | Confidentiality | Encryption with a correct mode | Access control alone; obfuscation; encoding |
| "Nobody should be able to change this undetected" | Integrity | MAC or signature | Encryption alone (§11.2, p. 321); CRC |
| "I need to know who sent this" | Data origin authentication | MAC (shared key) or signature (public key) | A username field; a source IP |
| "They must not be able to deny it" | Nonrepudiation | **Signature only** — a MAC cannot do this, since "both sender and receiver share the same key" (§11.2, p. 327) | HMAC |
| "This must not be usable twice" | Integrity + freshness | Nonce or timestamp, checked and recorded | A signature |
| "Only permitted users may do this" | Access control | Server-side check on the enforcement path | Client-side check; network position |
| "It must stay up under attack" | Availability | Quotas, cheap-before-expensive, load shedding | Autoscaling |

## 3.2 Symmetric encryption and mode (Ch. 6, p. 181)

| Need | Mode | Why / hazard |
|---|---|---|
| A single block, e.g. wrapping a key | ECB | Stallings' only sanctioned use: "ideal for a short amount of data, such as an encryption key" (p. 182) |
| General block-oriented data | CBC with a fresh unpredictable IV | Repeating plaintext no longer produces repeating ciphertext; IV must be unpredictable *and* protected (p. 184) |
| Stream / byte-at-a-time | CFB | Chained, so errors propagate |
| Stream over a noisy channel | OFB | Bit errors don't propagate, but it "is more vulnerable to a message stream modification attack than is CFB" (p. 187) |
| High throughput, random access | CTR | Only requirement: "the counter value must be different for each plaintext block that is encrypted" (p. 187). Parallelizable, preprocessable, encryption-only, "provably secure" (pp. 188–189) |
| **[Modern]** Anything new | AES-GCM / ChaCha20-Poly1305 | Confidentiality and integrity together, so §3.3's error becomes structurally impossible |

**Two rules that catch most real defects:**

1. **Never reuse a keystream.** "If two plaintexts are encrypted with the same key using a
   stream cipher … if the two ciphertext streams are XORed together, the result is the XOR of
   the original plaintexts" (§6.3, p. 190). This applies to CTR/OFB/CFB and every stream
   cipher, and it is triggered by a repeated nonce, not just a repeated key.
2. **Don't invent key derivation.** WEP failed not in RC4 but in "the way in which keys are
   generated for use as input to RC4" (§6.3, p. 193).

Deep dive: `references/02-cryptographic-primitives-and-modes.md`

## 3.3 Encryption is not integrity (Ch. 11, p. 320)

The single most common crypto defect in application code. Stallings' reasoning:

> "If M can be any bit pattern, then regardless of the value of X, the value Y = D(K, X) is
> some bit pattern and therefore must be accepted as authentic plaintext." (§11.2, p. 321)

Encryption authenticates only if valid plaintext is a recognizably small subset of all bit
patterns — and then only against random corruption, not against a chosen edit. In malleable
modes the edit is precise: "complementing a bit in the ciphertext complements the
corresponding bit in the recovered plaintext" (§6.2, p. 187).

| If you… | You get | You still need |
|---|---|---|
| Encrypt with a shared key | Confidentiality + a *degree* of authentication among key holders; **no signature** — "receiver could forge message, sender could deny message" (Table 11.1, p. 325) | A MAC if the plaintext is unstructured |
| Encrypt with the recipient's public key | Confidentiality only — "any party could use PU_b to encrypt message and claim to be A" (p. 325) | A signature |
| Sign with your private key | Authentication + signature; **no confidentiality** | Encryption |
| Append a CRC and encrypt | Internal error control — usable | Nothing, if the check is inside |
| Encrypt and append a CRC outside | **Broken** — "an opponent can construct messages with valid error-control codes" (§11.2, p. 323) | Move the check inside the authenticated envelope |

**Ordering rules.** Prefer authentication tied to the plaintext, `E(K₂, [M ‖ C(K₁, M)])`
(§11.2, p. 326). For signatures, sign first and encrypt second, so a dispute can be resolved
without handing the arbiter your decryption key (§13.1, p. 380).

Deep dive: `references/05-integrity-hashes-and-macs.md`

## 3.4 Hash, MAC, or signature? (Ch. 11–13)

| Property required | Use | Cost |
|---|---|---|
| Detect accidental corruption | Unkeyed hash | Anyone can recompute it — no attacker resistance |
| Detect deliberate modification, shared secret acceptable | **HMAC** or **CMAC** | Cheap; no nonrepudiation |
| Detect deliberate modification, verifier must not be able to forge | Signature | Public-key cost |
| Resolve disputes with a third party | Signature (+ possibly an arbiter, §13.1, p. 380) | Key management, revocation |

**Brute-force effort** (§11.5, pp. 341–342), the numbers that decide output length:

| Property | Effort | Means |
|---|---|---|
| One-way (preimage) | 2ⁿ | Given a digest, find an input |
| Weak collision resistance (2nd preimage) | 2ⁿ | Given *x*, find *y ≠ x* with the same digest |
| **Strong collision resistance** | **2^(n/2)** | Find *any* colliding pair — the birthday attack |
| MAC forgery | min(2^k, 2ⁿ) | Requires text–MAC pairs; Stallings wants min(k, n) ≥ ~128 |

The birthday bound is the one that surprises people: it halves your effective security
whenever the attacker controls **both** inputs (§11.4, p. 338). Decide which property you
need before choosing a digest length.

**Never key a hash by concatenation.** Use HMAC, which was chosen among many proposals
because it alone has "a well understood cryptographic analysis of the strength of the
authentication mechanism" (§12.3, p. 368). And never use CBC-MAC on variable-length
messages: "given the CBC MAC of a one-block message X, say T = MAC(K, X), the adversary
immediately knows the CBC MAC for the two-block message X‖(X ⊕ T)" (§12.4, p. 372).

Deep dive: `references/05-integrity-hashes-and-macs.md`

## 3.5 Freshness: the mechanism must be chosen, not assumed (Ch. 13, p. 382)

Confidentiality and **timeliness** are the two central issues in authenticated exchange, and
timeliness is the one that gets skipped. Replay "could allow an opponent to compromise a
session key or successfully impersonate another party. At minimum, a successful replay can
disrupt operations by presenting parties with messages that appear genuine but are not"
(§13.2, p. 382).

| Mechanism | Requires | Fails when | Suited to |
|---|---|---|---|
| **Sequence numbers** | Per-peer state of the last number seen | State is lost or windows wrap | Rarely used for auth — the bookkeeping "overhead" rules it out (p. 383) |
| **Timestamps** | Synchronized clocks; a justified window | Clocks drift or are attacked → **suppress-replay** (p. 385) | Connectionless / one-way messages |
| **Challenge/response (nonce)** | A round trip; an *unpredictable* nonce | You cannot afford the handshake | Connection-oriented exchanges |

Stallings' own guidance: timestamps "should not be used for connection-oriented
applications" because of clock-sync fragility, while challenge/response "is unsuitable for a
connectionless type of application because it requires the overhead of a handshake"
(§13.2, p. 383). Pick per flow, and write down which.

Deep dive: `references/06-signatures-and-authentication-protocols.md`

## 3.6 Randomness and keys (Ch. 7, pp. 218–227)

**Randomness and unpredictability are two different requirements** (§7.4, p. 220). Statistical
quality — uniform distribution and independence — is what simulations need. Security needs
unpredictability, and the standard is the **next-bit test**: no polynomial-time algorithm can
predict bit *k+1* from the first *k* with probability meaningfully better than ½ (§7.4,
p. 225).

A linear congruential generator fails this catastrophically: knowing the parameters, "once a
single number is discovered, all subsequent numbers are known," and four consecutive outputs
are enough to solve for the parameters themselves (§7.4, p. 222).

| Value | Requirement |
|---|---|
| Key, IV/nonce, salt, session ID, token, reset code | CSPRNG, OS-seeded |
| Challenge nonce | CSPRNG — must be unpredictable *to the peer* (§13.2, p. 385) |
| Test fixture, sharding, jitter | Ordinary PRNG is fine |

**Key hierarchy.** Separate a long-lived **master key**, which only wraps or derives, from
short-lived **session keys**: "the more frequently session keys are exchanged, the more
secure they are, because the opponent has less ciphertext to work with" (§7.3, p. 214).

**Key separation.** Give every key exactly one purpose. Stallings' control vector binds
usage restrictions to the key itself, because if "a master key is treated as a session key,
it may be possible for an unauthorized application to obtain plaintext of session keys
encrypted with that master key" (§7.3, pp. 217–218). Practically: never use one key for both
encryption and MAC, or across two trust domains.

Deep dive: `references/03-randomness-and-key-management.md`

## 3.7 Distributing trust: how does the verifier get the key? (Ch. 10.1, 14.2)

The algorithm is never the weak point here; the key's provenance is.

| Approach | Vulnerability |
|---|---|
| Public announcement (post the key) | "Anyone can forge such a public announcement … the forger is able to read all encrypted messages intended for A and can use the forged keys for authentication" (§10.1, p. 291) |
| Publicly available directory | Stronger, but compromise of the directory forges any identity |
| Public-key authority (online) | Requires the authority to be reachable for every exchange — a bottleneck and a target |
| **Public-key certificates** | Offline verification; shifts the problem to CA trust, validity, and **revocation** |

**Unauthenticated key agreement is the classic failure.** Diffie–Hellman gives you a shared
secret with *somebody*: Darth substitutes his own public values and ends up sharing one key
with each party, so "Alice and Bob think that they share a secret key, but instead Bob and
Darth share secret key K1 and Alice and Darth share secret key K2. All future communication
between Bob and Alice is compromised." The cause: "the protocol … does not authenticate the
participants" (§10.2, p. 300).

Certificate validation is therefore not optional, and it has four parts: chain signature,
identity/hostname, validity period, and **revocation** — a certificate can be revoked because
the private key "is assumed to be compromised," the user is no longer certified, or the CA's
own certificate is compromised (§14.2, p. 425).

Deep dives: `references/04-public-key-and-key-exchange.md`,
`references/07-identity-certificates-and-pki.md`

## 3.8 Where in the stack does the protection go? (Ch. 17.1, p. 530)

| Placement | Property | Trade-off |
|---|---|---|
| Network level (IPsec) | "Transparent to end users and applications"; general-purpose | Protects the path, not the parties; nothing survives the gateway |
| Transport level (TLS) | Protects a connection; may be application-aware | Ends at every terminator — a proxy sees plaintext |
| Application level | "Tailored to the specific needs of a given application"; can be end-to-end across intermediaries | You own the protocol, and therefore its bugs |

**Transport vs tunnel mode** is the same question about metadata: transport mode protects the
payload; tunnel mode encapsulates the entire packet, hiding the original addresses (§16.2,
p. 490). If who-talks-to-whom is sensitive, only the second helps — this is traffic-flow
confidentiality from §2.2.

Deep dive: `references/08-transport-and-channel-security.md`

## 3.9 Access control (Ch. 20.2, p. 634)

Authentication answers *who*; access control answers *what they may do*. Conflating them is
the most common authorization bug.

The **reference monitor** gives three requirements to check against any authorization design
(§20.2, p. 636):

| Requirement | Meaning | Review question |
|---|---|---|
| **Complete mediation** | "The security rules are enforced on every access, not just … when a file is opened" | Is there any path to the resource that skips the check? |
| **Isolation** | The monitor and its database "must be protected from unauthorized modification" | Can the subject edit its own permissions or logs? |
| **Verifiability** | The monitor must be provably correct | Is the logic small and centralized enough to audit? |

Complete mediation is the one that fails in practice: a gateway check bypassed by a direct
service call, a list endpoint filtered while the detail endpoint is not, an ownership check
missing on fetch-by-ID.

For classified data, also check both information-flow directions: **no read up** and **no
write down** (§20.2, p. 635). Write-down is how a Trojan horse exfiltrates (§20.2, p. 637),
and in ordinary applications it looks like sensitive data landing in logs, analytics, or an
error tracker.

Deep dives: `references/09-authentication-and-access-control.md`,
`references/12-perimeter-and-trusted-systems.md`

## 3.10 Passwords (Ch. 18.3, p. 582)

Store only a one-way transformation, with a **per-user random salt**, using a **deliberately
slow** function.

The salt has three distinct purposes (§18.3, p. 582) — an application-wide constant salt
delivers none of them:

1. "It prevents duplicate passwords from being visible in the password file."
2. "It effectively increases the length of the password without requiring the user to
   remember two additional characters."
3. "It prevents the use of a hardware implementation of DES, which would ease the difficulty
   of a brute-force guessing attack."

Slowness is the other half: the UNIX scheme's 25 iterations exist because "the encryption
routine is designed to discourage guessing attacks" — and Stallings immediately warns the
margin decays, since "hardware performance continues to increase, so that any software
algorithm executes more quickly" (§18.3, p. 583). Cost parameters are therefore a maintained
value, not a constant.

Assume the store leaks: relying on access control alone fails because systems "are
susceptible to unanticipated break-ins," "an accident of protection might render the password
file readable," and users reuse passwords across machines (§18.3, p. 585). Measured
outcome without strength enforcement: **24.2%** of ~14,000 real passwords recovered by
dictionary attack, and "even a single hit may be enough to gain a wide range of privileges"
(§18.3, pp. 585–586). Stallings' ranked answer is a **proactive password checker** at
selection time (§18.3, p. 587).

Deep dive: `references/09-authentication-and-access-control.md`

## 3.11 Detection and audit (Ch. 18.1–18.2)

"Inevitably, the best intrusion prevention system will fail. A system's second line of
defense is intrusion detection" (§18.2, p. 570). Design as if prevention fails, because §2.2
says active attacks cannot be prevented absolutely.

- **Audit records are the input**, and they need Denning's fields: subject, action, object,
  exception-condition, resource-usage, timestamp (§18.2, p. 572).
- **Protect the trail from its subject.** The **clandestine user** is defined as one who
  "seizes supervisory control … to evade auditing and access controls or to suppress audit
  collection" (§18.1, p. 567).
- **Know your base rate.** Intruder and legitimate profiles overlap, so loosening a rule
  raises false positives and tightening it raises false negatives — "there is an element of
  compromise and art" (§18.2, p. 571). With rare events, even an accurate detector produces
  mostly false alarms (the base-rate fallacy, Appendix 18A, p. 593), and an alert stream
  nobody trusts is not detection.

Deep dive: `references/10-intrusion-detection-and-audit.md`

## 3.12 Untrusted input and availability (Ch. 19, 20.1)

- **Untrusted input reaching an interpreter or a buffer** is the direct route to compromise:
  "Intruders can get access to a system by exploiting attacks such as buffer overflows on a
  program that runs with certain privileges. Privilege escalation can be done this way as
  well" (§18.1, p. 570).
- **Trap doors are a code-review problem by definition** — "a secret entry point into a
  program that allows someone who is aware of the trap door to gain access without going
  through the usual security access procedures," and they "have been used legitimately for
  many years by programmers to debug and test programs" (§19.1, p. 599). Debug bypasses are
  backdoors that happen to have been added on purpose.
- **The perimeter is not an authorization mechanism.** A firewall "cannot protect against
  attacks that bypass the firewall" and "does not protect against internal threats" (§20.1,
  pp. 624–625), and IP-based trust is defeated by **IP address spoofing** (§20.1, p. 628).
- **Make the attacker pay first.** Against clogging attacks, Oakley's cookie exchange
  requires a cheap address-bound round trip before any expensive modular exponentiation
  (§16.6, pp. 507–508). Generalize it: never commit memory or CPU to an unauthenticated peer
  before a cheap proof.

Deep dives: `references/11-malicious-software-and-availability.md`,
`references/12-perimeter-and-trusted-systems.md`

---

# 4. VULNERABILITY GATE (MANDATORY)

Before any security-relevant design or diff is accepted, run the named-vulnerability
catalog against it: `references/vulnerability-catalog.md`.

Each entry has a **detection signature** (what the code or design looks like), the
**consequence** (what the attacker gains), and the **required fix**. The catalog is the
source of truth for `code-review`'s `/review-security` mode and for `verify`'s security
checks.

Report format:

```md
### Vulnerability Scan
| ID | Vulnerability | Present | Evidence | Required Fix |
|---|---|---|---|---|
| V-09 | Non-cryptographic PRNG for a security value | Yes | `auth/token.py:31` builds the reset token with `random.choices` | Generate with `secrets.token_urlsafe(32)` |
| V-32 | Certificate verification disabled | Yes | `client/http.go:88` sets `InsecureSkipVerify: true` | Remove; use the system pool, or pin the internal CA |
| V-25 | No replay protection | No | Webhook handler dedupes on `Stripe-Signature` timestamp + event ID | — |
```

Only list vulnerabilities that are actually applicable. An honest "not applicable — this
change touches no secrets, no identity, and no untrusted input" is a valid and preferred
result. Inventing findings to look thorough is the failure mode here.

---

# 5. SECURITY RECORD OUTPUT

When invoked for a design decision, produce this record. It is short on purpose: every line
is a commitment that `verify` and `code-review` can later check.

```md
## Security Record: [Component]

### Assets and Adversary
- Assets: [what is worth attacking here — data, credentials, money, availability]
- Adversary: [unauthenticated internet / authenticated tenant / insider / compromised dependency]
- Adversary capability: [can observe? modify? replay? impersonate? run code?]
- Explicitly out of scope: [what we are NOT defending against, and why that is acceptable]

### Services Claimed
| Service | Claimed? | Mechanism | Enforced where |
|---|---|---|---|
| Confidentiality | | | |
| Data origin authentication | | | |
| Data integrity (incl. freshness) | | | |
| Access control | | | |
| Nonrepudiation | | | |
| Availability | | | |

> A service claimed with no mechanism is a wish. A mechanism with no enforcement point is
> a suggestion.

### Attack Surface
| Entry point | Authenticated? | Authorized by | Input validated where | Rate limited |
|---|---|---|---|---|

### Cryptographic Decisions
| Decision | Choice | Rationale | Rejected alternative & why |
|---|---|---|---|
| Primitive & mode | | | |
| Key length | | | |
| Integrity mechanism | | | |
| Randomness source | | | |
| Freshness mechanism | | | |

### Key and Credential Lifecycle
- Generation: [source of randomness, where it happens]
- Storage: [at rest, and what protects it]
- Distribution: [how the verifier obtains it, and how that channel is authenticated]
- Purpose separation: [one key, one purpose — list them]
- Rotation: [trigger, procedure, and how old ciphertext stays readable]
- Revocation / compromise response: [the runbook]

### Detection
- Security events emitted: [subject, action, object, outcome]
- Where they go: [sink, retention, who can modify them]
- Alerts: [condition, expected rate, owner]

### Failure Behavior
| Dependency | If unavailable | Fails open or closed? |
|---|---|---|

### Residual Risk
[What remains exploitable, why it is accepted, and who accepted it]

### Vulnerability Scan
[Table from Section 4]
```

---

# 6. LANGUAGE DISCIPLINE (ANTI-HAND-WAVING)

These phrases are **forbidden** in a security design or review without an accompanying
mechanism:

| Forbidden | Must be replaced with |
|---|---|
| "it's encrypted" | Algorithm, mode, key length, where the key lives, and **what provides integrity** |
| "it's hashed" | Which algorithm, salted how, how slow, and whether collision resistance is required |
| "we use HTTPS" | Which hops, whether certificates are verified, and what the terminator sees in plaintext |
| "we validate input" | Allowlist or denylist, at which boundary, and what the parser does with the rest |
| "it's behind the firewall / on the internal network" | The authorization check that runs regardless of origin (§20.1, p. 624) |
| "only our app calls this" | The authentication that enforces it — otherwise anyone who can reach it is "our app" |
| "the token is random" | Generator, entropy in bits, and whether the generator passes the next-bit test |
| "we use JWTs" | Algorithm pinned, signature verified, issuer/audience/expiry checked, revocation story |
| "the signature proves it's valid" | Signature verifies origin; state separately what provides **freshness** (§1.4, p. 18) |
| "we're using a nonce" | Generated how, checked against what, remembered for how long |
| "it's a secure random UUID" | Which version — v4 from a CSPRNG, or v1 from a timestamp and MAC address |
| "the library handles it" | Which call, which defaults, and what happens on the error path |
| "we log everything" | Which security events, with which fields, and who can delete them |
| "we'll rate limit it" | Per what key, at what threshold, and the behavior at the limit |
| "we sanitize it" | The exact transformation and the context it is safe for (SQL ≠ HTML ≠ shell) |
| "internal tool, low risk" | The adversary you are excluding, and why they cannot reach it |

---

# 7. REFERENCE INDEX

| Reference | Stallings source | Read when |
|---|---|---|
| `references/01-threat-model-and-security-services.md` | Ch. 1, p. 5 | Framing a threat model; deciding which service you owe |
| `references/02-cryptographic-primitives-and-modes.md` | Ch. 2, 3, 5, 6 | Choosing a cipher, mode, IV/nonce strategy, key length |
| `references/03-randomness-and-key-management.md` | Ch. 7, p. 198 | Tokens, salts, nonces, key hierarchy, rotation, separation |
| `references/04-public-key-and-key-exchange.md` | Ch. 9–10, p. 256 | RSA/ECC use, key agreement, key distribution, timing attacks |
| `references/05-integrity-hashes-and-macs.md` | Ch. 11–12, p. 316 | MACs, HMAC, hash properties, birthday bound, tag length |
| `references/06-signatures-and-authentication-protocols.md` | Ch. 13, p. 376 | Signatures, nonrepudiation, replay, nonces, timestamps |
| `references/07-identity-certificates-and-pki.md` | Ch. 14, p. 399 | Tickets, sessions, certificates, chain validation, revocation |
| `references/08-transport-and-channel-security.md` | Ch. 16–17, p. 483 | TLS config, layering, anti-replay windows, tunnel vs transport |
| `references/09-authentication-and-access-control.md` | Ch. 18.3, 20.2 | Passwords, credential storage, authorization, privilege |
| `references/10-intrusion-detection-and-audit.md` | Ch. 18.1–18.2, 18A | Audit records, security events, alert thresholds, base rate |
| `references/11-malicious-software-and-availability.md` | Ch. 19, p. 598 | Untrusted input, backdoors, supply chain, DoS, amplification |
| `references/12-perimeter-and-trusted-systems.md` | Ch. 20, p. 621 | Network controls, reference monitor, information flow, assurance |
| `references/vulnerability-catalog.md` | Cross-cutting | Every review and verification |

---

# 8. PROPORTIONALITY RULE (ANTI-OVER-ENGINEERING)

Rigor must scale with what an attacker gains. Demanding a threat model for a CSS change is
itself a failure mode, and it trains people to ignore the gate.

| Change class | Required output |
|---|---|
| No secrets, no identity, no untrusted input, no new surface | Nothing from this skill |
| New input parsed, or existing authorized endpoint extended | Vulnerability scan of the applicable sections only |
| New endpoint, new session/token handling, new outbound call | Vulnerability scan + Attack Surface table |
| Any cryptography, credential storage, or key handling | Full Security Record |
| New authentication or authorization model, or a new trust boundary | Full Security Record + explicit adversary and residual-risk sign-off |
| Money, PII, PHI, credentials, or a path to code execution | Full Security Record + Detection section + a named human approver |

**Simplicity is a security property.** Stallings' own account of why this field is hard is
that "successful attacks are designed by looking at the problem in a completely different
way, therefore exploiting an unexpected weakness in the mechanism" (§1, p. 9) — every extra
mechanism is another surface for that. The reference monitor's third requirement is
**verifiability** (§20.2, p. 636): a design too complex to audit is not secure, whatever it
contains. If a boring, standard, centralized mechanism meets the requirement, that *is* the
correct answer, and the Security Record should say so.

---

**Attribution:** concepts, terminology, and page references throughout this skill and its
references are drawn from *Cryptography and Network Security*, 4th edition, by William
Stallings (Prentice Hall, 2005). Page numbers refer to that edition. Guidance marked
**[Modern]** is not from that edition and reflects current practice.
