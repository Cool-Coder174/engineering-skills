# Digital Signatures and Authentication Protocols

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 13 (p. 376).

Chapter 13 is the most directly applicable chapter in the book for application security review,
because it is about *freshness* — the property that almost every homegrown token, webhook, and
callback scheme gets wrong.

---

## 1. What a signature adds over a MAC (§13.1, pp. 378–379)

> "Message authentication protects two parties who exchange messages from any third party. However,
> **it does not protect the two parties against each other**." (p. 378)

Two disputes follow (p. 379):

1. "Mary may forge a different message and claim that it came from John. Mary would simply have to
   create a message and append an authentication code using the key that John and Mary share."
2. "John can deny sending the message. Because it is possible for Mary to forge a message, there is no
   way to prove that John did in fact send the message."

With concrete stakes: "An electronic funds transfer takes place, and the receiver increases the amount
of funds transferred and claims that the larger amount had arrived from the sender," and "an
electronic mail message contains instructions to a stockbroker for a transaction that subsequently
turns out badly. The sender pretends that the message was never sent" (p. 379).

**Review trigger.** Whenever the two parties to a message have adverse interests — customer and
merchant, tenant and platform, client and auditor — a shared-key MAC is structurally inadequate.
Ask: could the verifier have produced this? If yes, it proves nothing in a dispute — and no amount
of key hygiene fixes it, because the mechanism cannot provide the service.

### Properties of a signature (§13.1, p. 379)

> "It must verify the author and the date and time of the signature. It must authenticate the contents
> at the time of the signature. It must be verifiable by third parties, to resolve disputes." (p. 379)

Note that **time is in the definition**, not an add-on. A signature over a payload with no timestamp
is only partially doing its job.

Requirements (p. 379): the signature depends on the message; it uses information unique to the sender
"to prevent both forgery and denial"; it is easy to produce and to verify; it is "computationally
infeasible to forge … either by constructing a new message for an existing digital signature or by
constructing a fraudulent digital signature for a given message"; and "It must be practical to retain
a copy of the digital signature in storage."

That last requirement is the one engineers skip. Nonrepudiation is worthless if you did not keep the
signature and the exact bytes that were signed. If your system verifies a signature and stores only
the parsed result, you cannot prove anything later — you have bought the mechanism and thrown away
the service.

---

## 2. Sign inside, encrypt outside (§13.1, p. 380)

> "Note that it is important to perform the **signature function first and then an outer
> confidentiality function**. In case of dispute, some third party must view the message and its
> signature. If the signature is calculated on an encrypted message, then the third party also needs
> access to the decryption key to read the original message. However, if the signature is the inner
> operation, then the recipient can store the plaintext message and its signature for later use in
> dispute resolution." (p. 380)

So the ordering rules differ by mechanism, and both are right:

- **MAC for integrity of a ciphertext:** encrypt-then-MAC (§2.2 of `05-integrity-hashes-and-macs.md`).
  The MAC is a transport-level check; you want to reject forgeries before decrypting.
- **Signature for nonrepudiation:** sign the plaintext, then encrypt. The signature is a durable
  artifact about *content*, and it must remain meaningful to a third party who never had the transport
  key.

An encrypt-then-sign design also lets an attacker strip your signature and apply their own to the same
ciphertext, claiming authorship of content they cannot read — which is nonsense that a verifier may
nonetheless accept.

---

## 3. The weakness of direct signatures (§13.1, p. 380)

> "All direct schemes described so far share a common weakness. **The validity of the scheme depends
> on the security of the sender's private key.** If a sender later wishes to deny sending a particular
> message, the sender can claim that the private key was lost or stolen and that someone else forged
> his or her signature." (p. 380)

Stallings' partial mitigation: "require every signed message to include a timestamp (date and time)
and to require prompt reporting of compromised keys to a central authority" (p. 380). But even that
leaves a hole: "some private key might actually be stolen from X at time T. The opponent can then send
a message signed with X's signature and stamped with a time before or equal to T" (p. 380).

