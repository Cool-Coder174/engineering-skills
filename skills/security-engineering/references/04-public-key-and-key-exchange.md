# Public-Key Cryptography and Key Exchange

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 9 (p. 256) and
Chapter 10 (p. 289).

Public-key crypto is where "we used a library so it's fine" fails most often, because the library
gives you a primitive and the security lives in how you distribute keys and bind identities.

---

## 1. Two misconceptions to kill first (§9.1, pp. 258–259)

> "One such misconception is that public-key encryption is **more secure** from cryptanalysis than is
> symmetric encryption. In fact, the security of any encryption scheme depends on the length of the
> key and the computational work involved in breaking a cipher. There is nothing in principle about
> either symmetric or public-key encryption that makes one superior to another from the point of view
> of resisting cryptanalysis." (p. 258)

> "A second misconception is that public-key encryption is a general-purpose technique that has made
> symmetric encryption **obsolete**. On the contrary, because of the computational overhead of current
> public-key encryption schemes, there seems no foreseeable likelihood that symmetric encryption will
> be abandoned." (p. 259)

In review terms: a design that RSA-encrypts a large payload is wrong on performance *and* usually on
security (see §4 below). The correct shape is hybrid — public key to establish or wrap a symmetric
key, symmetric AEAD for the data. Every real protocol (TLS, PGP, S/MIME, age, JWE) does this.

---

## 2. Pick the algorithm by the job it can do (§9.1, pp. 265–266)

Three distinct uses: "Encryption/decryption: The sender encrypts a message with the recipient's
public key. Digital signature: The sender 'signs' a message with its private key… Key exchange: Two
sides cooperate to exchange a session key" (pp. 265–266).

And not every algorithm does every job (Table 9.2, p. 266):

| Algorithm | Encryption | Signature | Key exchange |
|---|---|---|---|
| RSA | Yes | Yes | Yes |
| Elliptic curve | Yes | Yes | Yes |
| Diffie-Hellman | No | No | Yes |
| DSS | No | Yes | No |

**The review use is to catch category errors.** "We'll use Diffie-Hellman to authenticate the
client" is not a thing — DH establishes a shared secret and says nothing about who you shared it
with (§5). "We'll sign with our RSA encryption key" reuses one key for two purposes, which §2.3 of
`03-randomness-and-key-management.md` rules out.

> **Modern.** Prefer Ed25519 for signatures and X25519 for key agreement; ECDSA P-256 where a
> standard demands it. RSA is fine but must be RSA-OAEP for encryption and RSA-PSS for signatures,
> at ≥ 2048 bits (3072 for long-lived keys). Note that Ed25519 signs and X25519 agrees, and they are
> not interchangeable despite both being Curve25519 — the same category discipline applies.

### 2.1 The requirements, and why so few algorithms exist (§9.1, pp. 266–267)

Diffie and Hellman's conditions (p. 266), abbreviated: key generation is easy; encryption with the
public key is easy; decryption with the private key is easy; deriving the private key from the
public key is infeasible; and recovering the message from the public key plus ciphertext is
infeasible. A sixth, "although useful, is not necessary for all public-key applications": the keys
can be applied in either order (p. 266).

> "These are formidable requirements, as evidenced by the fact that only a few algorithms (RSA,
> elliptic curve cryptography, Diffie-Hellman, DSS) have received widespread acceptance in the
> several decades since the concept of public-key cryptography was proposed." (p. 266)

That is the strongest possible argument against novelty here. Four algorithms in several decades of
concentrated academic attention. A scheme invented during a sprint is not a fifth.

Requirement 6 also deserves a flag. That signing is "encryption with the private key" is an RSA
textbook artifact, not a general truth — it does not hold for DSA, ECDSA, or Ed25519, and it is the
source of the persistent and wrong mental model that a signature is a decryptable ciphertext.

---

## 3. Signing does not hide, and hiding does not authenticate (§9.1, pp. 264–265)

