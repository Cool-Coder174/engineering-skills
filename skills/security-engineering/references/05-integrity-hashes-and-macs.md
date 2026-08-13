# Integrity: Hashes and Message Authentication Codes

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 11 (p. 316) and
Chapter 12 (p. 348), especially §11.1, §11.3, §11.4, §12.3, §12.4.

Integrity failures are the most common serious crypto findings in application code, because
encryption feels like it should cover them and it does not.

---

## 1. The attack list that defines "message authentication" (§11.1, pp. 318–319)

Stallings enumerates eight network attacks and then partitions them by which mechanism handles
which. This partition is the most directly usable thing in the chapter.

| # | Attack | Definition (p. 318) |
|---|---|---|
| 1 | **Disclosure** | "Release of message contents to any person or process not possessing the appropriate cryptographic key" |
| 2 | **Traffic analysis** | "Discovery of the pattern of traffic between parties" |
| 3 | **Masquerade** | "Insertion of messages into the network from a fraudulent source… Also included are fraudulent acknowledgments of message receipt or nonreceipt" |
| 4 | **Content modification** | "Changes to the contents of a message, including insertion, deletion, transposition, and modification" |
| 5 | **Sequence modification** | "Any modification to a sequence of messages between parties, including insertion, deletion, and reordering" |
| 6 | **Timing modification** | "Delay or replay of messages… an entire session or sequence of messages could be a replay of some previous valid session, or individual messages in the sequence could be delayed or replayed" |
| 7 | **Source repudiation** | "Denial of transmission of message by source" |
| 8 | **Destination repudiation** | "Denial of receipt of message by destination" |

And the mapping (p. 319):

> "Measures to deal with the first two attacks are in the realm of **message confidentiality**…
> Measures to deal with items 3 through 6 … are generally regarded as **message authentication**.
> Mechanisms for dealing specifically with item 7 come under the heading of **digital signatures**…
> Dealing with item 8 may require a combination of the use of digital signatures and a protocol
> designed to counter this attack."

> "Message authentication is a procedure to verify that received messages come from the alleged
> source and have not been altered. Message authentication may also verify sequencing and
> timeliness." (p. 319)

**Use this list as a review instrument.** For any message-passing interface in the diff — a webhook,
a queue consumer, an RPC, a signed URL, an event stream — walk attacks 3 through 6 explicitly. Most
systems handle 3 and 4 with a MAC or a signature and silently accept 5 and 6. Note the word "may" in
"may also verify sequencing and timeliness": ordering and freshness are *additional* work, not
consequences of adding a MAC. A per-message MAC with no sequence number and no timestamp leaves
attacks 5 and 6 wide open, and that is V-25 and V-40.

---

## 2. Three ways to build an authenticator (§11.2, p. 320)

> "Message encryption: The ciphertext of the entire message serves as its authenticator.
> Message authentication code (MAC): A function of the message and a secret key that produces a
> fixed-length value that serves as the authenticator.
> Hash function: A function that maps a message of any length into a fixed-length hash value, which
> serves as the authenticator." (p. 320)

Note the two-level structure Stallings insists on: "there must be some sort of function that produces
an **authenticator**… This lower-level function is then used as a primitive in a higher-level
**authentication protocol** that enables a receiver to verify the authenticity of a message" (p. 320).

Almost all real findings are at the protocol level, not the primitive level. HMAC-SHA256 is
unbreakable; the protocol that computes it over the body but not the URL, or verifies it after
parsing, is broken.

### 2.1 Why encryption is a poor authenticator

Symmetric encryption "by itself can provide a measure of authentication" (p. 320) — but only if the
receiver can recognize valid plaintext, which requires structure the attacker cannot forge. In a
modern system where the plaintext is JSON, protobuf, or arbitrary bytes, that assumption fails.
Chapter 11 spends its early pages on exactly this: without some checksum or recognizable structure,
"any" ciphertext decrypts to something, and the receiver cannot tell.

Combined with the malleability results from `02-cryptographic-primitives-and-modes.md` (§5.2, §5.3),
the conclusion is: **decryption succeeding is not authentication**. Use an AEAD, or encrypt-then-MAC.

### 2.2 Ordering: MAC over plaintext or over ciphertext (§11.2, p. 326)

Stallings notes both arrangements and states a preference: "In the first case, the MAC is calculated
with the message as input and is then concatenated to the message. The entire block is then
encrypted. In the second case, the message is encrypted first. Then the MAC is calculated using the
resulting ciphertext… Typically, it is preferable to tie the authentication directly to the
plaintext" (p. 326).

> **Modern.** This is the one place where practice has moved decisively against the textbook. Prefer
> **encrypt-then-MAC** (MAC over the ciphertext, the IV, and any associated data), which is what
> every AEAD does internally. MAC-then-encrypt is what gave TLS its Lucky13 and padding-oracle
> problems, because the receiver must decrypt (and unpad) attacker-controlled data before it can
> check anything. Encrypt-then-MAC lets you reject forged ciphertext without touching the plaintext
> path at all. Read the textbook's preference as historical.

