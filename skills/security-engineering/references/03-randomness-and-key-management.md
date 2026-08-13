# Randomness and Key Management

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 7 (§7.3, p. 210;
§7.4, p. 218). Related: §10.1 (public key distribution) in
`04-public-key-and-key-exchange.md`.

Algorithms are the easy part. This file covers the two things that actually break: where the
random values come from, and how keys are created, scoped, delivered, and retired.

---

## 1. Randomness is not one property (§7.4, p. 219)

Random values are used for "nonces … for handshaking to prevent replay attacks," for "session key
generation," and for "generation of keys" (p. 219). Then the sentence that should govern every
review of a `random()` call:

> "These applications give rise to **two distinct and not necessarily compatible requirements** for
> a sequence of random numbers: randomness and unpredictability." (p. 219)

### 1.1 Randomness (statistical)

Two criteria (p. 219):

- **Uniform distribution:** "the frequency of occurrence of each of the numbers should be
  approximately the same."
- **Independence:** "No one value in the sequence can be inferred from the others."

With an important epistemological note: "there is no such test to 'prove' independence. Rather, a
number of tests can be applied to demonstrate if a sequence *does not* exhibit independence"
(p. 219). Passing statistical tests is evidence of nothing more than not having failed them.

### 1.2 Unpredictability (cryptographic)

> "In applications such as reciprocal authentication and session key generation, the requirement is
> not so much that the sequence of numbers be statistically random but that the successive members
> of the sequence are **unpredictable**… care must be taken that an opponent not be able to predict
> future elements of the sequence on the basis of earlier elements." (pp. 219–220)

**This distinction is the entire finding for V-09.** `Math.random()`, `rand()`, `java.util.Random`,
Python's `random` module, and any linear congruential generator satisfy §1.1 and fail §1.2. They are
built to look random to a statistician, not to resist an adversary. A password reset token from
`Math.random()` will pass every test of uniformity you can write and still be predictable.

The formal bar is the **next-bit test** (p. 226): a generator is a cryptographically secure
pseudorandom bit generator if "there is not a polynomial-time algorithm that, on input of the first
k bits of an output sequence, can predict the (k + 1)st bit with probability significantly greater
than 1/2."

### 1.3 Why LCGs fail, concretely (§7.4, pp. 222–223)

The linear congruential generator `X(n+1) = (aX(n) + c) mod m` is "By far, the most widely used
technique for pseudorandom number generation" (p. 221) — which is exactly why it turns up in
security-relevant code by default.

> "There is nothing random at all about the algorithm, apart from the choice of the initial value
> X₀. Once that value is chosen, the remaining numbers in the sequence follow deterministically.
> This has implications for cryptanalysis.
>
> If an opponent knows that the linear congruential algorithm is being used and if the parameters
> are known … then **once a single number is discovered, all subsequent numbers are known**. Even if
> the opponent knows only that a linear congruential algorithm is being used, knowledge of a small
> part of the sequence is sufficient to determine the parameters of the algorithm." (pp. 222–223)

Stallings then shows the algebra: observe X₀, X₁, X₂, X₃, solve three modular equations for a, c,
and m (p. 223).

**Translate to a real system.** Issue session IDs from an LCG, and an attacker who registers an
account (getting one legitimate ID) can compute every other user's ID, past and future. This is not
a theoretical weakening; it is total.

Note also the *fix Stallings mentions and you should not accept*: reseeding from the system clock
(p. 223), suggested by [BRIG79] to make the sequence nonreproducible. Clock-based seeding is
guessable — an attacker who knows roughly when a token was issued searches a small space. Same for
seeding with a PID, a request counter, or a user ID. **Seeding a weak PRNG better does not make it a
CSPRNG.**

### 1.4 What good generation looks like (§7.4, pp. 223–226)

Stallings' constructions all share one shape: run a real cipher under a protected key.

- **Cyclic encryption** (p. 223): encrypt a counter with a master key. "Because the master key is
  protected, it is not computationally feasible to deduce any of the session keys (random numbers)
  through knowledge of one or more earlier session keys."
- **OFB mode as a generator** (p. 224): "Successive 64-bit outputs constitute a sequence of
  pseudorandom numbers with good statistical properties… the use of a protected master key protects
  the generated session keys."
