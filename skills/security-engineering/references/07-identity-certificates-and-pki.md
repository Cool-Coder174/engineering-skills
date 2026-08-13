# Identity, Tickets, Certificates, and PKI

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 14 (p. 399): §14.1
Kerberos, §14.2 X.509, §14.3 Public-Key Infrastructure.

Chapter 14 is where the abstract mechanisms become the systems you actually operate: tokens with
lifetimes, certificate chains, and revocation. It is also the best available critique of bearer
tokens, written twenty years before everyone shipped them.

---

## 1. How much do you trust the client? (§14.1, p. 402)

Three architectural options for a distributed client/server system:

1. "Rely on each individual client workstation to assure the identity of its user or users and rely on
   each server to enforce a security policy based on user identification (ID)."
2. "Require that client systems authenticate themselves to servers, but trust the client system
   concerning the identity of its user."
3. "Require the user to prove his or her identity for each service invoked. Also require that servers
   prove their identity to clients."

> "In a small, closed environment, in which all systems are owned and operated by a single
> organization, the first or perhaps the second strategy may suffice. But in a more open environment,
> in which network connections to other machines are supported, the third approach is needed to
> protect user information and resources housed at the server." (p. 402)

With a footnote that is the whole zero-trust argument in one line: "However, even a closed environment
faces the threat of attack by a disgruntled employee" (p. 402).

**This is the trust-boundary question for microservices, restated.** Option 2 — the service trusts an
`X-User-Id` header set by an upstream caller — is extremely common and is only defensible if the
network genuinely cannot be reached by an adversary and no service is ever compromised. Option 3, where
each service independently verifies a credential that proves the end user's identity, is what Stallings
says an open environment requires. A diff that adds a new internal endpoint reading identity from an
unverified header is choosing option 2, usually without knowing it (V-46).

Note requirement 3's second half: **servers prove their identity to clients too.** One-directional
authentication is a half-measure, and §14.1 says why: "an opponent could sabotage the configuration so
that messages to a server were directed to another location. The false server would then be in a
position to act as a real server and capture any information from the user and deny the true service to
the user" (p. 407).

### Kerberos' four requirements (§14.1, p. 403)

- **Secure:** "A network eavesdropper should not be able to obtain the necessary information to
  impersonate a user. More generally, Kerberos should be strong enough that a potential opponent does
  not find it to be the weak link."
- **Reliable:** "For all services that rely on Kerberos for access control, **lack of availability of
  the Kerberos service means lack of availability of the supported services.**"
- **Transparent:** "Ideally, the user should not be aware that authentication is taking place, beyond
  the requirement to enter a password."
- **Scalable:** "The system should be capable of supporting large numbers of clients and servers."

The reliability requirement is the one architects underweight: **your auth service is in the critical
path of everything.** Its availability target must be at least as high as the sum of what depends on
it, and an auth outage is a total outage. That is a security-relevant availability finding, not just
an SRE concern. And the trust concentration is explicit: "the authentication service is secure if the
Kerberos server itself is secure," with a footnote insisting "the security of the Kerberos server
should not automatically be assumed but must be guarded carefully" (p. 403).

---

## 2. Tickets, lifetimes, and the bearer-token problem (§14.1, pp. 406–408)

Kerberos is built up through a series of flawed scenarios, and each flaw is one you will find in a
real token system.

**Flaw 1: a reusable credential can be stolen and replayed.**

> "We do not wish an opponent to be able to capture the ticket and use it. Consider the following
> scenario: An opponent captures the login ticket and waits until the user has logged off his or her
> workstation. Then the opponent either gains access to that workstation or configures his workstation
> with the same network address as that of the victim. The opponent would be able to reuse the ticket
> to spoof the TGS. To counter this, the ticket includes a **timestamp** … and a **lifetime**,
> indicating the length of time for which the ticket is valid (e.g., eight hours)." (p. 406)

**Flaw 2: the lifetime is a direct trade, and neither end of it is safe.**

> "If this lifetime is very short (e.g., minutes), then the user will be repeatedly asked for a
> password. If the lifetime is long (e.g., hours), then an opponent has a greater opportunity for
> replay… This would give the opponent unlimited access to the resources and files available to the
> legitimate user.
>
> Similarly, if an opponent captures a service-granting ticket and uses it before it expires, the
> opponent has access to the corresponding service.
>
> Thus, we arrive at an additional requirement. **A network service … must be able to prove that the
> person using a ticket is the same person to whom that ticket was issued.**" (p. 407)

That requirement is exactly what a bearer token does not satisfy. Kerberos' answer is the
**authenticator**: alongside the reusable ticket, the client sends a freshly encrypted block under a
session key that only the legitimate client received.

