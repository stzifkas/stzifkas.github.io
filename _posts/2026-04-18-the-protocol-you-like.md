---
layout: post
title: "The Protocol You Like Is Going to Come Back in Style"
date: 2026-04-18
categories: security
image: /assets/the-protocol-you-like.png
---

![The Protocol You Like Is Going to Come Back in Style](/assets/the-protocol-you-like.png)

There is a certain kind of software dream that always arrives dressed like innocence.

A little chat box. A few names in a sidebar. Some circles turning green. A message sent. A message received. A clean interface, nice spacing, a calm typeface, the illusion of ordinary life. And lately a fourth presence in the room — a small inline assistant, ready to summarize the thread, draft the reply, translate, transcribe, or quietly remember things for later.

But under the floorboards there is another story, and it is never ordinary. Under the floorboards there is key material, ratchets, tree paths, signature checks, nonces that must never repeat, public keys that look like harmless 32-byte strings and are in fact small pieces of a war against compromise, subpoenas, database leaks, rogue admins, future attackers, human forgetfulness — and now a new participant whose memory is less like a person's and more like a room full of GPUs that decline to forget on demand.

This is the long walk from "E2EE is that thing Signal does" to "all right, I can actually read the protocol now, and I can also see why putting an LLM in the middle of it is the most interesting threat model the field has acquired in years."

Not a pitch. Not a product page. Just the mechanics. Just the wires under the wallpaper.

The basic idea is simple enough that it almost feels suspicious. In a normal web app, your message travels under TLS to the server, the server decrypts it, stores plaintext, maybe indexes it, maybe backs it up, maybe hands it to another client later over TLS again. Transport encryption protects you from the guy sniffing packets on bad Wi-Fi. It does not protect you from the operator of the service, from an admin who goes bad, from a compromised database, from a cloud provider with too much visibility, or from legal compulsion. The server is trusted because it has to be. That is the whole architecture.

End-to-end encryption changes the trust boundary. The client encrypts before the server ever sees content. The server stores ciphertext, forwards ciphertext, replicates ciphertext, backs up ciphertext. Other clients decrypt. The server becomes a delivery service, not an interpreter of meaning. It still sees metadata, because metadata is the cigarette smoke that lingers in every room: who talked to whom, when, how often, in what group. But it does not get the text itself.

That sounds clean, and it is clean, but not cheap. Search becomes hard because the server cannot index what it cannot read. Key loss becomes message loss because there is no honest "reset password" button for ciphertext. Group membership becomes a cryptographic event rather than a database update. Adding someone means giving them cryptographic access. Removing someone means rotating future secrets so they cannot follow the next epoch forward.

That is where the old beautiful protocols come back in style. Not as nostalgia. As necessity.

And today? They are not just back. They are shipping. MLS, less than two years after becoming a published RFC, is poised for deployment across Android phones and iPhones thanks to a refreshed RCS specification, finally enabling interoperable encryption between platform vendors. The protocol that used to live in academic slides now lives in the SMS replacement on a billion devices.

## The bricks, before the cathedral

Most modern cryptography is just six or so primitives stacked with discipline. The magic dissolves once you see that. You stop thinking in terms of mysterious "military grade encryption" and start thinking in terms of a few hard tools used correctly.

A hash function such as SHA-256 takes arbitrary input and gives back a fixed 32-byte output:

```text
H(m) -> 32 bytes
```

It is deterministic. It is one-way, in the practical sense. It is collision resistant, meaning you should not be able to find two distinct messages with the same digest. In real protocol design, hashes are not usually used alone for secrecy. They show up as ingredients: in HMAC, in HKDF, in transcript hashes, in integrity checks.

Then comes HMAC, which is what happens when you take a hash and give it a key:

```text
HMAC(k, m) = H((k ⊕ opad) || H((k ⊕ ipad) || m))
```

That funny nested construction exists because the obvious thing, hashing `k || m`, is not robust enough. HMAC is the proper way to say: whoever made this authentication tag knew the secret key. It is one of those places where the right construction looks ugly because it has already survived contact with many clever attacks.