- **ANSI X9.17** (p. 224), "One of the strongest (cryptographically speaking) PRNGs," driven by two
  inputs — a date/time value and an internal seed — where "the seed produced by the generator … is
  distinct from the pseudorandom number produced by the generator," so "Even if a pseudorandom
  number Rᵢ were compromised, it would be impossible to deduce the Vᵢ₊₁ from the Rᵢ" (p. 225).
- **Blum Blum Shub** (p. 225), which "has perhaps the strongest public proof of its cryptographic
  strength," with security resting "on the difficulty of factoring n."

The X9.17 property is worth extracting as a design principle: **output and internal state must be
separated**, so that leaking an output does not leak the state and thereby every future output.
This is *backtracking resistance* / *forward secrecy* in a PRNG, and it is why you should never
build a generator by hashing a counter with a fixed secret and exposing the hash.

### 1.5 True randomness and skew (§7.4, p. 226)

"A true random number generator (TRNG) uses a nondeterministic source to produce randomness. Most
operate by measuring unpredictable natural processes" — ionizing radiation, gas discharge tubes,
thermal noise across undriven resistors, disk read timing variation (p. 226).

Two practical cautions:

- **Published random tables are not secret.** "Although the numbers in these books do indeed exhibit
  statistical randomness, they are predictable, because an opponent who knows that the book is in
  use can obtain a copy" (p. 226). The modern equivalent is a committed test fixture, a seed checked
  into the repo, or a "random" value baked into a container image.
- **Skew.** "A true random number generator may produce an output that is biased in some way, such
  as having more ones than zeros… One approach to deskew is to pass the bit stream through a hash
  function" (p. 226).

> **Modern.** You do not build any of this. Use the OS CSPRNG, which mixes hardware entropy sources
> and is already deskewed: `/dev/urandom`, `getrandom(2)`, `BCryptGenRandom`,
> `crypto.randomBytes`, `secrets` in Python, `SecureRandom` in Java, `crypto/rand` in Go,
> `window.crypto.getRandomValues`. The review rule is a naming rule: if the call site does not have
> "secure," "crypto," or "secrets" in it, and the value is security-relevant, it is a finding.
>
> Two modern traps Stallings predates: (a) a forked process or a cloned VM/container image
> inheriting PRNG state, producing identical "random" values across instances; (b) early-boot
> entropy starvation on embedded or freshly provisioned hosts, where keys get generated before the
> pool is seeded. Both produce duplicate keys in the field. `getrandom(2)` without `GRND_NONBLOCK`
> and modern kernels handle this; hand-rolled seeding does not.

### 1.6 Sizing random values

Not in Stallings, but the necessary companion to §1.2:

| Value | Minimum | Reasoning |
|---|---|---|
| Session ID, bearer token, reset token, invite code | 128 bits (~22 chars base64url) | Must resist online guessing across the whole population |
| Password salt | 128 bits, unique per credential | Uniqueness matters more than size; see V-49 |
| CBC IV | One block, unpredictable | §5.2 of `02-cryptographic-primitives-and-modes.md` |
| GCM/AEAD nonce | 96 bits random, or a guaranteed-unique counter | Reuse is catastrophic |
| Symmetric key | ≥ 128 bits from a CSPRNG or KDF | §3 of `02-…-modes.md` |

Short tokens are a real vulnerability at scale: a 6-digit code is a million possibilities, which is
fine only if throttling and expiry are enforced (V-53).

---

## 2. Key distribution (§7.3, p. 210)

> "For symmetric encryption to work, the two parties to an exchange must share the same key, and
> that key must be protected from access by others. Furthermore, **frequent key changes are usually
> desirable to limit the amount of data compromised if an attacker learns the key**. Therefore, the
> strength of any cryptographic system rests with the key distribution technique." (p. 210)

The four options (p. 210):

1. A selects a key and physically delivers it to B.
2. A third party selects and physically delivers to both.
3. If A and B already share a key, one sends the new key encrypted under the old.
4. If A and B each have an encrypted link to a third party C, C delivers a key over those links.

The verdict on option 3 is the one that matters for code review:

> "Option 3 is a possibility for either link encryption or end-to-end encryption, but **if an
> attacker ever succeeds in gaining access to one key, then all subsequent keys will be revealed**."
> (p. 211)