On using the private key to sign: "it is important to emphasize that the encryption process just
described does **not** provide confidentiality… Even in the case of complete encryption … there is
no protection of confidentiality because any observer can decrypt the message by using the sender's
public key" (p. 264).

To get both, you need both operations: `Z = E(PUb, E(PRa, X))` (p. 265) — sign then encrypt to the
recipient. "The disadvantage of this approach is that the public-key algorithm, which is complex,
must be exercised four times rather than two in each communication" (p. 265).

**Two findings this catches.** A "signed" payload assumed to be confidential because it looks like
opaque base64 — signing is not hiding, and a signed JWT's claims are readable by anyone holding it.
And a design that encrypts to a recipient and treats successful decryption as proof of sender
identity: anyone can encrypt to a public key, so decryption proves nothing about origin. The second
is the public-key form of V-15 — a mechanism credited with a service it does not provide.

The sign/encrypt ordering also matters in practice: encrypt-then-sign lets an attacker strip the
signature and re-sign the same ciphertext, claiming authorship. Bind the identity inside the signed
plaintext.

---

## 4. Textbook RSA is broken; the padding is the cryptosystem (§9.2, pp. 279–280)

> "The basic RSA algorithm is vulnerable to a **chosen ciphertext attack** (CCA)… the adversary
> exploits properties of RSA and selects blocks of data that, when processed using the target's
> private key, yield information needed for cryptanalysis." (p. 279)

Stallings gives the malleability directly: submit `X = (C × 2ᵉ) mod n` for decryption and receive
`Y = (2M)ᵈ`, from which M follows (p. 279).

And then the crucial escalation, which is why "we add some random bytes" is not a fix:

> "A simple padding with a random value has been shown to be **insufficient** to provide the desired
> security. To counter such attacks RSA Security Inc. … recommends modifying the plaintext using a
> procedure known as **optimal asymmetric encryption padding (OAEP)**." (p. 279)

OAEP's structure (p. 279): hash optional parameters, generate a random seed, expand it through a
mask generating function, XOR to mask the data block, then mask the seed with the masked block.

**Review rules.** Any use of a "raw," "textbook," or `NoPadding` RSA primitive is a finding.
PKCS#1 v1.5 encryption is a finding too — it is the target of Bleichenbacher's padding oracle, and
the modern variants (ROBOT) are still found in production. Use OAEP for encryption and PSS for
signatures. Where the code decrypts RSA, check that decryption failures are indistinguishable to the
caller, because differentiated errors rebuild the oracle even under good padding.