HKDF is next. Diffie–Hellman shared secrets are high entropy, but protocol designers do not just jam raw group elements into AES and hope for the best. HKDF turns input keying material into well-separated, uniform subkeys:

```text
PRK   = HMAC(salt, IKM)
OKM_i = HMAC(PRK, OKM_{i-1} || info || i)
```

The first step, extract, normalizes entropy. The second, expand, produces as many bytes as you need and labels them with context through the `info` field. One shared secret can safely give you a chain key, a nonce base, a message key, and more, as long as you domain-separate correctly.

Then there is AEAD, typically AES-128-GCM in this setting:

```text
ciphertext, tag = AEAD_enc(key, nonce, plaintext, aad)
plaintext        = AEAD_dec(key, nonce, ciphertext, tag, aad)
```

Authenticated encryption with associated data means one operation gives confidentiality and integrity together. The plaintext is encrypted. The `aad` is not encrypted, but it is authenticated, which is often exactly what you want for fields like channel identifiers or epochs. The critical rule with GCM is brutal and absolute: never reuse a nonce with the same key. Not "usually avoid." Never. Reuse is catastrophic. Protocols that derive nonces carefully are not being fussy. They are staying alive.

For signatures, Ed25519 has become the civilized default. One private signing key. One public verification key. Small keys, small signatures, fast operations, deterministic signing, a hard-to-misuse API:

```python
def public_raw(self) -> bytes:
    return self.public_key.public_bytes(
        encoding=serialization.Encoding.Raw,
        format=serialization.PublicFormat.Raw,
    )

def private_raw(self) -> bytes:
    return self.private_key.private_bytes(
        encoding=serialization.Encoding.Raw,
        format=serialization.PrivateFormat.Raw,
        encryption_algorithm=serialization.NoEncryption(),
    )

@classmethod
def generate(cls) -> "IdentityKeyPair":
    priv = Ed25519PrivateKey.generate()
    return cls(private_key=priv, public_key=priv.public_key())
```

This is identity. The device signs things with it. The private half stays on the device. The public half can travel.

For key exchange, X25519 gives you elliptic-curve Diffie–Hellman stripped down to something lean and practical. Alice has private scalar `a` and public point `aG`. Bob has `b` and `bG`. Alice computes `a * (bG)`. Bob computes `b * (aG)`. Both arrive at the same shared secret, `abG`, without ever sending the private scalars over the wire.

```python
@classmethod
def generate(cls) -> "InitKeyPair":
    priv = X25519PrivateKey.generate()
    return cls(private_key=priv, public_key=priv.public_key())
```

That is your other fundamental tool: not identity, but key agreement.

Six pieces. SHA-256, HMAC, HKDF, AES-GCM, Ed25519, X25519. A strange little family. Enough to build a secure messaging system if you are disciplined, and enough to destroy one if you are not.

In 2026 there is a new generation joining the family on the public-key side — ML-KEM and ML-DSA, the lattice-based replacements — but the symmetric pillars (SHA-256, HMAC, HKDF, AES-GCM) carry over essentially unchanged. Symmetric cryptography is not what the quantum computer is going to break. We will return to this.

## The curve behind the curtain

If you keep looking at crypto long enough, you eventually hit the wall of mathematics and discover it is less a wall than a red curtain. Pull it back and the machinery is ugly but intelligible.

Curve25519 lives over a finite field modulo the prime $2^{255} - 19$. In one common form, the curve equation looks like this:

$$y^2 = x^3 + 486662x^2 + x \pmod{2^{255} - 19}$$

Points on this curve form a group under a specially defined addition law. Pick a base point $G$. Multiply it by an integer scalar $k$, and you get another point $kG$. The security assumption is that given $G$ and $kG$, recovering $k$ is computationally infeasible. That is the elliptic-curve discrete logarithm problem.

Everything starts leaning on that hardness assumption.

In X25519, your private key is a scalar and your public key is the corresponding point. Shared secret computation is just scalar multiplication performed from opposite sides. In Ed25519, signatures rely on related algebra. In a simplified form, verification checks a relation like

$$sG = R + H(R, A, m)A$$