> "Unlike the ticket, which is reusable, the authenticator is intended for use **only once** and has a
> very short lifetime… In effect, the ticket says, 'Anyone who uses K_c,tgs must be C.' … In effect, the
> authenticator says, 'At time TS₃, I hereby use K_c,tgs.' **Note that the ticket does not prove
> anyone's identity but is a way to distribute keys securely. It is the authenticator that proves the
> client's identity.** Because the authenticator can be used only once and has a short lifetime, the
> threat of an opponent stealing both the ticket and the authenticator for presentation later is
> countered." (p. 408)

**Read that against a modern JWT or session cookie.** A bearer token is a ticket with no
authenticator. Possession is the entire proof. Stallings tells you the missing piece: a
proof-of-possession step, performed fresh on each use, keyed to something only the real client holds.
Modern equivalents are mTLS-bound tokens, DPoP, and per-request signatures. If a design uses plain
bearer tokens — which is usually fine — the review question becomes: is the lifetime short enough, is
the token bound to anything, and is there a revocation path? Those three are the compensating controls
for having skipped the authenticator.

Also worth extracting: the ticket is "encrypted with a secret key known only to the AS and the TGS.
This prevents alteration of the ticket" (p. 406). A token whose contents the client can read *and
modify* is not a ticket. And a token whose signature the resource server does not verify — trusting
that it came through the gateway — is a ticket nobody checked.

### 2.1 Version 5 fixes worth knowing (§14.1, pp. 414–419)

The v4→v5 changes read like a list of design mistakes to avoid:

- **Ticket lifetime.** "Lifetime values in version 4 are encoded in an 8-bit quantity in units of five
  minutes. Thus, the maximum lifetime that can be expressed is 2⁸ × 5 = 1280 minutes" (p. 414).
  Version 5 uses "an explicit start time and end time, allowing tickets with arbitrary lifetimes"
  (p. 414). A fixed-width, fixed-unit expiry field is a design trap; use absolute timestamps.
- **Authentication forwarding and delegation.** Version 4 cannot forward credentials; version 5 adds
  FORWARDABLE, PROXIABLE, and MAY-POSTDATE flags (pp. 417–418). Delegation is a real requirement, and
  the right answer is an *explicit, constrained, flagged* delegation, not sharing the original
  credential. When a service passes a user's token downstream to act on their behalf, that is
  unconstrained delegation — the modern fix is token exchange with a narrowed audience and scope.
- The long-lifetime/short-lifetime tension is restated for v5: "When a ticket has a long lifetime,
  there is the potential for it to be stolen and used by an opponent for a considerable period. If a
  short lifetime is used to lessen the threat, then overhead is involved in acquiring new tickets"
  (p. 418). That is the refresh-token pattern: short-lived access credential, longer-lived and more
  carefully protected renewal credential.

---

## 3. What a certificate is, field by field (§14.2, pp. 420–422)

> "The heart of the X.509 scheme is the public-key certificate associated with each user. These user
> certificates are assumed to be created by some trusted certification authority (CA) and placed in
> the directory by the CA or by the user. **The directory server itself is not responsible for the
> creation of public keys or for the certification function; it merely provides an easily accessible
> location for users to obtain certificates.**" (p. 420)

That last sentence matters: the repository is untrusted infrastructure. The security comes from the
signature, not from the place you fetched it. Hence: "**Because certificates are unforgeable, they can
be placed in a directory without the need for the directory to make special efforts to protect them**"
(p. 422).

The fields (pp. 420–421), with the review-relevant ones emphasized:

| Field | Definition (abbreviated from pp. 420–421) | Why you care |
|---|---|---|
| Version | "Differentiates among successive versions of the certificate format" | v3 required for extensions |
| **Serial number** | "An integer value, unique within the issuing CA, that is unambiguously associated with this certificate" | This is what a CRL entry names |
| Signature algorithm identifier | "The algorithm used to sign the certificate" — and note Stallings' dig: "Because this information is repeated in the Signature field … this field has little, if any, utility" | Algorithm must come from policy, not from the artifact |
| **Issuer name** | "X.500 name of the CA that created and signed this certificate" | Chain building |
| **Period of validity** | "Consists of two dates: the first and last on which the certificate is valid" | Must be checked (V-32) |
| **Subject name** | "The name of the user to whom this certificate refers. That is, this certificate certifies the public key of the subject who holds the corresponding private key" | Must be matched against the expected identity |
| Subject's public-key information | "The public key of the subject, plus an identifier of the algorithm for which this key is to be used" | Key/algorithm binding |
| Extensions | v3 additions | Where `keyUsage` and `basicConstraints` live |
| **Signature** | "Covers all of the other fields of the certificate; it contains the hash code of the other fields, encrypted with the CA's private key" | The only reason to believe any of the above |

