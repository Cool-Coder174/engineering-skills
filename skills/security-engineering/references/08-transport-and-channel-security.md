# Transport and Channel Security

**Source:** Stallings, *Cryptography and Network Security*, 4th ed., Chapter 16 (IP Security, p. 483)
and Chapter 17 (Web Security, p. 527).

These two chapters are the only place in the book where all the primitives get assembled into a
working channel, and they are the best available material on a question reviewers ask constantly and
usually answer by habit: **at which layer does this security service belong, and what does that choice
buy and cost?** Chapter 16 argues for the network layer, Chapter 17 lays the three options side by
side, and the SSL/TLS record and handshake protocols show what a correct channel actually contains.

---

## 1. The layering decision (§16.1, p. 484; §17.1, pp. 529–531)

Stallings opens Chapter 16 by naming the problem:

> "The Internet community has developed application-specific security mechanisms in a number of
> application areas, including electronic mail (S/MIME, PGP), client/server (Kerberos), Web access
> (Secure Sockets Layer), and others. However, **users have some security concerns that cut across
> protocol layers.**" (§16.1, p. 484)

That is the case for pushing the service down the stack:

> "By implementing security at the IP level, an organization can ensure secure networking not only for
> applications that have security mechanisms but also for the **many security-ignorant applications.**"
> (§16.1, p. 484)

Chapter 17 turns this into an explicit three-way comparison (§17.1, pp. 529–531). The framing is worth
quoting because it is exactly the framing a design review needs:

> "The various approaches that have been considered are similar in the services they provide and, to
> some extent, in the mechanisms that they use, but **they differ with respect to their scope of
> applicability and their relative location within the TCP/IP protocol stack.**" (§17.1, p. 529)

| Layer | Stallings' stated advantage | What you give up |
|---|---|---|
| **Network (IPsec)** | "transparent to end users and applications and provides a general-purpose solution. Further, IPSec includes a filtering capability so that only selected traffic need incur the overhead" (§17.1, p. 530) | The application cannot see or reason about identity; protection is per-host/per-network, not per-user or per-object |
| **Transport (TLS)** | "could be provided as part of the underlying protocol suite and therefore be transparent to applications. Alternatively, SSL can be embedded in specific packages" (§17.1, p. 530) | Protection ends at the socket; anything that terminates TLS sees plaintext |
| **Application** | "the service can be tailored to the specific needs of a given application" (§17.1, p. 531) | You are now implementing security code, per application, with all that implies |

The deployment benefits of the network-layer choice are stated concretely and they are the argument
for a service mesh, a VPN, or a TLS-terminating ingress, twenty years early:

> "When IPSec is implemented in a firewall or router, it provides strong security that can be applied
> to all traffic crossing the perimeter. **Traffic within a company or workgroup does not incur the
> overhead of security-related processing.**" (§16.1, p. 487)

> "IPSec is below the transport layer (TCP, UDP) and so is transparent to applications. There is no
> need to change software on a user or server system when IPSec is implemented in the firewall or
> router." (§16.1, p. 487)

> "IPSec can be transparent to end users. There is no need to train users on security mechanisms,
> issue keying material on a per-user basis, or revoke keying material when users leave the
> organization." (§16.1, p. 487)

**The review consequence.** Read that middle sentence — traffic inside the perimeter incurs no
security processing — as the assumption it is, not as a benefit. It is the perimeter model, and it is
the assumption that fails the moment one host inside the boundary is compromised. If a design places
its only channel protection at the edge, the finding is not "you chose the wrong layer"; it is that
**everything inside now trusts network position** (V-46), and every internal service must be assumed
reachable by an adversary who lands anywhere inside.

The corresponding failure in the other direction is choosing a layer that cannot express the property
you need. Transport-layer protection authenticates *a channel*; it says nothing about *which user*
sent a particular message, and it does not survive the message being stored, queued, or forwarded. If
the requirement is "this payload is attributable to this customer even after it lands in the queue,"
no amount of TLS provides it — that needs an object-level signature at the application layer. Naming
the layer wrongly is V-39.