where $A$ is the public key, $R$ is an ephemeral point, and $s$ is part of the signature. The beauty is not that it is simple. The beauty is that it is standardized, optimized, and old enough to trust more than anything newly improvised in a repo at 2:30 a.m.

When people say this ecosystem gives roughly 128-bit security, that means the best known attacks still require on the order of $2^{126}$ to $2^{128}$ work, depending on the primitive and attack model. That is why AES-128 is not some "smaller" and therefore weaker embarrassment next to AES-256. It is already absurdly secure in practice. AES-256 is not wrong; it is just often more aesthetic than necessary in this class of systems.

The lattice cousins, ML-KEM and ML-DSA, lean on different math entirely — module learning with errors. The hardness story over there is younger and the keys are larger, but the high-level shape of the API is reassuringly familiar: generate a keypair, encapsulate to a public key, decapsulate with the private key, get a shared secret out the other end. Same drama, different stage.

## The ciphersuite as a sentence

Sometimes protocols hide their entire worldview in one string. Consider this:

```text
mls-128-dhkemx25519-aes128gcm-sha256-ed25519
```

That is not branding. That is the full menu.

It says the protocol is MLS, Messaging Layer Security. It says the target security level is about 128 bits. It says the KEM side of HPKE uses X25519-based Diffie–Hellman. It says bulk authenticated encryption uses AES-128-GCM. It says the hash foundation is SHA-256. It says signatures are Ed25519.

A good ciphersuite string is like a proper street sign at night. It tells you exactly where you are.

The 2026 menu is longer. Alongside the classical suite, the working draft for post-quantum MLS registers names like:

```text
mls-256-dhkemx25519mlkem768-aes256gcm-sha384-ed25519mldsa65
mls-256-dhkemmlkem768-aes256gcm-sha384-mldsa65
```

The first is hybrid. Two key encapsulations stacked, classical and post-quantum, combined so that an attacker has to break both. The second is post-quantum only, no classical safety net, for when you have decided the lattice assumption is good enough on its own. Which one you pick is a real engineering question and a slightly philosophical one. Hybrid is the cautious answer. Pure PQ is the answer for people who think the classical curtain is going to fall and they would rather not be standing behind it.

## HPKE, the sealed envelope with an ephemeral key inside

MLS leans heavily on HPKE, Hybrid Public Key Encryption. HPKE is what you use when you want to encrypt to someone's public key but still end up with symmetric encryption under the hood, because public-key operations are for bootstrapping trust, not for carrying the bulk of your traffic.

The friendly API looks like this:

```text
ciphertext, enc = HPKE_Seal(recipient_pubkey, aad, plaintext)
plaintext       = HPKE_Open(recipient_privkey, enc, aad, ciphertext)
```

Underneath, the sender generates a fresh ephemeral X25519 keypair. Call it $(e, eG)$. The sender computes a Diffie–Hellman shared secret using $e$ and the recipient's public key. HKDF turns that shared secret into an AEAD key and nonce material. The plaintext gets sealed. The recipient uses their private key and the sender's ephemeral public key `enc` to derive the same shared secret and open the message.

That ephemeral key matters. It means a compromise later does not automatically reveal past HPKE-sealed payloads. Forward secrecy begins here, in these deliberately disposable little keys that exist just long enough to pass a secret across the gap and then disappear.

There is a separate IETF draft for post-quantum and hybrid HPKE, which slots ML-KEM into the same shape. Same envelope, different glue. The KEM step gets bigger. The AEAD step does not. The ergonomics of the calling code barely change. That is what good cryptographic abstractions look like — replaceable parts behind a stable seam.

## Asynchronous adds, or why prekeys exist

The first real awkwardness in messaging is this: how do you cryptographically include someone who is not online right now?

The answer, in Signal land and in MLS land alike, is some version of pre-published one-time public material. MLS calls them KeyPackages. Signal called them prekeys. Same basic energy, less mystique.

A device publishes signed bundles containing an identity public key, a fresh X25519 init public key, some metadata such as lifetime and ciphersuite, and a signature over the whole canonicalized object.