**This is why revocation needs a time dimension.** Marking a key as revoked *now* does not tell you
whether a signature dated last Tuesday was legitimate. Systems that need durable nonrepudiation use a
trusted timestamp — an arbiter, in Stallings' terms (§13.1, p. 380: "Every signed message from a
sender X to a receiver Y goes first to an arbiter A, who subjects the message and its signature to a
number of tests to check its origin and content. The message is then dated and sent to Y") — or an
append-only log. In practice, this is the argument for recording signature verification events with
server-side timestamps at the moment of receipt.

---

## 4. Timeliness is a first-class requirement (§13.2, pp. 382–384)

> "Central to the problem of authenticated key exchange are two issues: **confidentiality and
> timeliness**… The second issue, timeliness, is important because of the threat of message replays.
> Such replays, at worst, could allow an opponent to compromise a session key or successfully
> impersonate another party. **At minimum, a successful replay can disrupt operations by presenting
> parties with messages that appear genuine but are not.**" (p. 382)

That last sentence is the answer to "so what if they replay it?" A replayed order, a replayed webhook,
a replayed refund, a replayed "increment counter" — all appear genuine, because they *are* genuine
messages, just not fresh ones.

### 4.1 The four kinds of replay (§13.2, p. 383)

From [GONG93]:

| Kind | Definition (p. 383) |
|---|---|
| **Simple replay** | "The opponent simply copies a message and replays it later" |
| **Repetition that can be logged** | "An opponent can replay a timestamped message within the valid time window" |
| **Repetition that cannot be detected** | "The original message could have been suppressed and thus did not arrive at its destination; only the replay message arrives" |
| **Backward replay without modification** | "A replay back to the message sender. This attack is possible if symmetric encryption is used and the sender cannot easily recognize the difference between messages sent and messages received on the basis of content" |

The second is the one that catches teams who think they solved replay: a timestamp with a five-minute
window means replay is legal for five minutes. If the operation is not idempotent, that is a live
vulnerability. The third means absence of a duplicate is not evidence of no attack. The fourth is why
protocols must distinguish direction — bind a role, a direction flag, or distinct keys per direction.

### 4.2 Three freshness mechanisms, and their real costs (§13.2, pp. 383–384)

**Sequence numbers.** "A new message is accepted only if its sequence number is in the proper order.
The difficulty with this approach is that it requires each party to keep track of the last sequence
number for each claimant it has dealt with. Because of this overhead, sequence numbers are generally
not used for authentication and key exchange" (p. 383).

**Timestamps.** "Party A accepts a message as fresh only if the message contains a timestamp that, in
A's judgment, is close enough to A's knowledge of current time. This approach requires that clocks
among the various participants be synchronized" (p. 383).

**Challenge/response.** "Party A, expecting a fresh message from B, first sends B a nonce (challenge)
and requires that the subsequent message (response) received from B contain the correct nonce value"
(p. 383).

Then the trade-off analysis, which is the most useful paragraph in the chapter for design review:

> "It can be argued … that the timestamp approach should not be used for connection-oriented
> applications because of the inherent difficulties with this technique. First, some sort of protocol is
> needed to maintain synchronization among the various processor clocks. This protocol must be both
> fault tolerant, to cope with network errors, and secure, to cope with hostile attacks. Second, the
> opportunity for a successful attack will arise if there is a temporary loss of synchronization
> resulting from a fault in the clock mechanism of one of the parties. Finally, because of the variable
> and unpredictable nature of network delays, distributed clocks cannot be expected to maintain precise
> synchronization. Therefore, **any timestamp-based procedure must allow for a window of time
> sufficiently large to accommodate network delays yet sufficiently small to minimize the opportunity
> for attack**." (pp. 383–384)

> "On the other hand, the challenge-response approach is unsuitable for a connectionless type of
> application because it requires the overhead of a handshake before any connectionless transmission,
> effectively negating the chief characteristic of a connectionless transaction." (p. 384)

**Decision rule:**