Any scheme that derives each new key from the previous one — a rotation that encrypts the new key
under the outgoing key, a chain of derived credentials — has this property. Compromise once,
compromise forever. Rotation must break the chain by going back to a root of trust.

### 2.1 Scale, and why hierarchy exists (§7.3, pp. 210–212)

Pairwise keys grow quadratically: "if there are N hosts, the number of required keys is
[N(N−1)]/2… A network using node-level encryption with 1000 nodes would conceivably need to
distribute as many as half a million keys. If that same network supported 10,000 applications, then
as many as 50 million keys may be required for application-level encryption" (pp. 210–211).

The fix is a **key hierarchy** (p. 212):

> "Communication between end systems is encrypted using a temporary key, often referred to as a
> **session key**. Typically, the session key is used for the duration of a logical connection …
> and then discarded… session keys are transmitted in encrypted form, using a **master key** that
> is shared by the key distribution center and an end system or user." (p. 212)

> "If there are N entities that wish to communicate in pairs, then … as many as [N(N−1)]/2 session
> keys are needed at any one time. However, only N master keys are required, one for each entity.
> Thus, master keys can be distributed in some noncryptographic way, such as physical delivery."
> (p. 212)

**The architectural pattern to recognize.** A small number of long-lived, heavily protected keys,
never used to encrypt bulk data; a large number of short-lived working keys, derived or distributed
under them, discarded after use. That is a KMS root key wrapping per-tenant data keys, or a
long-lived credential exchanged for short-lived access tokens. A design where one long-lived key
directly encrypts everything has collapsed the hierarchy, and its blast radius is unbounded.

Two further properties of the hierarchy worth citing when arguing for regional or per-tenant
scoping: "A hierarchical scheme minimizes the effort involved in master key distribution, because
most master keys are those shared by a local KDC with its local entities. Furthermore, such a scheme
**limits the damage of a faulty or subverted KDC to its local area only**" (p. 214).

And the standing cost of centralization: "The use of a key distribution center imposes the
requirement that the KDC be trusted and be protected from subversion" (p. 216). A KDC — or a
KMS, or an auth service — is a universal single point of compromise. That is an acceptable trade
made deliberately, not a detail.

### 2.2 Key lifetime (§7.3, pp. 214–215)

> "The more frequently session keys are exchanged, the more secure they are, because the opponent
> has less ciphertext to work with for any given session key. On the other hand, the distribution of
> session keys delays the start of any exchange and places a burden on network capacity. A security
> manager must try to balance these competing considerations." (p. 214)

For connections: "use the same session key for the length of time that the connection is open,
using a new session key for each new session. If a logical connection has a very long lifetime,
then it would be prudent to change the session key periodically, perhaps every time the PDU
sequence number cycles" (p. 214).

For connectionless / transactional protocols, Stallings names the tension explicitly: "The most
secure approach is to use a new session key for each exchange. However, this negates one of the
principal benefits of connectionless protocols, which is minimum overhead and delay for each
transaction. **A better strategy is to use a given session key for a certain fixed period only or
for a certain number of transactions**" (p. 215).

Note the *or*: a key's life is bounded by **time and by volume**, whichever comes first. The
volume bound is the one modern designs forget, and it is precisely what governs AEAD nonce budgets
(§4 of `02-cryptographic-primitives-and-modes.md`). "We rotate annually" is half a policy.

### 2.3 Key separation (§7.3, pp. 217–218)

Beyond master versus session, define keys by *use*: "Data-encrypting key, for general communication
across a network; PIN-encrypting key, for personal identification numbers…; File-encrypting key,
for encrypting files stored in publicly accessible locations" (p. 217).

The failure this prevents is worth quoting in full, because it is the clearest statement in the book
of why key reuse across purposes is dangerous:

> "To illustrate the value of separating keys by type, consider the risk that a master key is
> imported as a data-encrypting key into a device. Normally, the master key is physically secured
> within the cryptographic hardware… Session keys encrypted with this master key are available to
> application programs, as are the data encrypted with such session keys. However, **if a master key
> is treated as a session key, it may be possible for an unauthorized application to obtain plaintext
> of session keys encrypted with that master key**." (p. 217)

A key used for two purposes lets an attacker use the weaker interface to attack the stronger one.
The application-level equivalents you will actually find in a diff:

- One HMAC key for session cookies and for signing webhook payloads: forge a session by getting the
  system to sign attacker-chosen webhook content.
- One AES key for user data and for internal config: an oracle in the config path decrypts user
  data.
- One RSA key pair used for both signing and encryption.
- A JWT signing key that is also the database encryption key.

The mechanisms Stallings gives for enforcing separation are the ancestors of modern key metadata.
The **tag** approach (p. 217) embeds usage bits in the key: session vs. master, may-encrypt,
may-decrypt. Its drawbacks are instructive — 8 bits is too few, and "because the tag is not
transmitted in clear form, it can be used only at the point of decryption, limiting the ways in
which key use can be controlled" (p. 218).

The better **control vector** (p. 218) attaches a variable-length vector of "fields that specify the
uses and restrictions for that session key," cryptographically bound to the key by hashing the
vector and XORing it into the wrapping key:

```
H = h(CV)
Ciphertext = E([Km ⊕ H], Ks)
```

So "The session key can be recovered only by using both the master key that the user shares with the
KDC **and** the control vector. Thus, the linkage between the session key and its control vector is
maintained" (p. 218). Its two advantages: unlimited length, and availability in the clear at every
stage so "control of key use can be exercised in multiple locations" (p. 218).

**This is exactly KMS key policy and JWK `use`/`key_ops`, and it is also exactly the argument for
binding context into a KDF.** `HKDF(master, info="webhook-signing-v2")` is a control vector: the
purpose string is bound to the derived key, so a key derived for one purpose is unusable for
another. When you review a design with one root secret and several uses, the fix is not "add more
secrets to the vault" — it is one root plus per-purpose derivation with the purpose in the `info`
parameter.

---

## 3. Nonces, and what they are for (§7.3, p. 212)

In the KDC scenario the nonce's job is stated plainly: the request carries "a unique identifier,
N₁, for this transaction, which we refer to as a nonce… The nonce may be a timestamp, a counter, or
a random number; the minimum requirement is that **it differs with each request**. Also, to prevent
masquerade, it should be difficult for an opponent to guess the nonce" (p. 212).

Two requirements, again distinct: **uniqueness** (else replay) and **unpredictability** (else
pre-computation and masquerade). A counter satisfies the first only. Note steps 4 and 5 of the
scenario — B sends a nonce N₂, A returns f(N₂) under the new session key — and Stallings' comment:
"These steps assure B that the original message it received (step 3) was not a replay… the actual
key distribution involves only steps 1 through 3 but … steps 4 and 5, as well as 3, perform an
authentication function" (p. 214). The freshness proof is a separate round trip, deliberately
added. See `06-signatures-and-authentication-protocols.md`.

---

## 4. Review checklist

**Random values**

- [ ] Is every security-relevant random value from a CSPRNG? (Grep the diff for `Math.random`,
      `rand()`, `mt_rand`, `java.util.Random`, `random.choice`, `uuid1`, `new Random(seed)`.)
- [ ] Is anything seeded from a clock, PID, counter, request ID, or user ID?
- [ ] Are tokens at least 128 bits? Are short codes compensated by expiry and throttling?
- [ ] Can PRNG state be duplicated by fork, container cloning, or a VM snapshot?
- [ ] Are UUIDs being used as unguessable capabilities? (v4 from a CSPRNG is acceptable; v1 and v7
      leak time and are not secrets.)

**Keys**

- [ ] Where does each key come from — a KMS, a KDF, an env var, or a literal in the repo?
- [ ] Is there a hierarchy, or does one long-lived key encrypt everything?
- [ ] Does each key have exactly one purpose? Is the purpose bound into the derivation or key
      policy?
- [ ] Is there a rotation path that does *not* derive the new key from the old one?
- [ ] Is there both a **time** bound and a **volume/transaction** bound on key use?
- [ ] Does the ciphertext or token carry a key identifier, so rotation and revocation are possible
      without a flag day?
- [ ] Is there a revocation story for a leaked key, and would anyone detect the leak?
- [ ] Are keys kept out of logs, error messages, crash dumps, and metrics labels?
- [ ] Are keys zeroed or scoped tightly in memory where the language allows it, and kept out of
      long-lived globals that end up in heap dumps?