Stallings is also honest about the cost of the low-layer choice: "**The IPSec specification has become
quite complex**" (§16.2, p. 487). Complexity at the channel layer is not free, and the more
configurable the channel, the more ways it can be configured insecurely — which §5 below is entirely
about.

---

## 2. What a channel actually has to provide (§16.2, p. 489; §17.2, p. 533)

IPsec's service list is the most complete enumeration in the book of what "secure channel" means:

> "The services are: Access control; Connectionless integrity; Data origin authentication; **Rejection
> of replayed packets (a form of partial sequence integrity)**; Confidentiality (encryption); Limited
> traffic flow confidentiality" (§16.2, p. 489)

Six services, of which only one is encryption. The SSL Record Protocol lists two:

> "**Confidentiality:** The Handshake Protocol defines a shared secret key that is used for
> conventional encryption of SSL payloads.
> **Message Integrity:** The Handshake Protocol also defines a shared secret key that is used to form a
> message authentication code (MAC)." (§17.2, p. 533)

And the pipeline, in order:

> "The Record Protocol takes an application message to be transmitted, fragments the data into
> manageable blocks, optionally compresses the data, **applies a MAC, encrypts**, adds a header, and
> transmits the resulting unit in a TCP segment. Received data are decrypted, verified, decompressed,
> and reassembled and then delivered to higher-level users." (§17.2, p. 533)

Note what this means for the two protocols' division of labour. AH and ESP are separate protocols
because the services are separable, and Stallings is explicit that an SA carries one or the other:
"An individual SA can implement either the AH or ESP protocol but not both" (§16.5, p. 503).
Authentication without confidentiality is a legitimate and *named* configuration — a fact worth
remembering when someone assumes encryption implies integrity. See
`05-integrity-hashes-and-macs.md` §2.1; the mistake is V-15.

---

## 3. Replay defence, concretely: the anti-replay window (§16.3, pp. 494–495)

This is the most directly reusable algorithm in either chapter. Every system that accepts
authenticated messages over an unreliable transport needs it, and almost none implement it.

The mechanism starts with a counter:

> "When a new SA is established, the sender initializes a sequence number counter to 0. Each time that
> a packet is sent on this SA, the sender increments the counter and places the value in the Sequence
> Number field. Thus, the first value to be used is 1." (§16.3, p. 494)

Then the rule that most homegrown schemes miss entirely:

> "If anti-replay is enabled (the default), the sender **must not allow the sequence number to cycle
> past 2³² − 1 back to zero.** Otherwise, there would be multiple valid packets with the same sequence
> number. If the limit of 2³² − 1 is reached, the sender **should terminate this SA and negotiate a new
> SA with a new key.**" (§16.3, p. 494)

A wrapped counter reintroduces duplicate-but-valid messages, and the prescribed response is not to
wrap gracefully — it is to **rekey**. This is the same rule as nonce reuse in a counter mode
(`02-cryptographic-primitives-and-modes.md` §5.4): exhausting the counter space ends the key's life.

Why a window, rather than "accept only N+1":

> "Because IP is a connectionless, unreliable service, the protocol does not guarantee that packets
> will be delivered in order and does not guarantee that all packets will be delivered. Therefore, the
> IPSec authentication document dictates that the receiver **should implement a window of size W, with
> a default of W = 64.**" (§16.3, p. 494)

> "The right edge of the window represents the highest sequence number, N, so far received for a valid
> packet. For any packet with a sequence number in the range from N − W + 1 to N that has been
> correctly received (i.e., properly authenticated), the corresponding slot in the window is marked."
> (§16.3, p. 494)

The three acceptance rules (§16.3, p. 495), which are worth transcribing into any at-least-once
message consumer:

1. "If the received packet falls within the window **and is new**, the MAC is checked. If the packet is
   authenticated, the corresponding slot in the window is marked."
2. "If the received packet is to the right of the window and is new, the MAC is checked. If the packet
   is authenticated, the window is advanced so that this sequence number is the right edge of the
   window, and the corresponding slot in the window is marked."
3. "If the received packet is to the left of the window, or if authentication fails, **the packet is
   discarded; this is an auditable event.**"

