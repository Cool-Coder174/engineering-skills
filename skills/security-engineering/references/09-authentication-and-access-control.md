# Authentication and Access Control

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., §18.1 (Intruders, p. 567), §18.3
(Password Management, p. 582), and §20.2 (Trusted Systems, p. 635).

Password storage is the one topic in this book where the 1970s answer and the 2020s answer are the same
shape — salt, slow, protect the file anyway — and where Stallings' numbers still land. Access control
is the other half: §20.2 gives the vocabulary (subject, object, access right) and the three ways to
represent a policy, which is the vocabulary a review needs to ask whether an authorization decision
exists at all.

The reference monitor and its complete-mediation property live in
`12-perimeter-and-trusted-systems.md` §4; this file is about credentials and the authorization
decision itself.

---

## 1. Why the credential store is a separate asset (§18.1, pp. 568–569)

Stallings starts from the obvious statement and then makes it interesting:

> "Typically, a system must maintain a file that associates a password with each authorized user. **If
> such a file is stored with no protection, then it is an easy matter to gain access to it and learn
> passwords.**" (§18.1, p. 568)

Two countermeasures, and note that they are *two*, not one:

> "**One-way function:** The system stores only the value of a function based on the user's password."
> (§18.1, p. 568)

> "**Access control:** Access to the password file is limited to one or a very few accounts." (§18.1,
> p. 569)

> "**If one or both of these countermeasures are in place**, some effort is needed for a potential
> intruder to learn passwords." (§18.1, p. 569)

The threat model for the file, stated as a privilege-escalation stepping stone:

> "For example, if an intruder can gain access with a low level of privileges to an encrypted password
> file, then the strategy would be to capture that file and then **use the encryption mechanism of that
> particular system at leisure** until a valid password that provided greater privileges was
> discovered." (§18.1, p. 569)

And the condition that makes offline attack so much worse than online:

> "**Guessing attacks are feasible, and indeed highly effective, when a large number of guesses can be
> attempted automatically and each guess verified, without the guessing process being detectable.**"
> (§18.1, p. 569)

**This is the sentence to keep.** It contains the whole distinction between the two attacks and
therefore the whole reason for two different defences. Online guessing is *detectable* and *rate-
limitable*, so throttling and monitoring work. Offline guessing against a stolen hash is neither, so
the only defence is the cost of each individual guess — which is why the hash function's slowness is
the entire control. A design that relies on rate limiting to protect against a leaked hash has
confused the two.

Stallings also gives the concrete escalation path, which is a nice reminder that access control on the
file is not a strong boundary:

> "A low-privilege user produced a game program and invited the system operator to use it in his or her
> spare time. The program did indeed play a game, but in the background it also contained code to
> **copy the password file, which was unencrypted but access protected**, into the user's file."
> (§18.1, p. 569)

Access control was in place. The file still leaked, because the attacker borrowed a privileged
process's authority rather than attacking the control directly. See
`12-perimeter-and-trusted-systems.md` §5 for the full Trojan horse treatment; the lesson here is that
**"the credential store is access-controlled" is not a substitute for making the stored form useless to
whoever gets it** (V-51).

---

## 2. What the salt is actually for (§18.3, p. 582)

Three purposes, all stated explicitly. Most engineers can name one.

> "**It prevents duplicate passwords from being visible in the password file.** Even if two users choose
> the same password, those passwords will be assigned at different times. Hence, the 'extended'
> passwords of the two users will differ." (§18.3, p. 582)

> "**It effectively increases the length of the password** without requiring the user to remember two
> additional characters. Hence, the number of possible passwords is increased by a factor of 4096,
> increasing the difficulty of guessing a password." (§18.3, p. 582)

> "**It prevents the use of a hardware implementation of DES**, which would ease the difficulty of a
> brute-force guessing attack." (§18.3, p. 582)

Purpose 1 is the one that matters most and is least understood. Without a per-user salt, identical
hashes reveal identical passwords — so a leaked file immediately tells the attacker which accounts
share a password, and cracking one cracks all of them. Purpose 2 is why precomputation (rainbow
tables) fails: the attacker cannot build one table that covers all users, because each user's search
space is distinct. Purpose 3 generalises to "deny the attacker a fast, off-the-shelf implementation."