| Situation | Mechanism | Notes |
|---|---|---|
| Interactive, connection-oriented, both parties online | Challenge/response with a nonce | No clock dependency; costs a round trip |
| One-shot, connectionless, store-and-forward | Timestamp with a bounded window | Requires a clock policy; window is a security parameter |
| Ordered stream between two known parties | Sequence numbers | Requires per-peer state |
| Anything where the operation is not idempotent | **Timestamp *and* a recorded message ID** | Window-bounded replay is still replay |

The practical composition for an HTTP webhook or signed request is all three ideas: a timestamp with a
tight window (bounded exposure), a unique message ID stored until the window expires (kills
within-window replay), and both fields inside the signed payload (so neither can be edited). Store the
ID, don't just check it — and note that the ID store must be shared across replicas or the attacker
simply retries against a different instance.

---

## 5. Protocols are hard: the Needham–Schroeder lineage (§13.2, pp. 384–386)

Read this section not for the protocols but as evidence about the difficulty of the genre.

**Needham–Schroeder** (p. 384) distributes a session key via a KDC, with a nonce handshake in steps 4
and 5 whose purpose "is to prevent a certain type of replay attack" (p. 384).

It is still broken:

> "Despite the handshake of steps 4 and 5, the protocol is still vulnerable to a form of replay attack.
> Suppose that an opponent, X, has been able to compromise an old session key… X can impersonate A and
> trick B into using the old key by simply replaying step 3. **Unless B remembers indefinitely all
> previous session keys used with A, B will be unable to determine that this is a replay.** If X can
> intercept the handshake message, step 4, then it can impersonate A's response, step 5. From this point
> on, X can send bogus messages to B that appear to B to come from A using an authenticated session
> key." (p. 384)

**Denning's fix** (p. 385) adds a timestamp T so "both A and B know that the key distribution is a
fresh exchange," verified by `|Clock − T| < Δt₁ + Δt₂` where Δt₁ is the expected clock discrepancy and
Δt₂ the expected network delay (p. 385).

**Gong's attack on the fix** (p. 385) is the one to carry away:

> "A new concern is raised: namely, that this new scheme requires reliance on clocks that are
> synchronized throughout the network… The problem occurs when a sender's clock is ahead of the
> intended recipient's clock. In this case, **an opponent can intercept a message from the sender and
> replay it later when the timestamp in the message becomes current at the recipient's site**. This
> replay could cause unexpected results. Gong refers to such attacks as **suppress-replay attacks**."
> (p. 385)

With a footnote that has aged well: "Such things can and do happen. In recent years, flawed chips were
used in a number of computers … to track the time and date. The chips had a tendency to skip forward
one day" (p. 385).

Countermeasures (p. 385): "enforce the requirement that parties regularly check their clocks against
the KDC's clock," or "rely on handshaking protocols using nonces. This latter alternative is not
vulnerable to a suppress-replay attack because the nonces the recipient will choose in the future are
unpredictable to the sender."

And then the **Kehne–Langendörfer–Schoenwaelder** protocol (p. 385), which fixes both — and carries
Stallings' driest footnote, attached to the fact that it too had to be corrected by [NEUM93a]:

> "It really is hard to get these things right." (p. 385, footnote 3)

**That footnote is the justification for the entire "don't invent protocols" rule.** Four iterations
by named researchers over fifteen years, each fixing the last, each publishing a new flaw. Your
bespoke handshake, designed in an afternoon, is not better. When a diff introduces a custom
authentication or key-exchange flow, the finding is the invention itself.

One design detail from that protocol is worth stealing (p. 386): "the time specified in T_b is a time
relative to B's clock. Thus, **this timestamp does not require synchronized clocks because B checks
only self-generated timestamps**." If you must use expiry, have the verifier issue and check its own
timestamps rather than trusting the sender's clock. Server-issued nonces and server-set expiry times
sidestep the whole clock-synchronization problem.

Also note the reuse pattern (p. 386): after the initial exchange, subsequent sessions use the ticket
plus fresh nonces N′ₐ and N′_b, and "When B receives the message in step 1, it verifies that the ticket
has not expired." Cheap re-authentication, but expiry is still checked every time. Skipping the expiry
check on a cached credential is V-26.

---

## 6. One-way authentication: the store-and-forward case (§13.2, pp. 388–389)