Three details carry the whole design. **Authentication is checked only after the cheap window test** —
so replayed traffic cannot be used to force expensive verification. **The window advances only on a
packet that authenticates** — otherwise an attacker moves the window forward with garbage and causes
the receiver to drop legitimate traffic. And **a rejection is an audit event**, not a silent drop:
this is where the audit-record discipline of `10-intrusion-detection-and-audit.md` attaches to the
channel.

The SA also carries a flag for the overflow case — "Sequence Counter Overflow: A flag indicating
whether overflow of the Sequence Number Counter should generate an auditable event and prevent further
transmission of packets on this SA" (§16.2, p. 490). The behaviour at the boundary is configuration,
declared in advance, not an accident.

**The review question this produces.** For any consumer of signed webhooks, queue messages, or
device telemetry: what is the replay window, how are seen identifiers stored, for how long, and what
happens at the edges — a duplicate inside the window, a message far in the future, a counter that
wraps? "We verify the signature" answers none of those. Absent replay defence is V-25; a window that
is missing or a counter allowed to wrap is V-40.

---

## 4. Ordering, nesting, and what the authentication covers (§16.5, pp. 503–504)

IPsec lets you combine SAs, and the discussion of how is the clearest treatment in the book of
authenticate-then-encrypt versus encrypt-then-authenticate.

> "The term **security association bundle** refers to a sequence of SAs through which traffic must be
> processed to provide a desired set of IPSec services." (§16.5, p. 503)

With ESP's built-in authentication option, "for both cases, **authentication applies to the ciphertext
rather than the plaintext**" (§16.5, p. 504) — that is encrypt-then-MAC, the ordering modern practice
insists on because it lets the receiver reject forgeries without touching the decryption path.

Stallings then gives the argument for the other order, and it is a good argument, which is why the
question is worth understanding rather than memorising:

> "The use of authentication prior to encryption might be preferable for several reasons. First,
> because the authentication data are protected by encryption, it is impossible for anyone to
> intercept the message and alter the authentication data without detection." (§16.5, p. 504)

> "Second, it may be desirable to **store the authentication information with the message at the
> destination for later reference.** It is more convenient to do this if the authentication information
> applies to the unencrypted message; otherwise the message would have to be reencrypted to verify the
> authentication information." (§16.5, p. 504)

The second reason is the durable one, and it is not about ordering within a channel at all — it is
about the difference between a channel property and an object property. A MAC or signature over the
plaintext is *evidence about the message* that survives decryption and storage; a MAC over the
ciphertext is evidence about *this transmission*. If you need to prove later what was sent, you need
the former.

The coverage argument matters too. Stacking AH outside ESP is more expensive but authenticates more:

> "The advantage of this approach over simply using a single ESP SA with the ESP authentication option
> is that **the authentication covers more fields, including the source and destination IP addresses.**
> The disadvantage is the overhead of two SAs versus one SA." (§16.5, p. 504)

**Generalise it: whatever is outside the authenticated region is attacker-controlled.** Routing
metadata, headers, envelope fields, the algorithm identifier, the key id — if a field influences how
the receiver interprets the message and is not covered by the MAC, an attacker can change it. This is
V-20, and the IPsec answer is the design answer: pull the field inside the authenticated region, or
accept that it cannot be trusted.

> **Modern.** Prefer an AEAD (AES-GCM, ChaCha20-Poly1305) and pass everything that must be
> authenticated-but-not-encrypted as *associated data*. That gives you Stallings' "authentication
> covers more fields" without a second pass, a second key, or a second protocol layer. TLS 1.3 removed
> the non-AEAD suites entirely, which retired a decade of MAC-ordering vulnerabilities.

---

## 5. Negotiation is an attack surface (§17.2, pp. 533–549)

The handshake exists to agree on parameters, and every agreement is an opportunity to agree on
something weak.

> "The Handshake Protocol allows the server and client to **authenticate each other** and to negotiate
> an encryption and MAC algorithm and cryptographic keys to be used to protect data sent in an SSL
> record." (§17.2, p. 538)