```python
bundle_obj = {
    "schema": KEYPACKAGE_SCHEMA,
    "ciphersuite": ciphersuite,
    "user_id": user_id,
    "device_id": device_id,
    "identity_pubkey": _b64(identity.public_raw()),
    "init_pubkey": _b64(init_key.public_raw()),
    "lifetime": lifetime,
    "nonce": _b64(secrets.token_bytes(16)),
}
bundle_bytes = json.dumps(
    bundle_obj,
    separators=(",", ":"),
    sort_keys=True
).encode("utf-8")

signature = identity.private_key.sign(bundle_bytes)

return {
    "ciphersuite": ciphersuite,
    "public_bundle": _b64(bundle_bytes),
    "signature": _b64(signature),
}
```

The canonical JSON detail looks boring and is not. Signatures are over bytes, not abstract objects. If two implementations serialize the same object differently, verification breaks. So you sort keys, strip whitespace, and commit to one representation. Real MLS implementations use TLS presentation language for this sort of thing, but the principle is the same: canonicalize or die by ambiguity.

The one-shot property also matters. A KeyPackage's init key is supposed to be consumed once. One Welcome, one use. Burn after reading. Reusing it stretches the blast radius of compromise across multiple onboarding events, which is exactly what you do not want.

Signal's PQXDH and the post-quantum extensions to MLS preserve this shape. The prekey bundle gets a lattice public key alongside the classical one. The handshake derives a shared secret from both. The code looks more verbose. The mental model survives intact.

## Why MLS exists instead of "just do Signal but for groups"

Pairwise encrypted messaging is manageable when the room is small. In a large group, naïve pairwise state turns ugly fast. If everyone has to maintain separate secure relationships with everyone else, membership changes become expensive and messy. MLS exists because secure groups needed a protocol designed for groups rather than retrofitted from two-party assumptions.

The central idea is TreeKEM. Imagine a binary tree. Leaves are devices. Internal nodes also carry key material. Each device knows the private keys on its path from leaf to root, and the root secret anchors the current group epoch.

```text
            root
           /    \
         n1      n2
        / \     / \
      L1  L2  L3  L4
```

A member at leaf `L1` knows `L1`, `n1`, and `root`. Another member on the same side knows a different leaf secret but the same path upward. Members on the opposite side know their own path. This structure lets the protocol update group secrets in logarithmic rather than quadratic fashion.

When membership changes, a member creates a fresh path from its leaf to the root. New path secrets are derived and corresponding public keys are distributed to the right sibling subtrees using HPKE. Then the epoch advances. New message keys, nonces, and sender state are derived from the new epoch secret.

You can write the spirit of it as:

$$\text{epoch}_{n+1} = \mathrm{KDF}(\text{epoch}_n, \text{commit}, \text{transcripthash})$$

The exact derivation structure is more elaborate, but the important thing is the property: each epoch change injects fresh entropy and re-anchors the group.

That gives you forward secrecy, meaning compromising today's state does not unlock yesterday's messages if old secrets were properly deleted. It also gives post-compromise security. If an attacker steals your current secrets but the group continues and a fresh commit lands, the attacker can be pushed back out of future epochs. This is one of the reasons MLS is so compelling. It does not just try to be secure at a frozen instant. It tries to heal.

The 2026 footnote: RFC 9750, the MLS Architecture document, was published in April 2025, codifying the operational guidance that the protocol document RFC 9420 deliberately left out. The GSMA's Universal Profile 3.0 made MLS the basis for end-to-end encryption in RCS, with Apple committing to support it on Apple Messages, while Matrix has announced its migration as well. What was a paper protocol is now the substrate of mass-market texting between rival platforms. That is, for a standards effort, a fairy-tale ending.

## Welcomes, removals, and the fact that crypto is not a time machine

When a new member is added, someone claims one of that member's KeyPackages, updates the tree, and creates a Welcome message encrypted to the recipient's init key. The Welcome contains enough information for the new member to enter the current epoch with the right tree context and path secrets.

That Welcome is the quiet doorway. It is how an asynchronous group says, you were absent, but the cryptography left the porch light on.