### 2.3 Why you want a MAC even when you have encryption (§11.2, pp. 327)

Six reasons, worth knowing because they are the arguments for integrity-without-confidentiality
designs:

1. Broadcast to many destinations, where "it is cheaper and more reliable to have only one
   destination responsible for monitoring authenticity" (p. 327).
2. A loaded receiver that "cannot afford the time to decrypt all incoming messages," checking
   selectively (p. 327).
3. Authenticating a program in plaintext, so "the computer program can be executed without having to
   decrypt it every time… if a message authentication code were attached to the program, it could be
   checked whenever assurance was required of the integrity of the program" (p. 327).
4. Applications where secrecy is unnecessary but authenticity is critical — SNMPv3 being the
   example, "particularly if the message contains a command to change parameters" (p. 327).
5. Architectural flexibility: "it may be desired to perform authentication at the application level
   but to provide confidentiality at a lower level, such as the transport layer" (p. 327).
6. Duration of protection: "**With message encryption, the protection is lost when the message is
   decrypted, so the message is protected against fraudulent modifications only in transit but not
   within the target system**" (p. 327).

Reason 3 is code signing and package integrity. Reason 5 is why "we have TLS" does not remove the
need for webhook signatures. Reason 6 is the argument for integrity tags on stored records, not just
on the wire.

### 2.4 A MAC is not a signature (§11.2, p. 327)

> "Note that the MAC does not provide a digital signature because **both sender and receiver share
> the same key**." (p. 327)

Anything the receiver can verify, the receiver could have produced. If the requirement is "prove
this came from them and they cannot deny it," you need Chapter 13. See
`06-signatures-and-authentication-protocols.md`.

---

## 3. MAC requirements, and how homemade MACs die (§11.3, pp. 331–333)

A MAC is `MAC = C(K, M)`, "where M is a variable-length message, K is a secret key shared only by
sender and receiver, and C(K, M) is the fixed-length authenticator" (p. 331).

Stallings first shows that brute-forcing the *key* is hard — because the MAC is many-to-one, an
attacker needs several message/MAC pairs to narrow the candidate keys (pp. 331–332). Then comes the
turn:

> "Thus, a brute-force attempt to discover the authentication key is no less effort and may be more
> effort than that required to discover a decryption key of the same length. **However, other attacks
> that do not require the discovery of the key are possible.**" (p. 332)

The worked example is the one to remember. Take a "MAC" defined as encrypt the XOR of all blocks:
`Δ(M) = X₁ ⊕ X₂ ⊕ … ⊕ Xₘ`, `C(K, M) = E(K, Δ(M))`. Brute-forcing the key takes 2⁵⁶ DES operations.
But an attacker can pick *any* desired blocks Y₁ … Yₘ₋₁ and set

```
Yₘ = Y₁ ⊕ Y₂ ⊕ … ⊕ Yₘ₋₁ ⊕ Δ(M)
```

> "The opponent can now concatenate the new message, which consists of Y₁ through Yₘ, with the
> original MAC to form a message that will be accepted as authentic by the receiver. With this
> tactic, **any message of length 64 × (m − 1) bits can be fraudulently inserted**." (p. 333)

Total forgery, key never discovered, cipher never broken. This is what "we hash the fields together
with a secret" looks like from the attacker's side.

The three formal requirements (pp. 333–334):

1. "If an opponent observes M and C(K, M), it should be computationally infeasible for the opponent
   to construct a message M′ such that C(K, M′) = C(K, M)."
2. "C(K, M) should be uniformly distributed in the sense that for randomly chosen messages M and M′,
   the probability that C(K, M) = C(K, M′) is 2⁻ⁿ."
3. "Let M′ be equal to some known transformation on M… In that case, Pr[C(K, M) = C(K, M′)] = 2⁻ⁿ."

Requirement 3 exists because "the authentication algorithm should not be weaker with respect to
certain parts or bits of the message than others. If this were not the case, then an opponent who had
M and C(K, M) could attempt variations on M at the known 'weak spots'" (p. 334).

**The review consequence is absolute: never construct a MAC.** Use HMAC or CMAC. Specifically, all
of these are findings — the first two are V-18, the third V-17, the fourth V-20:

- `sha256(secret + message)` — vulnerable to length extension on Merkle–Damgård hashes (SHA-1,
  SHA-256, MD5): the attacker appends data and computes a valid tag without the secret.
- `sha256(message + secret)` — collisions in the hash become forgeries.
- A CRC, checksum, or plain hash with no key at all — anyone can recompute it.
- Any "sign the concatenated fields" scheme where field boundaries are ambiguous, so `a="1|2", b="3"`
  and `a="1", b="2|3"` produce the same signing input. Canonicalize with explicit lengths or a
  structured encoding.