> "**CipherSuite:** This is a list that contains the combinations of cryptographic algorithms supported
> by the client, in decreasing order of preference." (§17.2, p. 539)

> "The **Version field contains the lower of the version suggested by the client and the highest
> supported by the server.**" (§17.2, p. 539)

That last rule is downgrade by design: the parties end up at the weaker of the two, so an attacker who
can influence either side's advertised set steers the outcome. The 2005 text still lists 40-bit
suites as legitimate options — "RC4-40 40", "DES-40 40" (§17.2, p. 534) — with a session parameter
"IsExportable: True or False" (§17.2, p. 540) and a dedicated alert, "export_restriction: A
negotiation not in compliance with export restrictions on key length was detected" (§17.2, p. 547).
Deliberately weak, negotiable options were a normal part of the protocol. That is V-38, and the
history of protocol downgrade attacks is the history of leaving them enabled.

The countermeasure is in the protocol, and it is the pattern to copy: **bind the negotiation to the
session by authenticating the entire transcript.**

> "The **finished** message verifies that the key exchange and authentication processes were
> successful." (§17.2, p. 543)

> "handshake_messages is all of the data from all handshake messages up to but not including this
> message." (§17.2, p. 543)

Because the finished message hashes every prior handshake message under the freshly derived secret,
any tampering with the advertised cipher suites or versions produces a mismatch and the connection
dies. The server's key exchange signature does the same for the nonces: "So the hash covers not only
the Diffie-Hellman or RSA parameters, **but also the two nonces from the initial hello messages. This
ensures against replay attacks and misrepresentation**" (§17.2, p. 541) — with the nonces' purpose
stated plainly: "These values serve as nonces and are used during key exchange to prevent replay
attacks" (§17.2, p. 539).

**A negotiation not covered by a transcript hash is a negotiation an attacker can rewrite.** This is
V-42, and it applies to anything with a capability-exchange step: your own protocol handshakes,
feature negotiation between services, and content negotiation that selects a security-relevant codec
or parser.

Two more negotiation lessons:

- **Mutual authentication is optional in the protocol and therefore optional in practice.** "Next, a
  nonanonymous server (server not using anonymous Diffie-Hellman) can request a certificate from the
  client" (§17.2, p. 541) — and if none is available "the client sends a `no_certificate` alert
  instead" (§17.2, p. 542). Server-only authentication is the default; client authentication is a
  thing you must ask for. When it *is* used, note what makes it sound: the `certificate_verify`
  message proves possession, not just presentation — "the purpose is to verify the client's ownership
  of the private key for the client certificate. **Even if someone is misusing the client's
  certificate, he or she would be unable to send this message**" (§17.2, p. 543). That is the
  authenticator idea from `07-identity-certificates-and-pki.md` §2: a certificate identifies, a fresh
  proof-of-possession authenticates. One-sided authentication where both sides act on the result is
  V-31.
- **Anonymous key agreement is a listed option.** "The certificate message is required for any agreed-on
  key exchange method **except anonymous Diffie-Hellman**" (§17.2, p. 541). Unauthenticated DH is
  V-35; see `04-public-key-and-key-exchange.md` §5 for the man-in-the-middle it enables.

---

## 6. Sessions, resumption, and cached security state (§17.2, pp. 532–537)

The session/connection split is a performance optimisation with security consequences, and the
definitions say so:

> "**Session:** An SSL session is an association between a client and a server… Sessions define a set
> of cryptographic security parameters, **which can be shared among multiple connections. Sessions are
> used to avoid the expensive negotiation of new security parameters for each connection.**"
> (§17.2, p. 532)

> "**Connection:** … For SSL, such connections are peer-to-peer relationships. **The connections are
> transient.** Every connection is associated with one session." (§17.2, p. 532)

So authentication happens once, per session, and its result is then reused by many connections. Two
state variables control the reuse: "**Session identifier:** An arbitrary byte sequence chosen by the
server to identify an active or resumable session state" (§17.2, p. 532) and "**Is resumable:** A flag
indicating whether the session can be used to initiate new connections" (§17.2, p. 533).