Removal is more sobering. When someone is removed, the group rotates forward. Their leaf is blanked, the tree updates, the epoch changes, and they cannot derive future keys. But the past remains the past. If they already received old ciphertext and had the right keys at the time, you do not get to reach into their memory and erase it. End-to-end encryption is powerful, but it does not reverse causality.

People often discover this too late. Revocation means no future access. It does not mean retroactive amnesia.

This becomes painfully relevant when the new member is not a person.

## The third party in the conversation

End-to-end encryption defines security in terms of who the "ends" are. For decades, the ends were humans, and the threat model was clear: anyone in the middle is suspect. In 2026, a third kind of end has shoved its way into the room. It does not have a face. It does not show up in the member list. It is an LLM, and somebody — maybe the platform, maybe a single member, maybe the user themselves — has plugged it into the conversation.

Once that happens, the trust model the protocol was designed around quietly bends.

Think about what the LLM has to be able to do in order to be useful. To summarize a thread, it must read the thread. To draft a reply, it must read the context. To translate, transcribe, search the chat history, remember preferences, suggest a meeting time — every one of those features requires plaintext on the LLM's side of the wire. End-to-end encryption was built to keep messages private, but that privacy starts to fray as soon as decrypted content is handed to an AI assistant. Even if the processing happens in a careful environment, the message is no longer constrained to sender and recipient.

This does not break the cryptographic protocol. The protocol still does exactly what it promises. The bytes are sealed in transit, the server still cannot read them, the tree still rotates, the epoch still advances. What breaks is the implicit social contract that "encrypted" meant "only the people in the chat will see this." When an assistant joins, the set of entities who see plaintext has grown by one. Sometimes that one is on your phone. Sometimes it is in a data center on another continent. Those two cases are not the same, and treating them as the same is how good protocols start producing bad outcomes.

There is also a collective consent problem that has no precedent in the cryptographic literature. If a single participant in a group enables an AI assistant, every other participant's messages may be processed by a model none of them opted into. The math of MLS does not know what to do with this. No epoch update will save you from a member who is faithfully relaying every commit message into a vendor's API. Honest behavior at the protocol layer can be perfectly compatible with what feels, at the human layer, like a quiet betrayal.

And then there is the new attack class. Vulnerabilities like EchoLeak demonstrated that AI assistants can be coaxed into leaking sensitive material — a category researchers have started calling LLM scope violation, where the model exposes context it was never supposed to share, based purely on how it interpreted a crafted prompt. This is the new nonce reuse. It does not look like the old attacks. It does not require breaking AES or finding a discrete log. It requires writing a sentence that sounds innocuous and is, in fact, a small key turned in a lock the assistant did not know it was guarding.

The protocol people have a useful instinct here, and it is worth borrowing: declare the threat model explicitly, then design for it. If the LLM is in the trust boundary, say so. If it is not, do not let it touch plaintext. Anything in between — "well, the model is processed in a special place" — is where the interesting security engineering of this decade is going to live.

## Private Cloud Compute, or the new attestation ceremony

The most elaborate answer to the LLM-in-the-middle problem, so far, is to push the model into a hardware-rooted enclave that the platform itself cannot read into.

Apple's Private Cloud Compute architecture works roughly like this: the device builds a request containing the prompt and inferencing parameters, encrypts it directly to the public keys of specific PCC nodes the device has cryptographically verified, and the data is supposed to be deleted after the response is returned and never available to Apple staff, including those with administrative access. The trust story is no longer "Apple promises." It is "Apple publishes the binary running on the node, the device attests that it is talking to that binary, and any deviation is detectable."

That is a meaningful change. It is also not magic. PCC and confidential computing are not the same thing — PCC focuses on hardening the communication path with verifiable software transparency, while confidential computing focuses on encrypting workloads in use within trusted execution environments, defending against malicious operating systems and hypervisors. The two approaches share primitives — TEEs, remote attestation, sealed channels — but they are answering slightly different questions. Whether either of them gives you the same guarantee that classical end-to-end encryption gives is a more subtle conversation than the marketing usually allows.