---

## 4. Hash function requirements (§11.4, pp. 334–335)

Six properties (adapted from [NECH92], p. 335):

1. "H can be applied to a block of data of any size."
2. "H produces a fixed-length output."
3. "H(x) is relatively easy to compute for any given x."
4. "For any given value h, it is computationally infeasible to find x such that H(x) = h. This is
   sometimes referred to … as the **one-way property**."
5. "For any given block x, it is computationally infeasible to find y ≠ x such that H(y) = H(x). This
   is sometimes referred to as **weak collision resistance**." (Modern name: second-preimage
   resistance.)
6. "It is computationally infeasible to find any pair (x, y) such that H(x) = H(y). This is sometimes
   referred to as **strong collision resistance**." (Modern name: collision resistance.)

Stallings even warns about the vocabulary: "Unfortunately, these terms are not used consistently…
The reader must take care in reading the literature to determine the meaning of the particular terms
used" (p. 335).

Which property you need depends on the use, and this is where design mistakes live:

| Use | Property required | Failure if absent |
|---|---|---|
| Hash a secret and send the hash | 4 (one-way) | Attacker inverts and recovers the secret |
| Verify a specific known document wasn't altered | 5 (second preimage) | Attacker finds a different document with the same digest |
| Signatures, certificates, content addressing, dedup | 6 (collision) | Attacker crafts two colliding documents and swaps them after signing |

Stallings' own illustration of property 4's necessity: with a keyed-hash scheme `C = H(S_AB‖M)`, "if
the hash function is not one way, an attacker can easily discover the secret value… Because the
attacker now has both M and S_AB‖M, it is a trivial matter to recover S_AB" (p. 335).

**Property 3 is a liability, not an asset, for passwords.** "H(x) is relatively easy to compute" is
exactly what you do not want when hashing credentials. SHA-256 is a correct hash and a catastrophic
password hash. See `09-authentication-and-access-control.md` and V-50.

---

## 5. The birthday attack, and why tag length matters (§11.4, pp. 338–340; App. 11A)

The naive estimate is wrong by a square root. For a 64-bit hash, finding a second message matching a
*given* digest takes about 2⁶³ tries (p. 338). But Yuval's attack does far better (p. 338):

1. A signs by appending an m-bit hash and encrypting it with A's private key.
2. "The opponent generates 2^(m/2) variations on the message, all of which convey essentially the
   same meaning," plus an equal number of variations on the fraudulent message.
3. Compare the two sets for a collision — "The probability of success, by the birthday paradox, is
   greater than 0.5."
4. "The opponent offers the valid variation to A for signature. This signature can then be attached
   to the fraudulent variation… Because the two variations have the same hash code, they will produce
   the same signature; **the opponent is assured of success even though the encryption key is not
   known**."

> "Thus, if a 64-bit hash code is used, the level of effort required is only on the order of 2³²."
> (p. 338)

And generating the variations is easy: "the opponent could insert a number of 'space-space-backspace'
character pairs between words throughout the document… Alternatively, the opponent could simply
reword the message but retain the meaning" (p. 338). Figure 11.8 shows one letter in 2³⁷ variations.

> "The conclusion to be drawn from this is that the length of the hash code should be substantial."
> (p. 338)

Also: "some form of birthday attack will succeed against any hash scheme involving the use of cipher
block chaining without a secret key, provided that either the resulting hash code is small enough
(e.g., 64 bits or less) or that a larger hash code can be decomposed into independent subcodes"
(p. 340).

**Practical rules.** Effective security is n/2 bits against collisions, so a 128-bit digest gives 64
bits of collision resistance — not enough for signatures. Use SHA-256 or better where collision
resistance matters. Never truncate a hash or a MAC below 128 bits without a specific analysis, and
never truncate because the field is short. MD5 and SHA-1 are collision-broken in practice and must
not be used for signatures, certificates, or content identity (V-07, V-23). The step-4 shape of the attack
— get the victim to sign something benign that collides with something malicious — is also the
argument for never signing attacker-supplied blobs you have not canonicalized.

---

## 6. HMAC (§12.3, pp. 368–371)

The problem: "A hash function such as SHA was not designed for use as a MAC and cannot be used
directly for that purpose because it does not rely on a secret key. There have been a number of
proposals for the incorporation of a secret key into an existing hash algorithm. The approach that
has received the most support is HMAC" (p. 368).

Construction (p. 369):

```
HMAC(K, M) = H[(K⁺ ⊕ opad) ‖ H[(K⁺ ⊕ ipad) ‖ M]]
```

Two different pads mean "we have pseudorandomly generated two keys from K" (p. 369) — the nested
structure is what defeats length extension.