And the invalidation rule:

> "If the level is fatal, SSL immediately terminates the connection. Other connections on the same
> session may continue, but **no new connections on this session may be established.**" (§17.2, p. 537)

**This is a general pattern for any cached authorization decision.** Authenticate once, reuse the
result many times, and define exactly when the cached result stops being valid. A design that caches
an auth decision without an invalidation rule has made the reuse permanent — see V-26 on unbounded
lifetime, and V-30 on failing to renew session state at a privilege change. The identifier is also
"chosen by the server," which means it must be unpredictable: a guessable session identifier is
V-09/V-11 territory (`03-randomness-and-key-management.md` §1.6).

Closure is part of the protocol too:

> "**close_notify:** Notifies the recipient that the sender will not send any more messages on this
> connection. **Each party is required to send a close_notify alert before closing the write side of a
> connection.**" (§17.2, p. 537)

The requirement exists so that the receiver can tell "the stream ended" from "the stream was cut."
Without it an attacker who can drop packets can silently truncate a response, and the receiver treats
a partial message as complete. That is V-41, and it applies well beyond TLS: any framing where
end-of-stream is inferred from the transport rather than declared in the data has the same defect.

---

## 7. The TLS-over-SSLv3 fixes worth generalising (§17.2, pp. 545–549)

The v3→TLS delta is a compact list of "we built it ourselves and then replaced it with the analysed
construction."

- **The homemade MAC was replaced by HMAC.** "There are two differences between the SSLv3 and TLS MAC
  schemes: the actual algorithm and the scope of the MAC calculation. **TLS makes use of the HMAC
  algorithm defined in RFC 2104**" (§17.2, p. 545). What SSLv3 did instead is the exact construction
  `05-integrity-hashes-and-macs.md` §3 forbids: "SSLv3 uses the same algorithm, except that the
  padding bytes are **concatenated with** the secret key rather than being XORed with the secret key
  padded to the block length" (§17.2, p. 545). A shipped, widely deployed protocol got this wrong.
  That is V-18, in the wild, at scale.
- **The MAC scope grew to cover the version.** "The MAC calculation covers all of the fields covered by
  the SSLv3 calculation, **plus the field TLSCompressed.version**, which is the version of the protocol
  being employed" (§17.2, p. 545). A version field outside the MAC is a version field an attacker can
  change (§4).
- **The MAC inputs are the anti-tampering design.** `HMAC_hash(MAC_write_secret, seq_num ||
  TLSCompressed.type || TLSCompressed.version || TLSCompressed.length || TLSCompressed.fragment)`
  (§17.2, p. 545). Sequence number defeats reordering and replay; type prevents a record of one kind
  being reinterpreted as another; length prevents re-framing; and each direction has its own counter,
  reset at each cipher change — "Each party maintains separate sequence numbers for transmitted and
  received messages for each connection. When a party sends or receives a change cipher spec message,
  the appropriate sequence number is set to zero" (§17.2, p. 533). **Separate counters per direction is
  the fix for backward replay** (`06-signatures-and-authentication-protocols.md` §4.1): with a shared
  counter, a message reflected back at its sender can verify.
- **Padding became variable to hide length.** "In TLS, the padding can be any amount that results in a
  total that is a multiple of the cipher's block length, up to a maximum of 255 bytes" (§17.2, p. 549)
  — because "a variable padding length may be used **to frustrate attacks based on an analysis of the
  lengths of exchanged messages**" (§17.2, p. 549). Length is a side channel. Compression plus
  encryption over attacker-influenced data is the modern version of this problem (CRIME, BREACH), and
  the fix is not to compress secrets alongside attacker input.
- **Fields that added no security were removed.** "In the TLS certificate_verify message, the MD5 and
  SHA-1 hashes are calculated only over handshake_messages. Recall that for SSLv3, the hash calculation
  also included the master secret and pads. **These extra fields were felt to add no additional
  security**" (§17.2, p. 548). Extra cryptographic ingredients are not extra security; they are extra
  surface. See `02-cryptographic-primitives-and-modes.md` §6.

---

