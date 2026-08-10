---
layout: post
title: "The Ciphertexts Are Not What They Seem"
date: 2026-08-10
categories: [security, crypto, python]
image: /assets/the-ciphertexts-are-not-what-they-seem.png
---

![The Ciphertexts Are Not What They Seem](/assets/the-ciphertexts-are-not-what-they-seem.png)

On June 22, 2026, the White House ordered federal agencies to accelerate the transition to post-quantum cryptography. Note the verb. Not "begin." *Accelerate.* NIST has been running the PQC competition since 2016 - this thing is older than most of your `node_modules` - and in August 2024 it crowned the winners: FIPS 203 (ML-KEM) for key exchange, FIPS 204 (ML-DSA) for signatures. So no, Washington didn't panic. Washington did something far more unsettling: it read a standards document to completion.

The rationale is "harvest now, decrypt later" - adversaries recording raw ciphertext today so they can crack it in 15 years, once a cryptographically relevant quantum computer finally finishes booting. The easy joke writes itself: some analyst in Langley sweating over a disk array going, "Sir, if we don't act NOW, foreign intelligence will learn the exact payload Dave sent to his Redis cache in 2026!" And sure, in 2040 nobody gives a shit about Dave's session token. Dave's session token doesn't give a shit about Dave's session token. It expired 47 minutes after issuance.

But the NSA was never worried about Dave. It's worried about the stuff riding the same TLS pipes as Dave: medical records, diplomatic cables, source identities, database replication of anything with a 25-year secrecy lifetime. HNDL is the rare threat model that's just arithmetic. If (years your data must stay secret) plus (years your org needs to migrate) exceeds (years until the quantum computer exists), you are already late. And if the "years to migrate" term sounds pessimistic, please recall that it's 2026 and I can still find MD5 in production. Cryptographic migrations move at the speed of the slowest appliance nobody remembers purchasing.

Which is why open source moved first. `pyca/cryptography` v48 - Trail of Bits, funded by the Sovereign Tech Agency - put ML-KEM and ML-DSA one `pip install` away from the entire Python ecosystem. Well. One `pip install` away, initially, if your backend was AWS-LC or BoringSSL, since the wheels ship OpenSSL, meaning day-one quantum resistance was reserved for the kind of person who compiles their own crypto backend, i.e. the five people who already knew all of this. OpenSSL 3.5 support followed, and now the wheels everyone actually installs carry the goods.

Now, the honest part. Quantum resistance isn't a free swap. We traded elliptic curves for lattice math, and lattice math is *thicc*:

* Ed25519 signatures were a sleek 64 bytes. ML-DSA-65 signatures: **3,309 bytes** (and the public key balloons from 32 to 1,952).
* X25519 put 32 bytes on the wire. An ML-KEM-768 ciphertext: **1,088 bytes**.

We didn't defeat quantum physics. We made the payloads so fat they constitute a denial-of-service attack on the intercepting agency's storage budget. Harvest *that*.

But here's the correction the doomer version of this post skips: you pay in bytes, not cycles. ML-KEM keygen and encapsulation are computationally *fast*, competitive with X25519. And bandwidth is the one resource 2026 has in obscene surplus. You ship four megabytes of JavaScript to render a login form. You can afford one kilobyte to protect it.

Also true: ML-KEM isn't Diffie-Hellman anymore. Nobody derives a shared secret together over a candlelit handshake. The sender encapsulates a key and blasts a 1KB ciphertext at you. Unromantic. But notice nobody's asking you to bet civilization on lattices alone. The deployed reality is hybrid X25519MLKEM768, where an attacker has to break the curve *and* the lattice. It's already the default in Chrome and across Cloudflare's edge, and Signal and iMessage went post-quantum before your compliance team learned the acronym. This isn't bleeding edge. You're behind the *messaging apps*.

Which brings us to the real terror: if your legacy protocol hardcoded a 32-byte key field, or a middlebox somewhere "helpfully" assumes handshakes fit in one packet, the migration will find it. Chrome's rollout already flushed out ossified network gear choking on bigger ClientHellos. That's not an argument against migrating. It's the argument *for starting now*. You want to discover that hardcoded buffer in a planned rollout on a Tuesday in 2026, not in an incident channel in 2033 while an auditor watches.

`pip install -U cryptography` is the cheap part. Finding every place your stack quietly assumed keys are 32 bytes forever, that's the migration. The math is sound, the bytes are fat, and the deadline is real.

Encapsulate, Dave.

---

*Cover: Pieter Bruegel the Elder, "The Harvesters" (1565), public domain, via Wikimedia Commons.*