Email is the model for anything asynchronous — queues, webhooks, batch files, offline clients. Two
requirements: the handling system must not need the plaintext ("the e-mail message should be encrypted
such that the mail-handling system is not in possession of the decryption key," p. 388), and "the
recipient wants some assurance that the message is from the alleged sender" (p. 388).

The KDC-based scheme must drop the handshake, "Because we wish to avoid requiring that the recipient
(B) be on line at the same time as the sender (A), steps 4 and 5 must be eliminated" (p. 388). The
consequence is stated plainly:

> "This approach guarantees that only the intended recipient of a message will be able to read it. It
> also provides a level of authentication that the sender is A. **As specified, the protocol does not
> protect against replays.** Some measure of defense could be provided by including a timestamp with
> the message. However, because of the potential delays in the e-mail process, such timestamps may
> have limited usefulness." (p. 388)

**Generalize.** Removing the round trip removes freshness. Any fire-and-forget authenticated message —
a webhook, a queued job, a signed URL, an offline license file — is replayable by default, and
timestamps are weak there precisely because the legitimate delay is large and variable. The defense
must be receiver-side: idempotency keys, exactly-once processing on a durable dedup store, or
consuming a server-issued single-use token.

The efficient hybrid for confidentiality is also given here (p. 389): `A → B: E(PUb, Ks) ‖ E(Ks, M)` —
"the message is encrypted with a one-time secret key. A also encrypts this one-time key with B's public
key… This scheme is more efficient than simply encrypting the entire message with B's public key."
That is the shape of every real encrypted-message format.

---

## 7. Modern additions

> **Modern.** Stallings predates JWT, OAuth, and SAML, but the failures in those systems are exactly
> the ones in this chapter, plus a few implementation-specific ones worth naming:
>
> - **Algorithm confusion.** The verifier must not take the algorithm from the token. `alg: none` and
>   RS256→HS256 substitution (verifying an HMAC using the public key as the secret) are the classic
>   forms. Pin the expected algorithm and key by policy. This is V-33 (verification key taken from
>   the message being verified) compounded by V-38 (downgrade permitted by negotiation).
> - **Missing claim validation.** `exp`, `nbf`, `iss`, `aud`, and `sub` are all §13.1 requirements in
>   disguise. An unvalidated `aud` means a token minted for service A is accepted by service B —
>   Stallings' masquerade, via a legitimately signed message.
> - **No revocation.** A stateless bearer token is valid until expiry no matter what. That is the
>   §13.1 stolen-private-key problem at token scale: decide the maximum acceptable window and keep a
>   denylist or short expiry with refresh.
> - **Signature verified after parsing.** Deserializing before verifying hands the attacker your parser
>   as an attack surface.
> - **Bearer tokens are replayable by definition.** Anything that reaches the verifier unchanged and
>   is accepted more than once is a §4.1 simple replay. Bind tokens to a channel (mTLS, DPoP), or
>   accept the risk explicitly.

---

## 8. Review checklist

- [ ] Does this need nonrepudiation (signature) or only authenticity (MAC)? Justify the choice against
      §1.
- [ ] If nonrepudiation is claimed: are the exact signed bytes and the signature retained?
- [ ] Is the signature applied to the plaintext, with encryption outside it?
- [ ] Does the signed content include the time, the sender, the intended recipient/audience, and the
      purpose — not just the payload?
- [ ] What makes each accepted message fresh? Name the mechanism: nonce, timestamp window, or
      sequence number.
- [ ] If a timestamp window is used: how large, and is the protected operation idempotent within it?
- [ ] Is a message ID recorded and checked, in a store shared across all replicas?
- [ ] Whose clock is trusted, and what happens on skew? Could a suppress-replay succeed?
- [ ] Can a message be replayed *back to its sender* (backward replay)? Is direction bound in?
- [ ] Is the algorithm and key chosen by the verifier's policy, not by the message?
- [ ] Are `exp`, `nbf`, `iss`, `aud` checked on every use, including cached credentials and tickets?
- [ ] Is any part of this a custom handshake or key-exchange design? (If so, that is the finding.)