## 8. Key exchange must survive being cheap to attack (§16.6, pp. 507–508)

Oakley is presented as Diffie-Hellman plus fixes, and the fixes name the failure modes.

> "It is subject to a **man-in-the-middle attack**, in which a third party C impersonates B while
> communicating with A and impersonates A while communicating with B." (§16.6, p. 507)

> "It is **computationally intensive.** As a result, it is vulnerable to a **clogging attack**, in which
> an opponent requests a high number of keys. The victim spends considerable computing resources doing
> useless modular exponentiation rather than real work." (§16.6, pp. 507–508)

The three countermeasures (§16.6, p. 508): "It employs a mechanism known as **cookies** to thwart
clogging attacks." / "It uses **nonces** to ensure against replay attacks." / "It **authenticates the
Diffie-Hellman exchange** to thwart man-in-the-middle attacks."

The cookie mechanism is the general pattern for protecting an expensive operation from an
unauthenticated caller, and it is worth reading closely:

> "The cookie exchange requires that each side send a pseudorandom number, the cookie, in the initial
> message, which the other side acknowledges. This acknowledgment must be repeated in the first message
> of the Diffie-Hellman key exchange. **If the source address was forged, the opponent gets no answer.
> Thus, an opponent can only force a user to generate acknowledgments and not to perform the
> Diffie-Hellman calculation.**" (§16.6, p. 508)

Plus two design constraints on the cookie itself:

> "The cookie **must depend on the specific parties.** This prevents an attacker from obtaining a cookie
> using a real IP address and UDP port and then using it to swamp the victim with requests from
> randomly chosen IP addresses or ports." (§16.6, p. 508)

> "The cookie generation and verification methods **must be fast** to thwart attacks intended to
> sabotage processor resources." (§16.6, p. 508)

**Three transferable rules.** Prove round-trip reachability before spending real resources. Bind the
proof to the specific requester so it cannot be replayed from elsewhere. Make the check itself cheap,
or you have moved the denial-of-service target rather than removed it. Committing expensive work to an
unauthenticated request is V-63; see `11-malicious-software-and-availability.md` §5 for the same
shape at the network layer.

---

## 9. Two things IPsec makes explicit that most designs leave implicit

**Security associations are directional.** "An association is a **one-way relationship** between a
sender and a receiver that affords security services to the traffic carried on it. If a peer
relationship is needed, for two-way secure exchange, **then two security associations are required**"
(§16.2, p. 490). Separate keys per direction, by construction — no shared keystream, no reflected
message that verifies. See V-05 and `03-randomness-and-key-management.md` §2.3.

**Every SA has a declared lifetime.** "**Lifetime of This Security Association:** A time interval or
byte count after which an SA must be replaced with a new SA (and new SPI) or terminated, plus an
indication of which of these actions should occur" (§16.2, p. 491). Note the shape: a bound on *both*
time and volume, and a declared action at the limit. Keys also carry their own: "AH Information:
Authentication algorithm, keys, **key lifetimes**, and related parameters" (§16.2, p. 491).

**Copy this into your own key and credential design.** A key with no expressed lifetime, no byte or
message budget, and no defined behaviour at the limit is V-13 — and the anti-replay counter of §3
means a byte budget is not optional bookkeeping, it is a correctness bound.

The traffic-analysis point is also worth keeping: tunnel mode "can be used to counter traffic
analysis" whereas transport mode leaves the fact that "it is possible to do traffic analysis on the
transmitted packets" (§16.4, p. 502). Encryption hides content, not the existence, size, timing, or
destination of a conversation. If the metadata is the sensitive part — who talked to which service,
how often, at what volume — encryption alone does not address it.

---

## 10. A separate lesson from SET: give each party only what it needs (§17.3, pp. 549–556)

SET is dead, but its dual signature encodes a principle that applies to every integration:

> "An interesting and important feature of SET is that it **prevents the merchant from learning the
> cardholder's credit card number**; this is only provided to the issuing bank." (§17.3, p. 550)

> "The purpose of the dual signature is to link two messages that are intended for two different
> recipients… **The merchant does not need to know the customer's credit card number, and the bank does
> not need to know the details of the customer's order.**" (§17.3, p. 553)