**A single application-wide salt (a "pepper" used alone) satisfies none of purpose 1 and none of
purpose 2.** It is V-49. The salt must be per-credential, generated fresh, and stored alongside the
hash — it is not a secret, and treating it as one adds nothing.

Purpose 3 leads directly into the deliberate-slowness argument:

> "**The encryption routine is designed to discourage guessing attacks.** Software implementations of
> DES are slow compared to hardware versions, and the use of **25 iterations multiplies the time
> required by 25.**" (§18.3, p. 583)

Iteration count as an explicit, tunable cost parameter — in 1979. That is the entire idea behind
bcrypt, scrypt, PBKDF2, and Argon2, and it is why a general-purpose hash is the wrong tool: SHA-256 is
fast by design, and fast is exactly the property you do not want here. Using it for password
verification is V-50; see `05-integrity-hashes-and-macs.md` §4 for why the same property that makes a
hash good for integrity makes it catastrophic for credentials.

> **Modern.** Use Argon2id, scrypt, or bcrypt, with cost parameters tuned to your hardware and
> reviewed periodically — Stallings' "25 iterations" was calibrated to 1979 hardware and has been
> obsolete for decades, which is the point: **the cost parameter is a moving target and needs an owner.**
> Password *storage* aside, prefer not to hold passwords at all: delegate to an identity provider, or
> use passkeys/WebAuthn, where there is no shared secret to steal. And keep the two-defence structure
> from §1: slow hash for the offline case, throttling and monitoring for the online case.

---

## 3. Protecting the file is necessary and insufficient (§18.3, pp. 584–585)

The two threats, named:

> "Thus, there are **two threats** to the UNIX password scheme. First, a user can gain access on a
> machine using a guest account or by some other means and then **run a password guessing program,
> called a password cracker, on that machine.**" (§18.3, p. 584)

> "In addition, if an opponent is able to obtain a copy of the password file, then **a cracker program
> can be run on another machine at leisure.** This enables the opponent to run through many thousands
> of possible passwords in a reasonable period." (§18.3, p. 584)

The obvious defence, and its flaws:

> "One way to thwart a password attack is to deny the opponent access to the password file. If the
> encrypted password portion of the file is accessible only by a privileged user, then the opponent
> cannot read it without already knowing the password of a privileged user." (§18.3, p. 585)

> "**An accident of protection might render the password file readable**, thus compromising all the
> accounts." (§18.3, p. 585)

**Design for the leak.** The stored form must remain expensive to attack after it is exfiltrated,
because "an accident of protection" is the normal case: a misconfigured bucket, a debug endpoint, a
backup, a log line, a replica with weaker controls, a SQL injection. Note also the second-order
finding: **backups and replicas of the credential store inherit its sensitivity and rarely inherit its
controls.** If the primary is locked down and the nightly dump is not, the control is decorative.

The cost of an offline attack, with numbers:

> "Using the fastest Thinking Machines implementation listed earlier, the time to encrypt all these
> words for all possible salt values is **under an hour.** Keep in mind that such a thorough search
> could produce a **success rate of about 25%**, whereas **even a single hit may be enough to gain a
> wide range of privileges on a system.**" (§18.3, p. 585)

That last clause is the risk calculus. You do not need to defend every account; the attacker needs one.
Any argument of the form "most of our users have decent passwords" is answering the wrong question.

---

## 4. Users pick guessable passwords, measurably (§18.1, p. 569; §18.3, pp. 584–587)

The intruder's playbook, in order of cost (§18.1, p. 569):

> "**Try default passwords used with standard accounts that are shipped with the system. Many
> administrators do not bother to change these defaults.**"

> "Exhaustively try all short passwords (those of one to three characters)."

> "Try words in the system's online dictionary or a list of likely passwords."

The first line is V-54 and it is still, decades later, how large fleets of devices get taken over.
Shipped-default credentials, seeded admin accounts in a migration, a fixture password that survives
into staging and then production — all the same finding. The measured outcome of the other two:

> "Almost 3% of the passwords were three characters or fewer in length." (§18.3, p. 584)

> "In all, **nearly one-fourth of the passwords were guessed.**" (§18.3, p. 585)

Four strategies for doing something about it (§18.3, pp. 586–587): "User education",
"Computer-generated passwords", "Reactive password checking", "Proactive password checking". Stallings
rates them, and the ratings are still right:

> "**This user education strategy is unlikely to succeed at most installations**, particularly where
> there is a large user population or a lot of turnover." (§18.3, p. 587)

> "A **reactive** password checking strategy is one in which the system periodically runs its own
> password cracker to find guessable passwords. The system cancels any passwords that are guessed and
> notifies the user." (§18.3, p. 587)

> "Because a determined opponent who is able to steal a password file can devote full CPU time to the
> task for hours or even days, **an effective reactive password checker is at a distinct disadvantage.**
> Furthermore, **any existing passwords remain vulnerable until the reactive password checker finds
> them.**" (§18.3, p. 587)

**The structural argument for checking at the point of choice.** Reactive checking races an attacker
who has more compute and no deadline, and leaves a window of exposure that lasts until the scan runs.
Proactive checking — reject the weak password when it is set — closes the window entirely. That is the
general principle: **validate at the boundary where you can still refuse, not afterwards where you can
only apologise.** Absent strength enforcement is V-52.

Two subtleties worth carrying:

- Even the good mechanisms are probabilistic. "Both the Markov model and the Bloom filter involve the
  use of probabilistic techniques. In the case of the Markov model, there is a small probability that
  some passwords in the dictionary will not be caught and a small probability that some passwords not
  in the dictionary will be rejected" (§18.3, p. 590). A strength checker has a false-negative rate; it
  is a mitigation, not a guarantee.
- A rejection message leaks. "This scheme alerts crackers as to which passwords not to try but may
  still make it possible to do password cracking" (§18.3, p. 587). Telling the user precisely why a
  password was rejected tells an attacker how to shape a guess list. Prefer a policy statement over a
  per-guess oracle.

> **Modern.** NIST SP 800-63B ended the composition-rules era: the effective controls are a minimum
> length, a check against known-breached and dictionary passwords, and **no** mandatory periodic
> rotation or composition rules (they push users toward predictable transformations). Check against a
> breach corpus at set time — that is proactive checking with a much better dictionary than 1990 had.

---

## 5. Online guessing: the control Stallings mentions in one line (§18.1, p. 569)

> "For example, a system can simply **reject any login after three password attempts**, thus requiring
> the intruder to reconnect to the host to try again. Under these circumstances, **it is not practical
> to try more than a handful of passwords.**" (§18.1, p. 569)

> "**However, the intruder is unlikely to try such crude methods.**" (§18.1, p. 569)

Two things in three sentences. Rate limiting is extremely effective against online guessing — it turns
an unbounded search into a handful of attempts. And it is not sufficient on its own, because the
attacker will simply choose the offline path instead. Both defences are required, aimed at different
attacks (§1).

Detection is the third leg, and §18.2 supplies the signal: "a large number of login attempts over a
short period suggests an attempted intrusion" (§18.2, p. 575), with the audit table entry "Password
failures at login — Operational — Attempted break-in by password guessing" (§18.2, p. 576). See
`10-intrusion-detection-and-audit.md` for what to record and why.

**The review questions.** Is there a limit on authentication attempts, and is it keyed on something the
attacker cannot trivially rotate (account plus source, not source alone)? Does the same limit protect
every credential-checking path — password login, token refresh, MFA code entry, password reset, and
any internal or legacy endpoint that also verifies credentials? Does lockout create a denial-of-service
against the legitimate user, and is that trade deliberate? Absent throttling is V-53; note especially
that a short numeric code (an MFA or reset code) has a small enough space that **throttling and expiry
are the only things standing between the attacker and the account** — see
`03-randomness-and-key-management.md` §1.6.

---

## 6. The vocabulary of an authorization decision (§20.2, pp. 635–636)

Three definitions that make it possible to ask precise questions:

> "**Subject:** An entity capable of accessing objects. Generally, the concept of subject equates with
> that of process." (§20.2, p. 635)