The two properties a CA-issued certificate gives you (p. 422):

> "Any user with access to the public key of the CA can verify the user public key that was certified.
> No party other than the certification authority can modify the certificate without this being
> detected."

And the bootstrapping problem that never goes away: "each participating user must have a copy of the
CA's own public key to verify signatures. **This public key must be provided to each user in an
absolutely secure (with respect to integrity and authenticity) way** so that the user has confidence
in the associated certificates" (p. 422).

**That is your trust store.** Every question about pinning, custom CAs, and "just add this cert to the
container image" is a question about how the root arrives. A root injected by an unauthenticated build
step, or a trust store that includes hundreds of CAs you have never heard of when you only ever talk
to your own services, are both real findings.

---

## 4. Chains, and why the chain is where bugs live (§14.2, pp. 422–423)

If A trusts X₁ and B is certified by X₂, then A cannot use B's certificate directly: "A can read B's
certificate, but A cannot verify the signature" (p. 422). The fix is a chain — A verifies X₂'s
certificate signed by X₁, then B's certificate signed by X₂ (p. 423):

```
X1<<X2>> X2<<B>>
```

Chains can be long: `Z<<Y>> Y<<V>> V<<W>> W<<X>> X<<A>>` (p. 423). And crucially, "B can obtain this set
of certificates from the directory, or **A can provide them as part of its initial message to B**"
(p. 423).

**The attacker supplies the chain.** That single fact is the source of most historical TLS
vulnerabilities. The verifier must independently enforce every constraint; nothing in the presented
chain can be taken as authoritative about itself. The extensions exist for exactly this:

- **Basic constraints:** "Indicates if the subject may act as a CA. If so, a certification path length
  constraint may be specified" (p. 428). Not enforcing this is the classic "any leaf certificate can
  sign for any domain" catastrophe.
- **Key usage:** "Indicates a restriction imposed as to the purposes for which, and the policies under
  which, the certified public key may be used. May indicate one or more of the following: digital
  signature, nonrepudiation, key encryption, data encryption, key agreement, CA signature verification
  on certificates, CA signature verification on CRLs" (p. 427). This is the control-vector idea from
  `03-randomness-and-key-management.md` §2.3, applied to public keys.
- **Criticality.** "Each extension consists of an extension identifier, a criticality indicator, and an
  extension value. The criticality indicator indicates whether an extension can be safely ignored. **If
  the indicator has a value of TRUE and an implementation does not recognize the extension, it must
  treat the certificate as invalid**" (p. 427). A fail-closed rule on unknown metadata — a good pattern
  to copy in your own token and policy formats.
- **Subject alternative name** (p. 428), which exists "for supporting certain applications, such as
  electronic mail, EDI, and IPSec, which may employ their own name forms." Modern hostname verification
  uses SAN, not CN.

Also note the separate lifetimes: "with digital signature keys, the usage period for the signing
private key is typically shorter than that for the verifying public key" (p. 427). You stop signing
long before you stop verifying — a distinction most homegrown key rotation schemes miss, which is why
rotation breaks old signatures.

**Never write chain validation yourself.** Use the platform verifier, and check that the code has not
disabled parts of it.

---

## 5. Revocation is the hard part (§14.2, pp. 423–424)

Three reasons to revoke early (p. 423):

1. "The user's private key is assumed to be compromised."
2. "The user is no longer certified by this CA."
3. "The CA's certificate is assumed to be compromised."

The mechanism:

> "Each CA must maintain a list consisting of all revoked but not expired certificates issued by that
> CA, including both those issued to users and to other CAs… Each certificate revocation list (CRL)
> posted to the directory is signed by the issuer and includes the issuer's name, the date the list was
> created, the date the next CRL is scheduled to be issued, and an entry for each revoked certificate.
> Each entry consists of the serial number of a certificate and revocation date for that certificate."
> (p. 424)

And then the sentence that describes essentially every real deployment's weak spot:

> "When a user receives a certificate in a message, **the user must determine whether the certificate
> has been revoked**. The user could check the directory each time a certificate is received. To avoid
> the delays (and possible costs) associated with directory searches, it is likely that the user would
> **maintain a local cache** of certificates and lists of revoked certificates." (p. 424)

**Caching revocation data is caching the absence of bad news.** The staleness window is the window in
which a compromised key still works. So the review questions are: is revocation checked at all; how
stale can the cached list be; and what happens when the check fails — fail open (available but
insecure) or fail closed (secure but brittle)? That decision must be explicit, and its default in most
libraries is fail open.