Related, and worth noting because it looks like an optimization: the private exponent must not be
small (Wiener's attack, cited p. 276), and the same message encrypted to several recipients under a
small public exponent with no padding is recoverable. Both are handled by using the standard
constructions and not the primitive.

---

## 5. Diffie-Hellman gives you a secret, not a peer (§10.2, pp. 298–301)

The math gives both sides the same value from public exchanges; "an adversary only has the following
ingredients to work with: q, α, YA, and YB" (p. 299).

Then the whole point:

> "The protocol depicted in Figure 10.8 is insecure against a **man-in-the-middle attack**." (p. 301)

Darth generates two key pairs, intercepts Alice's public value and substitutes his own toward Bob,
and vice versa. "At this point, Bob and Alice think that they share a secret key, but instead Bob and
Darth share secret key K1 and Alice and Darth share secret key K2. All future communication between
Bob and Alice is compromised" — Darth decrypts, reads, and optionally substitutes M′ (p. 301).

> "The key exchange protocol is vulnerable to such an attack because **it does not authenticate the
> participants**. This vulnerability can be overcome with the use of digital signatures and
> public-key certificates." (p. 301)

The chapter summary states the same thing as a one-line rule: "The protocol is secure only if the
authenticity of the two participants can be established" (p. 289).

Also note the limit of the static-DH directory variant: with long-lasting private values in a central
directory, you get confidentiality and "a degree of authentication," but "the technique does not
protect against replay attacks" (p. 300).

**What this means in review.** An encrypted channel with no peer authentication is not secure; it is
secure against a passive attacker only, which is half the threat model (§2 of
`01-threat-model-and-security-services.md`). Concretely:

- TLS with certificate verification disabled (`verify=False`, `InsecureSkipVerify`,
  `rejectUnauthorized: false`, a trust-all `X509TrustManager`) is exactly this attack. It is V-32,
  and it is severity-critical no matter how internal the network is.
- Custom handshakes that exchange raw public keys with no signature, pinning, or certificate.
- Key agreement where the transcript is not bound into the derived key, allowing an attacker to
  substitute parameters.

---

## 6. Distributing public keys is the actual problem (§10.1, pp. 291–295)

Four schemes, in increasing strength, and the attack on each. This progression is the best short
argument in the book for why you cannot skip PKI.

**Public announcement** (p. 291). Broadcast your key; anyone can use it.

> "Although this approach is convenient, it has a major weakness. **Anyone can forge such a public
> announcement.** That is, some user could pretend to be user A and send a public key to another
> participant or broadcast such a public key. Until such time as user A discovers the forgery and
> alerts other participants, the forger is able to read all encrypted messages intended for A and can
> use the forged keys for authentication." (p. 291)

**Publicly available directory** (p. 291), maintained by a trusted entity. Better, but:

> "This scheme is clearly more secure than individual public announcements but still has
> vulnerabilities. **If an adversary succeeds in obtaining or computing the private key of the
> directory authority, the adversary could authoritatively pass out counterfeit public keys and
> subsequently impersonate any participant** and eavesdrop on messages sent to any participant.
> Another way to achieve the same end is for the adversary to **tamper with the records kept by the
> authority**." (p. 292)

**Public-key authority** (p. 292), with tighter control and a live request per lookup. Costs seven
messages; mitigated by caching, and "Periodically, a user should request fresh copies of the public
keys of its correspondents to ensure currency" (p. 293). The remaining problems: "The public-key
authority could be somewhat of a bottleneck … for a user must appeal to the authority for a public
key for every other user that it wishes to contact. As before, the directory of names and public keys
maintained by the authority is vulnerable to tampering" (p. 293).

**Public-key certificates** (p. 294), the answer:

> "A certificate consists of a public key plus an identifier of the key owner, with the whole block
> signed by a trusted third party." (p. 294)

With four requirements (pp. 294–295):

1. "Any participant can read a certificate to determine the name and public key of the certificate's
   owner."
2. "Any participant can verify that the certificate originated from the certificate authority and is
   not counterfeit."
3. "Only the certificate authority can create and update certificates."
4. (Denning) "Any participant can verify the **currency** of the certificate."

And the enrollment requirement that is always the weak link in practice: "Application must be in
person or by some form of secure authenticated communication" (p. 295).

**Map this onto your system.** Every service that consumes a public key, a JWKS document, a
verification key, or a webhook signing key is implementing one of these four schemes, whether or not
anyone said so:

| Stallings scheme | What it looks like in your code | Attack you have accepted |
|---|---|---|
| Public announcement | Key pasted into a config file or fetched from any URL | Anyone who can forge that key or that response impersonates the peer |
| Directory | JWKS endpoint fetched over HTTPS | Compromise or tamper the endpoint, impersonate everyone |
| Authority | Live introspection call per request | Availability and latency coupling; still trusts the authority |
| Certificates | Real certificate chain validation, pinning | Requires revocation and currency checking |

The finding to look for is a key fetched over an unauthenticated channel (V-33), and requirement 4
being skipped — expiry and revocation not checked. See `07-identity-certificates-and-pki.md`.

---

## 7. Side channels are real and were unexpected (§9.2, pp. 277–278)

> "If one needed yet another lesson about how difficult it is to assess the security of a
> cryptographic algorithm, the appearance of timing attacks provides a stunning one. Paul Kocher …
> demonstrated that a snooper can determine a private key by keeping track of how long a computer
> takes to decipher messages… This attack is alarming for two reasons: **It comes from a completely
> unexpected direction and it is a ciphertext-only attack**." (p. 277)

Stallings' analogy: "somewhat analogous to a burglar guessing the combination of a safe by observing
how long it takes for someone to turn the dial from number to number" (p. 277). And the attack "can
be adapted to work with any implementation that does not run in fixed time" (p. 277).

Countermeasures (p. 278):

- **Constant exponentiation time** — "a simple fix but does degrade performance."
- **Random delay** — with a warning: "if defenders don't add enough noise, attackers could still
  succeed by collecting additional measurements to compensate for the random delays." Treat adding
  jitter as insufficient on its own.
- **Blinding** — "Multiply the ciphertext by a random number before performing exponentiation. This
  process prevents the attacker from knowing what ciphertext bits are being processed." Cost: "a 2
  to 10% performance penalty" (p. 278).

**Application relevance.** You will not implement modular exponentiation, but the shape recurs
everywhere: any operation whose duration depends on secret data leaks that data. The forms that show
up in code review are non-constant-time comparison of MACs and tokens (V-21), early-return password
checks, cache-timing-visible table lookups in hand-written crypto, and — most commonly — an endpoint
that takes measurably longer for an existing account than a nonexistent one, which enumerates users
for free.

The meta-lesson is the one to internalize: security properties can be destroyed by facts outside the
algorithm's model. That is why "the math is correct" is not a review conclusion.

---

## 8. Key size, and where RSA stands (§9.2, §10.3)

RSA's security rests on factoring. Stallings notes the practical trade: "the larger the number of
bits in d, the better. However, because the calculations involved … are complex, the larger the size
of the key, the slower the system will run" (p. 275).

ECC's motivation (p. 301):

> "The key length for secure RSA use has increased over recent years, and this has put a heavier
> processing load on applications using RSA… The principal attraction of ECC, compared to RSA, is
> that it appears to offer equal security for a far smaller bit size, thereby reducing processing
> overhead." (pp. 301–302)

With an honest caveat for 2005 — "the confidence level in ECC is not yet as high as that in RSA"
(p. 302) — which has since been resolved in ECC's favor for new designs.

> **Modern.** Minimums for new work: RSA 2048 (3072 for keys living past ~2030), ECDSA/ECDH P-256,
> or Curve25519. Reject RSA-1024 and any 160-bit curve. Do not accept custom or unnamed curve
> parameters. Post-quantum migration (ML-KEM for key establishment, ML-DSA/SLH-DSA for signatures)
> matters now for anything where recorded traffic must stay confidential for a decade — the
> harvest-now-decrypt-later threat is a direct application of Stallings' own criterion that "the
> time required to break the cipher exceeds the useful lifetime of the information" (§2.2, p. 34).

---

## 9. Review checklist

- [ ] Is the public-key operation doing a job the algorithm actually supports?
- [ ] Is bulk data being encrypted directly with a public key instead of a hybrid scheme?
- [ ] Is RSA encryption using OAEP, and RSA signing using PSS? Any `NoPadding` or PKCS#1 v1.5?
- [ ] Are decryption and verification failures indistinguishable from the outside?
- [ ] For any key agreement: what authenticates the peer? Name the mechanism.
- [ ] Is certificate/hostname verification enabled everywhere, including internal traffic, health
      checks, test harnesses that leak into prod, and any custom trust manager?
- [ ] How does each verification key arrive, and over what authenticated channel?
- [ ] Is a fetched key set cached, and is the cache refreshed, bounded, and unable to be poisoned?
- [ ] Are expiry and revocation actually checked (requirement 4)?
- [ ] Is the same key pair used for more than one purpose?
- [ ] Is the signature computed over enough context — identity, audience, purpose, and time — or
      just over the payload?
- [ ] Are key sizes at or above current minimums, and is the algorithm identifier controlled by the
      verifier rather than the message?