The broader pattern across the industry in 2026 is something called a private AI cloud. These architectures lean on three primitives — trusted execution environments that hardware-isolate memory from the host, GPU confidential compute (NVIDIA's offering extends the trust boundary to include the accelerator), and remote attestation that lets clients verify which code is actually running. Anthropic, Google, and Meta all have variants in production or development. The honest framing is that these clouds give real privacy benefits but do not deliver the same kind of mathematical guarantee as end-to-end encryption — users still have to trust hardware vendors, attestation infrastructure, and abuse-monitoring layers.

This matters because the security argument is no longer "we cannot read your data." It is "we have arranged the world such that, assuming the chip vendor did their job and the attestation chain holds, we should not be able to read it, and you can verify that arrangement before you send it." The shift from cryptographic impossibility to verified arrangement is enormous, and easy to miss.

There is a strain of thought that calls this a downgrade. There is another that calls it the only practical way to get useful AI without surrendering everything. Both are partly right.

## On-device, or, the small model that doesn't tell

The other answer — quieter, less heroic, increasingly viable — is to keep the model on the device.

The economic and regulatory pressure here is genuine. Local LLMs are gaining traction precisely because they sidestep concerns about transmitting sensitive material over the internet, and the on-device AI market is projected to grow into the tens of billions of dollars by 2030. Phones in 2026 routinely run quantized 1B to 3B parameter models well enough to handle summarization, translation, dictation, basic agent workflows, and the small set of tasks that used to require a network round trip. Cross-platform inference SDKs now report sub-50ms time-to-first-token for on-device models, eliminating network latency and defaulting to total privacy.

For the threat model we have been building, this is the cleanest fit. If the model lives on your device, then "the LLM saw it" and "your local app saw it" are the same statement. The tree of trust does not get an extra branch. The bytes do not leave. The Welcome message gets opened, the epoch keys get derived, the plaintext gets handed to a small model that runs on the same silicon as your photo library, and nothing crosses the network that was not already going to.

This is not a complete answer. The on-device models are smaller and dumber than their cloud cousins. Some workloads still demand the bigger brain. The compromise is roughly: do small things locally, and fall back to a verified private cloud only when you have to. That is the architecture quietly emerging in shipped products — Apple Intelligence, Proton's Lumo, certain enterprise RAG stacks. Lumo's privacy story explicitly grapples with what end-to-end encryption even means when one of the ends is a language model rather than a person, and tries to encrypt both in transit and at rest while still letting the model do useful work.

The protocol designers do not get to settle this debate for the product designers. But protocol designers can at least insist that the seam between "encrypted message" and "AI feature" be honest. If the assistant runs locally, say so. If it does not, say where it runs and what it is allowed to remember. Anything else is a rendering trick, and rendering tricks tend to age badly.

## The server as a dumb and useful machine

One of the most attractive architectural consequences of doing this properly is that the server gets demoted. Not removed, not romanticized, just demoted.

It authenticates users and devices. It enforces access control. It stores blobs. It forwards notifications. It keeps monotonic epoch state so clients do not fork the group history accidentally or maliciously.

A server-side epoch check can look as plain as this:

```python
if req.epoch != mls.epoch + 1:
    raise HTTPException(
        status_code=status.HTTP_409_CONFLICT,
        detail=f"epoch out of order: expected {mls.epoch + 1}, got {req.epoch}",
    )
```

That is almost disappointingly simple, which is a sign of a good separation of concerns. The server should not be doing cryptographic interpretation of runtime group content. It should not need private keys. It should not parse more than it must. It should not become the wise central brain that every future compromise wants to interrogate.

The beauty of opaque blob storage is not elegance alone. It is strategic humility. If later you swap a reference implementation for OpenMLS or AWS's mls-rs, or move from purely classical suites to hybrid post-quantum ones, the server ideally barely notices. If you decide to add an on-device assistant, the server still does not learn anything new. The server's ignorance is the system's strength.

## The threat model, without neon and lies

There is always a temptation in security writing to sound apocalyptic on the sales pages and omnipotent in the architecture docs. Better to stay sober.

What this kind of design defends against: passive network attackers, database readers, many classes of server compromise, leaked backups, and some forms of device compromise once the group rotates. Clients verify cryptographic commits, so a malicious admin can censor or destroy availability, but cannot forge valid group evolution undetected.

What it does not defend against: metadata analysis, compromised clients, an attacker who is legitimately added to a group, a future quantum adversary powerful enough to break the classical public-key assumptions underneath X25519 and Ed25519, and — newly important — any AI assistant that has been granted plaintext access to the conversation, regardless of how nicely its inference is wrapped.

Metadata remains a live wound. Even if content is sealed, the delivery service still sees patterns. Presence, read receipts, typing indicators, retention, and logs all become political choices as much as technical ones. Every extra signal you store is another little lantern for a future adversary.

A compromised client is game over for that endpoint. This is a hard truth worth stating plainly. If the software that performs encryption has already been subverted, then "but the protocol is sound" is a eulogy, not a defense. The 2026 corollary is harder still: an LLM-enabled client is a client whose attack surface includes prompt injection, model jailbreaks, and any data flow the assistant can reach. Securing the cryptography buys you nothing against an assistant that is smoothly persuaded to exfiltrate the very chat it just helped you summarize.

## The quantum ghost in the corner, now wearing a name tag

In the last edition of this essay, the quantum threat felt like a long shadow at the end of a hallway. It is closer now, and it has acquired specific names.

NIST has finalized its first post-quantum standards. ML-KEM, the lattice-based KEM previously known as Kyber, was standardized as FIPS 203 — strong security with relatively small keys and ciphertexts of around 1.5 KB, efficient on modest hardware, and the post-quantum half of most hybrid TLS and VPN deployments today. ML-DSA, formerly Dilithium, is its signature counterpart. In 2025 NIST also selected HQC, a code-based KEM, as a backup standard alongside Kyber, with a final standard expected by 2027. Two different mathematical families, in case one of them turns out to have a hidden window.

The migration is no longer hypothetical. AWS has begun rolling out ML-KEM support across services such as KMS, ACM, and Secrets Manager, with the older pre-standard Kyber implementations slated for removal across all AWS endpoints in 2026. The IETF has a working draft for MLS post-quantum ciphersuites that pairs ML-KEM with ML-DSA inside the existing protocol framing. The hybrid HPKE draft does the same for the envelope primitive everyone leans on.

The reason ciphersuite agility was always emphasized: this is exactly the moment it has to pay off. Architectures that hardwired X25519 are now staring down expensive surgery. Architectures that treated the ciphersuite as opaque metadata and let the clients decide are doing migrations as configuration changes.

A future hybrid suite looks like this:

```text
mls-128-dhkemx25519hybridmlkem768-aes128gcm-sha256-ed25519
```

The idea is to combine a classical component and a post-quantum KEM. Break both or fail. That is how sane migrations happen: hybrid first, caution always, no messianic declarations that one shiny new primitive has already replaced decades of cryptanalysis.

If your architecture stores the ciphersuite as opaque metadata and leaves cryptographic interpretation to the clients, then supporting such a migration is not a religious conversion. It is a controlled evolution.

That is what good protocol design looks like. Not immortality. Replaceable parts.

The harvest-now-decrypt-later concern, once theoretical, is the operational reason for moving early. If an adversary is recording your ciphertext in 2026 and a cryptographically relevant quantum computer exists in 2036, anything you sent under classical-only crypto today is potentially in their reading queue. The hybrid suites exist to take that bet off the table, retroactively, for the messages you send now.

## The ugly bits that are not the same as broken bits

Every real system has debt. The important distinction is between debt that is operational and debt that is cryptographic.

Race conditions in KeyPackage claims on a lightweight database. In-process pubsub that will not survive multi-worker scale. No finalized key backup and recovery design. Garbage collection of consumed prekeys left for later. An LLM integration that helpfully summarizes the chat into a third-party API without telling users where the bytes ended up. These are flaws, some annoying, some serious, but not all of them are cryptographic in nature.

A system can ship with operational debt and live to improve. A system that ships with nonce reuse, unauthenticated state transitions, or home-rolled key derivation is a crime scene wearing sneakers. A system that ships with an undisclosed AI assistant in the trust boundary is somewhere between the two — not a cryptographic break, but a category of breach that the user could not have consented to because they were not told it existed.

That distinction matters because protocol work attracts a lot of performative purity. Better to be exact. Some compromises are survivable. Others poison the well. And some, increasingly, look survivable in the architecture diagram and become catastrophic the moment you check what the assistant is actually doing with the plaintext.

## Why these old protocols still feel strange and new

There is a David Lynch quality to good security engineering. The surface looks domesticated. A room, a lamp, a clean UI, a person typing, a friendly assistant offering to summarize the day. But behind the wall there is another room, and behind that room there is machinery, and behind the machinery there is an old mathematics that does not care about your aesthetic preferences, and lately, behind all of that, there is a model with weights large enough to encode an unsettling amount of human writing, humming away on a chip in a building you have never visited.

You send a message and what really happens is a signature binds a canonical object, an ephemeral keypair blooms and dies, HKDF stretches a shared secret into clean material, a nonce is derived with priestly care, a tree path rotates, a transcript hash ratifies continuity, an epoch advances, a server shrugs and forwards bytes it cannot read — and then, optionally, on one device, the bytes are turned back into language and handed to a model that chooses words back. Ordinary life on top, algebra underneath, a speaking animal in a cage at the end.

This is why secure messaging is so easy to misunderstand. It looks like product behavior, but it is protocol behavior. It looks like UX, but under the UX is a chain of assumptions so brittle and exact that one reused nonce or one badly scoped secret can turn the whole palace into vapor. And with the assistant in the room, the brittle exactness extends now to a new question: what is the model allowed to remember, who decided, and how would you ever know.

And yet the result, when done right, is almost poetic in a severe, technical way. A group changes shape and the cryptography changes with it. A stolen key does not mean the attacker owns the future forever. A server becomes less powerful by design. Trust is pushed outward to endpoints and bounded by verifiable math rather than institutional promises. And, in the best versions of the assistant story, the model lives close to the user, the prompt never leaves, the helpful little voice is bounded by the same silicon that holds the keys.

The protocol you like is going to come back in style because the world keeps rediscovering the same hard lesson: if the middle can read everything, then eventually the middle matters too much. And when the middle matters too much, someone always tries to own it. The middle has a new shape now — it can be a database, a delivery service, a side channel, or a 70-billion-parameter model in a rack — but the lesson is unchanged.

So the old names return. Diffie–Hellman. EdDSA. HKDF. AEAD. HPKE. MLS. They do not return as retro chic. They return because the conditions that made them necessary never really left. They were just waiting under the stage lights for everybody else to catch up.

The new names are joining them. ML-KEM. ML-DSA. HQC. Confidential inference. Attested enclaves. On-device models. They are not replacements. They are companions. The cathedral is still being built. The bricks have just gotten a little bigger and a little stranger.

## Further reading, for when the curtain has lifted

If you want the standards behind the machinery, read RFC 9420 for MLS, RFC 9750 for the MLS Architecture, RFC 9180 for HPKE, RFC 7748 for X25519, RFC 8032 for Ed25519, RFC 5869 for HKDF, FIPS 203 for ML-KEM, and FIPS 204 for ML-DSA. For the works in progress, look at the IETF drafts for MLS post-quantum ciphersuites, HPKE post-quantum modes, and the CFRG hybrid KEM design.

If you want the historical road into this territory, read the Signal X3DH and Double Ratchet papers, then the Signal blog post introducing PQXDH. MLS did not appear from nowhere. It is what happens when the industry learns, painfully and repeatedly, that secure groups are not just pairwise messaging with more tabs open.

If you want the new territory of AI-meets-encryption, the NYU and Cornell paper "How to think about end-to-end encryption and AI" is a good entry point, alongside Apple's Private Cloud Compute write-up and Anthropic's Confidential Inference via Trusted Virtual Machines report. They will not give you a settled answer. They will give you the right questions, which is more useful at this stage.

And if you take only one practical lesson from all this, let it be this one: do not roll your own crypto. Do not, while you are at it, roll your own AI privacy story either. Even when you do not, even when you stand on standardized primitives and modern protocols and verifiable enclaves, there is still enough to think about to keep the room humming all night.