> **Modern.** CRLs largely gave way to OCSP, then to OCSP stapling, and increasingly to short-lived
> certificates that expire before revocation would matter — which is the same trade as Kerberos ticket
> lifetimes (§2). Browsers use CRLite/CRLSets. For your own tokens the analogue is identical: either
> keep a revocation list you actually consult, or make lifetimes short enough that you don't need one.
> Choosing neither is the finding (V-34). Certificate Transparency adds the detection layer — a
> misissued certificate is now discoverable, which is the §2 lesson of
> `01-threat-model-and-security-services.md` applied to PKI.

---

## 6. Authentication procedures: one, two, and three-way (§14.2, pp. 425–426)

All three "make use of public-key signatures. It is assumed that the two parties know each other's
public key" (p. 424).

**One-way** carries a timestamp, a nonce, and B's identity, signed by A. The nonce's role: the
receiver must "reject any new messages with the same nonce" until the timestamp expires (p. 425). Note
this is a nonce *and* a timestamp — the timestamp bounds how long you must remember nonces. That is the
practical design for a replay cache: store IDs only for the length of the acceptance window.

Also: "This information, sgnData, is included within the scope of the signature, guaranteeing its
authenticity and integrity. The message may also be used to convey a session key to B, encrypted with
B's public key" (p. 426). Everything that matters goes inside the signature.

**Two-way** adds "The identity of B and that the reply message was generated by B; That the message was
intended for A; The integrity and originality of the reply" (p. 426). Note "intended for A" — audience
binding, as a named requirement.

**Three-way** exists purely to remove the clock dependency:

> "In three-way authentication, a final message from A to B is included, which contains a signed copy of
> the nonce r_B. The intent of this design is that **timestamps need not be checked**: Because both
> nonces are echoed back by the other side, each side can check the returned nonce to detect replay
> attacks. This approach is needed when synchronized clocks are not available." (p. 426)

Same conclusion as §4.2 of `06-signatures-and-authentication-protocols.md`: an extra round trip buys
you freedom from clock synchronization. Pick which cost you'd rather pay, and say which one you picked.

---

## 7. PKIX roles (§14.3, pp. 428–429)

The components, and the separation of duties worth copying:

- **End entity:** "end users, devices (e.g., servers, routers), or any other entity that can be
  identified in the subject field of a public key certificate" (p. 428).
- **Certification authority (CA):** "The issuer of certificates and (usually) certificate revocation
  lists (CRLs)" (p. 429).
- **Registration authority (RA):** "An optional component that can assume a number of administrative
  functions from the CA. The RA is often associated with the End Entity registration process" (p. 429).
- **CRL issuer:** "An optional component that a CA can delegate to publish CRLs" (p. 429).
- **Repository:** "any method for storing certificates and CRLs so that they can be retrieved by End
  Entities" (p. 429).

**The RA/CA split is the important design lesson**: identity *vetting* is separated from key
*signing*. The signing key can then live in an HSM with a tiny attack surface, while the messy human
process of deciding who someone is happens elsewhere. Apply the same split to your own systems: the
service that mints credentials should do nothing but mint credentials, from a decision made and
recorded by something else.

---

## 8. Review checklist

**Tokens and sessions**

- [ ] Which of the three trust models (§1) does this service implement? Does it verify end-user
      identity itself, or trust an upstream header?
- [ ] Does the server authenticate itself to the client, not just the reverse?
- [ ] Is the credential a pure bearer token? If so, is the lifetime short, is it bound to a channel or
      key, and is there a revocation path?
- [ ] Are expiry and audience checked on every use, at every hop — including cached credentials?
- [ ] Is the token integrity-protected against modification by its holder, and is that verified by the
      resource server rather than only the gateway?
- [ ] Is delegation explicit and constrained (narrowed audience/scope), or is the user's token forwarded
      verbatim to downstream services?
- [ ] Is expiry an absolute timestamp with adequate range, not a fixed-width offset?
- [ ] Is the auth service's availability target consistent with everything that depends on it?

**Certificates**

- [ ] Is chain validation done by the platform verifier, with nothing disabled?
- [ ] Are hostname/SAN, validity period, `basicConstraints`, and `keyUsage` all enforced?
- [ ] Where does the trust anchor come from, and is that path integrity-protected?
- [ ] Is the trust store scoped to the CAs you actually need?
- [ ] Is revocation checked? How stale can the answer be, and does a failed check fail open or closed?
- [ ] Are signing and verification key lifetimes distinguished, so rotation doesn't invalidate history?
- [ ] Is anything in the presented chain trusted to describe its own permissions?