Design objectives from RFC 2104 (p. 368), and why they matter to you:

- Use existing hash functions unmodified.
- **"To allow for easy replaceability of the embedded hash function in case faster or more secure hash
  functions are found or required."** Concretely: "if the security of the embedded hash function were
  compromised, the security of HMAC could be retained simply by replacing the embedded hash function
  with a more secure one" (p. 368). This is agility designed in, and it is why your tags should carry
  an algorithm identifier that *you* control.
- Preserve performance; handle keys simply.
- "To have a well understood cryptographic analysis of the strength of the authentication mechanism."

That last is the real argument: "The last design objective … is, in fact, the main advantage of HMAC
over other proposed hash-based schemes. HMAC can be proven secure provided that the embedded hash
function has some reasonable cryptographic strengths" (p. 369).

### 6.1 HMAC survives a broken hash (§12.3, pp. 371)

The security proof reduces HMAC forgery to one of two attacks on the compression function, both of
which must work "even with an IV that is random, secret, and unknown to the attacker" (p. 371). This
has a striking consequence:

> "Does this mean that a 128-bit hash function such as MD5 is unsuitable for HMAC? The answer is no…
> when attacking HMAC, the attacker **cannot generate message/code pairs off line because the attacker
> does not know K**. Therefore, the attacker must observe a sequence of messages generated by HMAC
> under the same key… For a hash code length of 128 bits, this requires 2⁶⁴ observed blocks (2⁷² bits)
> generated using the same key. On a 1-Gbps link, one would need to observe a continuous stream of
> messages with no change in key for about 150,000 years in order to succeed." (p. 371)

**Do not misread this.** It says HMAC-MD5 is not trivially forgeable, and it is a good illustration of
why keying changes an attack's economics. It does not license MD5 in new code — MD5 is disqualified
elsewhere (signatures, certificates, content identity), tooling and auditors reject it, and there is
no upside. Use HMAC-SHA-256. The transferable lesson is that a keyed construction is far more robust
than an unkeyed one against the same broken primitive.

---

## 7. CMAC, and the CBC-MAC trap (§12.4, p. 372)

CBC-MAC (the FIPS PUB 113 Data Authentication Algorithm) is CBC with a zero IV, keeping only the last
block (§11.3, p. 333). It is secure — with a restriction that is nearly always violated:

> "[BELL00] demonstrated that this MAC is secure under a reasonable set of security criteria, with the
> following restriction. **Only messages of one fixed length of mn bits are processed**, where n is the
> cipher block size and m is a fixed positive integer. As a simple example, notice that given the CBC
> MAC of a one-block message X, say T = MAC(K, X), the adversary immediately knows the CBC MAC for the
> two-block message X‖(X ⊕ T) — since this is once again T." (p. 372)

That is a one-line forgery for variable-length messages, requiring no key and no computation.

CMAC fixes it: Black and Rogaway's three-key construction, refined by Iwata and Kurosawa so "the two
n-bit keys could be derived from the encryption key, rather than being provided separately," adopted
by NIST SP 800-38B for AES and triple DES (p. 372).

**Review rule.** Raw CBC-MAC over variable-length input is a finding. If a codebase computes a tag
using AES-CBC and takes the last block, check for length handling; if it is absent, it is forgeable.
Use CMAC, or better, HMAC or an AEAD.

Note the pattern this shares with §3: both the XOR-MAC and CBC-MAC failures are *structural*
forgeries that require neither key recovery nor cipher weakness. Attackers do not break AES. They
break the way you glued AES to your message format.

---

## 8. Review checklist

- [ ] For every inbound message: which of attacks 3–6 (masquerade, content modification, sequence
      modification, timing/replay) is defended, and by what?
- [ ] Is any integrity claim resting on decryption succeeding? (It shouldn't.)
- [ ] Is the tag HMAC or CMAC or an AEAD — not a bare hash, keyed hash, CRC, or bespoke construction?
- [ ] Does the MAC cover *everything* that matters: body, IV/nonce, headers used in decisions, URL
      path, method, key id, algorithm, version, recipient, and expiry?
- [ ] Is the signing input canonical and unambiguous — no delimiter injection across concatenated
      fields?
- [ ] Is the tag verified **before** the payload is parsed, deserialized, or acted on?
- [ ] Is the comparison constant-time?
- [ ] Is the tag at least 128 bits, and untruncated?
- [ ] Is the hash algorithm appropriate for the property needed (one-way vs. second preimage vs.
      collision), and is MD5/SHA-1 absent from any collision-sensitive path?
- [ ] Is a slow, salted KDF used for passwords rather than a fast hash?
- [ ] Is the algorithm chosen by the verifier's policy rather than read from the message?
- [ ] Is there a key/algorithm identifier so the scheme can be rotated?
- [ ] For stored data: is integrity protected at rest, or only in transit?