> "**Object:** Anything to which access is controlled. Examples include files, portions of files,
> programs, and segments of memory." (§20.2, p. 635)

> "**Access right:** The way in which an object is accessed by a subject. Examples are read, write, and
> execute." (§20.2, p. 635)

Note "the concept of subject equates with that of process." The subject is not the human — it is the
executing process acting on someone's behalf, which is exactly why confused-deputy problems exist: the
process has its own authority *and* a request from someone else, and it must not use the former to
satisfy the latter.

The access matrix decomposes two ways, and the two ways have different operational properties:

> "The matrix may be decomposed by **columns**, yielding **access control lists**. Thus, for each
> object, an access control list lists users and their permitted access rights." (§20.2, p. 635)

> "Decomposition by **rows** yields **capability tickets**. A capability ticket specifies authorized
> objects and operations for a user." (§20.2, p. 636)

**This distinction is worth holding in a review**, because most systems are a muddle of both and the
muddle is where the bugs live:

- An **ACL** is attached to the object, so the check happens at the resource and the answer is always
  current. Revocation is easy (edit one list); answering "what can this user reach?" is hard (scan
  every object).
- A **capability** travels with the subject, so the resource can decide by inspecting the token. That
  is a signed JWT with scopes, a pre-signed URL, an API key with embedded permissions. Answering "what
  can this subject do?" is easy; **revocation is hard**, because the authority is now a copy in
  somebody's hands. Every warning in `07-identity-certificates-and-pki.md` §2 and §5 about bearer
  tokens and revocation is a warning about capabilities.

The practical finding is a **mixed model with no clear authority**: a capability token carrying scopes
that the resource server trusts without also checking the object's own ACL, so a permission revoked at
the resource remains effective until the token expires. Decide which representation is authoritative
for each decision, and if it is the capability, then lifetime and revocation are load-bearing, not
details.

The scoping question follows: a capability that names broad object classes rather than specific objects
grants everything in the class. A token scoped `files:read` for all files, where the request needs one
file, is V-45 — and it is the form excess privilege usually takes in practice, because the coarse scope
is the convenient one to issue.

---

## 7. Where the authorization check has to be (§20.2, p. 637)

The reference monitor's first property is the one that matters most for application review:

> "**Complete mediation:** The security rules are enforced on **every access**, not just, for example,
> when a file is opened." (§20.2, p. 637)

Two failures follow directly, and both are extremely common:

**The check happens once and the result is trusted thereafter.** Authorize at the start of a session,
a batch, or a multi-step flow, then perform many operations under that one decision. Anything that
changes in between — a revoked role, a transferred resource, a modified parameter on step three — is
not seen. Stallings' "not just when a file is opened" is precisely this.

**The check is not on the enforcement path.** The UI hides the button, the gateway filters the route,
the list endpoint returns only your rows — but the handler that mutates the object does not verify
ownership, so a direct request with someone else's identifier succeeds. This is V-44, and it is the
defect behind most real-world broken-object-level-authorization findings. The test is mechanical: for
each new endpoint, consumer, job, or admin action, **can it be reached without passing through the
thing that makes the decision?** If yes, the decision is advisory.

The complete absence of a check is V-43 and needs no argument. The one worth stating is that a check
that exists but is derived from attacker-supplied data — a tenant id from the request body, a role
claim from an unverified header, a user id from a client-set cookie — is the same as no check. See
`07-identity-certificates-and-pki.md` §1 on option 2 versus option 3, and V-46 on trusting network
position.

**Fail closed.** §14.2's criticality rule is the model: an unrecognised critical extension means the
certificate "must be treated as invalid" (p. 427). If the authorization service is unreachable, the
policy is unparseable, or the resource's owner cannot be determined, the answer is deny — and if
availability makes that unacceptable, the degraded behaviour must be a stated decision rather than an
exception handler that returns `true`. That undefined-degraded-mode gap is V-64.

---

## 8. Privilege, and the argument for keeping it small (§20.1, p. 632)

The bastion host design characteristics are the cleanest statement of least privilege in the book, and
they read as a checklist for a service:

> "Each proxy module is a very small software package specifically designed for network security.
> **Because of its relative simplicity, it is easier to check such modules for security flaws.**"
> (§20.1, p. 632)

> "**Each proxy is independent of other proxies** on the bastion host. If there is a problem with the
> operation of any proxy, or if a future vulnerability is discovered, **it can be uninstalled without
> affecting the operation of the other proxy applications.**" (§20.1, p. 632)

> "A proxy generally **performs no disk access** other than to read its initial configuration file.
> **This makes it difficult for an intruder to install Trojan horse sniffers or other dangerous files**
> on the bastion host." (§20.1, p. 632)

> "**Each proxy runs as a nonprivileged user** in a private and secured directory on the bastion host."
> (§20.1, p. 632)

> "**Only the services that the network administrator considers essential are installed** on the
> bastion host." (§20.1, p. 631)

Five properties: small, independent, no write access it does not need, unprivileged, and minimal
surface. Each has a stated *reason*, which is what makes it a principle rather than a rule — and each
maps onto a container or service review question. Does this service run as root? Does it have a
writable filesystem it does not need? Does it hold a credential that grants more than its own job? Can
a compromise of it be contained, or does it share a database user, a service account, or a network
namespace with everything else? Broad-by-default privilege is V-45.

The corresponding limitation, from the same chapter, applies just as well to a service boundary as to
a firewall: "The firewall **does not protect against internal threats**" (§20.1, p. 624). A boundary
constrains what crosses it and says nothing about what is already inside.

---

## 9. Review checklist

**Credential storage**

- [ ] Are passwords stored only as the output of a deliberately slow, salted password hash — never
      encrypted, never reversible? (V-48)
- [ ] Is the salt per-credential and freshly generated, not a single application-wide value? (V-49)
- [ ] Is a password-specific KDF used (Argon2id, scrypt, bcrypt, PBKDF2) rather than a fast general
      hash? (V-50)
- [ ] Who owns the cost parameter, and when was it last reviewed against current hardware?
- [ ] Does the design assume the credential store will eventually leak — including backups, replicas,
      dumps, and logs? (V-51)
- [ ] Is verification done with a constant-time comparison? (V-21)

**Authentication paths**

- [ ] Is there a limit on authentication attempts, keyed so it cannot be trivially rotated around?
      (V-53)
- [ ] Does that limit cover *every* credential-verifying path — login, refresh, MFA entry, reset,
      legacy and internal endpoints?
- [ ] Are short numeric codes (MFA, reset, invite) protected by both throttling and a short expiry,
      given how small their space is?
- [ ] Are failed authentication attempts recorded as security events? (V-55)
- [ ] Is password strength checked at the point the password is set, including against a breach corpus?
      (V-52)
- [ ] Does the rejection message avoid acting as a per-guess oracle?
- [ ] Are there any shipped, seeded, fixture, or default credentials — in code, migrations, images, or
      docs? (V-54)
- [ ] Do error messages and response timings avoid revealing whether an account exists?

**Authorization decisions**

- [ ] For every new endpoint, consumer, scheduled job, and admin action: is there an authorization
      check at all? (V-43)
- [ ] Is the check on the enforcement path, so the operation cannot be reached without it — not only in
      the UI, the gateway, or the list query? (V-44)
- [ ] Is the check re-evaluated on every access, rather than once per session, batch, or flow? (V-44)
- [ ] Does the decision use only trusted inputs — never a tenant id, role, or user id taken from
      attacker-controlled request data? (V-46)
- [ ] Is the authoritative representation clear: does the resource decide from its own ACL, or does it
      trust a capability the caller presents? If the latter, is lifetime short and revocation real?
- [ ] Are capabilities and scopes narrowed to the specific objects and operations needed? (V-45)
- [ ] Does the service run with the least privilege it can: unprivileged user, no unnecessary write
      access, credentials scoped to its own job? (V-45)
- [ ] Can a compromise of this component be contained, or does it share credentials and reach with
      everything else?
- [ ] When the authorization dependency fails or the policy cannot be evaluated, does it fail closed —
      and if it must not, is the degraded behaviour a stated decision? (V-64)