> "The customer is afforded extra protection in terms of privacy by keeping these two items separate.
> **However, the two items must be linked in a way that can be used to resolve disputes if
> necessary.**" (§17.3, p. 553)

And the attack that the linkage prevents, which is a nice example of why data separation alone is not
enough:

> "To see the need for the link, suppose that the customers send the merchant two messages: a signed OI
> and a signed PI, and the merchant passes the PI on to the bank. **If the merchant can capture another
> OI from this customer, the merchant could claim that this OI goes with the PI rather than the
> original OI.** The linkage prevents this." (§17.3, p. 553)

**Two rules for third-party integrations.** Send each party the minimum it needs to do its job — a
payment processor does not need the cart contents, an analytics vendor does not need the email
address, a webhook consumer does not need the full record. And when related facts are split across
parties, **bind them cryptographically**, or an intermediary can recombine them dishonestly. The
mix-and-match attack above is the same class as V-20: independently valid pieces, reassembled into a
claim nobody authorised.

---

## 11. Review checklist

**Layer and scope**

- [ ] Which layer provides each security service, and can that layer express the property required?
      (Channel authentication ≠ message attribution; see V-39.)
- [ ] Does protection end at a terminating proxy, gateway, or mesh sidecar? What sees plaintext after
      that point, and is that acceptable?
- [ ] Is any traffic exempted from protection because it is "internal"? That is a trust-by-network-
      position decision (V-46), and it should be stated deliberately.
- [ ] If the payload must be verifiable after it is stored or forwarded, is there an object-level
      signature and not just a protected channel?

**Channel configuration**

- [ ] Is certificate and hostname verification on, everywhere, including internal calls and test
      harnesses that could ship? (V-32)
- [ ] Is a minimum protocol version and cipher suite set enforced by policy, so negotiation cannot
      settle on the weaker option? (V-38)
- [ ] Is mutual authentication used where the design requires the client to be identified — and is it
      proof-of-possession, not just certificate presentation? (V-31)
- [ ] Is key agreement authenticated? (V-35)
- [ ] Is an AEAD used, with everything that must be authenticated-but-visible passed as associated
      data?

**Replay, ordering, and framing**

- [ ] Is there a replay defence on every authenticated message stream: a window, a seen-identifier
      cache, or a bounded freshness check? (V-25, V-40)
- [ ] What is the window size, how long are identifiers retained, and what bounds that retention?
- [ ] Is the cheap window/freshness test done *before* expensive verification, and does the window
      advance only on a message that verifies?
- [ ] Can the counter or identifier space wrap or be exhausted? What happens at the limit — is rekeying
      or termination defined?
- [ ] Are separate counters or keys used per direction, so a message cannot be replayed back at its
      sender? (V-05)
- [ ] Is end-of-stream declared in the authenticated data rather than inferred from the transport?
      (V-41)
- [ ] Are rejected messages recorded as security events, not silently dropped? (V-55)

**Negotiation and session state**

- [ ] Is every negotiated parameter covered by a transcript hash or equivalent binding? (V-42)
- [ ] Is every field that influences interpretation — version, algorithm, key id, length, type,
      audience — inside the authenticated region? (V-20)
- [ ] Are cached authentication or authorization decisions given an explicit invalidation rule and a
      maximum age? (V-26)
- [ ] Is session state renewed at authentication and at every privilege change? (V-30)
- [ ] Are session identifiers generated by a cryptographically secure generator and long enough to
      resist guessing? (V-09, V-11)

**Keys, resources, and metadata**

- [ ] Does every key and credential have a declared lifetime in both time and volume, with a defined
      action at the limit? (V-13)
- [ ] Is expensive cryptographic or connection-establishment work gated behind a cheap, requester-bound
      round-trip proof? (V-63)
- [ ] If traffic metadata (peers, volumes, timing, sizes) is itself sensitive, is that addressed —
      because encryption does not address it?
- [ ] For third-party integrations: does each party receive only the data it needs, and are related
      facts split across parties bound together cryptographically?
