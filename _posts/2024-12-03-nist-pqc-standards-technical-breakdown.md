---
title: "FIPS 203, 204, and 205: What NIST's Post-Quantum Standards Actually Mean for Your Systems"
date: 2024-12-03 14:00:00 -0600
categories: [Research, Cryptography]
tags: [post-quantum, NIST, ML-KEM, ML-DSA, SLH-DSA, FIPS, lattice-cryptography, implementation]
author: siddarth
description: "August 2024 marked the end of a seven-year standardization process. Here's the technical breakdown nobody's giving you."
toc: true
comments: true
---

## The Standardization Process

I've been following NIST's post-quantum competition since Round 1 in 2017. Watched SIKE get absolutely demolished by a single-core laptop attack in 2022. Saw Rainbow's multivariate scheme collapse under the weight of its own key sizes. And now, finally, we have actual standards to implement.

But here's the thing—most coverage of FIPS 203/204/205 is either "quantum computers will break everything" panic or vendor press releases. Neither helps you understand what's actually happening mathematically, why these specific constructions survived years of cryptanalysis, or how to implement them without introducing side-channel vulnerabilities that make the whole exercise pointless.

So let's fix that.

---

## The Mathematical Foundations You Actually Need

All three lattice-based standards (ML-KEM and ML-DSA) rely on variants of the same underlying hard problem: Learning With Errors. The hash-based SLH-DSA is different—I'll get to that. But first, LWE.

### Learning With Errors: Why Lattices?

The basic LWE problem looks deceptively simple. You have a secret vector **s** in Z_q^n. An adversary sees pairs (a_i, b_i) where:

```
b_i = ⟨a_i, s⟩ + e_i (mod q)
```

The a_i vectors are uniformly random. The e_i terms are "errors"—small values sampled from some distribution (typically discrete Gaussian or centered binomial). The adversary's job: recover **s**.

Without the error terms, this is trivial linear algebra. Gaussian elimination solves it in polynomial time. The errors are what make it hard. And "hard" here means reducible to worst-case lattice problems that have resisted algorithmic improvement for decades, including from quantum algorithms.

The reduction goes through the Shortest Vector Problem (SVP). Regev's 2005 proof showed that if you can solve average-case LWE, you can solve worst-case SVP in any lattice. Grover's algorithm gives quantum computers a quadratic speedup on unstructured search, but lattice problems don't have that structure. The best known quantum algorithms for SVP are only marginally better than classical ones.

This is why lattice cryptography survived NIST's gauntlet while RSA and ECC won't survive actual quantum computers.

### Module-LWE: The Practical Variant

Raw LWE has a problem: key sizes. For 128-bit security, you'd need matrices with thousands of entries. ML-KEM and ML-DSA use Module-LWE instead, which structures the problem over polynomial rings.

The ring R_q = Z_q[X]/(X^n + 1) gives you elements that are polynomials of degree less than n with coefficients mod q. Module-LWE works over R_q^k—vectors of k ring elements.

Key generation for ML-KEM looks like this:

```
KeyGen():
  ρ, σ ← {0,1}^256
  A ← Sample(ρ) ∈ R_q^(k×k)
  s, e ← β_η(σ) ∈ R_q^k
  t := As + e
  return (pk := (t, ρ), sk := s)
```

The matrix **A** is pseudorandomly generated from seed ρ—this is important for key size. Rather than transmitting A directly (which would be huge), you just send the seed. The secret **s** and error **e** come from a centered binomial distribution with parameter η.

What makes this secure? An adversary seeing (A, t = As + e) can't distinguish it from (A, u) where u is uniformly random. That's the decisional MLWE assumption. Breaking it would require either solving the underlying lattice problem or finding some structural weakness in the ring R_q.

The X^n + 1 polynomial matters here. Earlier schemes used other rings and got burned—NTRU Prime exists specifically because the original NTRU ring had concerning structure. The cyclotomic polynomial X^n + 1 where n is a power of 2 has been extensively analyzed. No exploitable structure has been found, and there's strong theoretical evidence none exists.

---

## FIPS 203: ML-KEM in Detail

ML-KEM (formerly CRYSTALS-Kyber) is a key encapsulation mechanism. It doesn't directly encrypt data—it establishes a shared secret that you then use with a symmetric cipher. This is how TLS works anyway, so the interface is natural.

### Parameter Sets and Security Levels

NIST defines three parameter sets:

| Variant | k | η₁ | η₂ | d_u | d_v | Security | Public Key | Ciphertext |
|---------|---|----|----|-----|-----|----------|------------|------------|
| ML-KEM-512 | 2 | 3 | 2 | 10 | 4 | Level 1 (AES-128) | 800 bytes | 768 bytes |
| ML-KEM-768 | 3 | 2 | 2 | 10 | 4 | Level 3 (AES-192) | 1,184 bytes | 1,088 bytes |
| ML-KEM-1024 | 4 | 2 | 2 | 11 | 5 | Level 5 (AES-256) | 1,568 bytes | 1,568 bytes |

The k parameter is the module rank—how many ring elements in your vectors. Higher k means more lattice dimension, which means harder problems but bigger keys.

η₁ and η₂ control the error distribution during key generation and encapsulation. Smaller errors are easier to decode but might leak more about the secret. The centered binomial distribution with parameter η samples values in [-η, η] with binomial probabilities.

d_u and d_v are compression parameters. This is something people miss about ML-KEM: the ciphertext is lossy compressed. You don't send the full ring elements—you round them to fewer bits. This introduces decryption failures at some rate, but the parameters are chosen to make failures negligibly rare (< 2^-140).

### Why ML-KEM-768 is the Sweet Spot

I'll be direct: unless you have specific compliance requirements, use ML-KEM-768.

ML-KEM-512's security margin makes me uncomfortable. The concrete security estimates put it around 118 bits against known quantum attacks—technically above 128, but with less margin than I'd like given potential algorithmic improvements. The 2022 improvements to lattice sieving algorithms weren't predicted by many.

ML-KEM-1024 is overkill for most applications. You're adding ~400 bytes to every handshake for security that won't matter unless we see dramatic, unexpected breaks in lattice cryptography. If that happens, we have bigger problems than your choice of parameter set.

ML-KEM-768 hits 183-bit classical security and ~161-bit quantum security against known attacks. The key sizes are manageable—1,184 bytes for a public key is larger than ECC but not dramatically larger than RSA-2048.

### The Fujisaki-Okamoto Transform

Raw lattice encryption (like raw ElGamal) isn't CCA-secure. An adversary who can submit ciphertexts for decryption can extract the secret key. This is a problem because protocols like TLS let attackers see whether decryption succeeded.

ML-KEM applies the Fujisaki-Okamoto transform to get IND-CCA2 security. The transform works like this:

1. Encrypt a random message m
2. Derive the symmetric key by hashing m
3. Include a hash of m in the ciphertext
4. On decryption, re-encrypt to verify the ciphertext is well-formed
5. If verification fails, return a pseudorandom key derived from the secret key and ciphertext

Step 5 is crucial. Returning a deterministic error would let attackers probe. Instead, they get a random-looking key that reveals nothing. The verification in step 4 prevents chosen-ciphertext attacks.

Implementation-wise, this means decapsulation is about twice as expensive as encapsulation—you're running the encapsulation routine inside decapsulation to verify. In practice, this is fine. We're talking 40,000 cycles versus 30,000 cycles on modern hardware.

---

## FIPS 204: ML-DSA for Digital Signatures

ML-DSA (formerly CRYSTALS-Dilithium) handles digital signatures. Same lattice foundations, completely different construction.

### The Fiat-Shamir with Aborts Paradigm

Signature schemes from identification protocols use the Fiat-Shamir transform: make a commitment, receive a random challenge, produce a response. ML-DSA does this, but with a twist—sometimes the signing algorithm aborts and restarts.

Here's why. The signature reveals z = y + cs where y is random masking, c is the challenge, and s is the secret. If |z| is too large, it might leak information about s through the distribution of outputs. So we reject and try again whenever z falls outside an acceptable range.

The rejection sampling introduces a timing side-channel in naive implementations. The number of iterations varies based on the secret key. Good implementations run constant iterations and select the valid output, or use more sophisticated constant-time rejection sampling.

```python
def sign(message, secret_key):
    # Unpack secret key
    seed, s1, s2, t0 = unpack_sk(secret_key)
    A = expand_a(seed)
    mu = sha3_256(message).digest()

    kappa = 0
    while True:
        y = sample_uniform_gamma1(seed, kappa)
        w = matrix_vector_multiply(A, y)
        w1 = highbits(w)

        c_tilde = sha3_256(mu + pack_w1(w1)).digest()
        c = sample_challenge(c_tilde)

        z = vector_add(y, scalar_multiply(c, s1))

        # Rejection sampling - this is the "abort"
        if not check_bounds(z, gamma1 - beta):
            kappa += 1
            continue

        h = make_hint(w, vector_add(w, scalar_multiply(c, s2)))
        if count_ones(h) > omega:
            kappa += 1
            continue

        return pack_signature(c_tilde, z, h)
```

### Signature Sizes: The Uncomfortable Truth

This is where lattice signatures hurt. ML-DSA-65 (the Level 3 variant) produces 3,293-byte signatures. For comparison, ECDSA P-256 signatures are 64 bytes. Ed25519 signatures are 64 bytes.

That's a 50x increase.

For bulk signing operations, this probably doesn't matter. For constrained environments, bandwidth-limited protocols, or applications signing millions of items, it matters a lot.

The parameter sets:

| Variant | Security Level | Public Key | Secret Key | Signature |
|---------|---------------|------------|------------|-----------|
| ML-DSA-44 | Level 2 | 1,312 bytes | 2,528 bytes | 2,420 bytes |
| ML-DSA-65 | Level 3 | 1,952 bytes | 4,000 bytes | 3,293 bytes |
| ML-DSA-87 | Level 5 | 2,592 bytes | 4,864 bytes | 4,595 bytes |

There's no Level 1 variant. NIST decided the security margins were too thin. I agree with this decision.

### Performance Characteristics

ML-DSA signing is slower than ECDSA—roughly 10-100x depending on the implementation and hardware. Verification is comparable to ECDSA, sometimes faster.

On a modern x86_64 processor with AVX2:
- Key generation: ~100,000 cycles
- Signing: ~200,000-300,000 cycles (varies due to rejection)
- Verification: ~100,000 cycles

Compare to Ed25519:
- Key generation: ~60,000 cycles
- Signing: ~80,000 cycles
- Verification: ~200,000 cycles

So ML-DSA verification is actually competitive. Signing is the bottleneck, and the variance from rejection sampling complicates benchmarking.

---

## FIPS 205: SLH-DSA and the Hash-Based Alternative

SLH-DSA (formerly SPHINCS+) takes a completely different approach. No lattices. No number-theoretic assumptions. Just hash functions.

The security assumption: SHA-256 (or SHAKE256) is a secure hash function. That's it. If an adversary can break SLH-DSA, they've found collisions, preimages, or some other weakness in SHA-2 or SHA-3.

This is simultaneously reassuring and terrifying. Reassuring because we've studied hash functions for decades and trust them deeply. Terrifying because if SHA-256 breaks, we have catastrophic problems far beyond signatures.

### The SPHINCS+ Construction

SLH-DSA uses a hypertree of many-time signature schemes, each built from one-time signatures (specifically, WOTS+). The one-time signatures can only sign a single message safely. The tree structure lets you sign many messages by using each one-time key exactly once.

The signature includes an authentication path through the tree. This path proves that a particular one-time public key is part of the larger public key structure.

### The Size Problem

SLH-DSA signatures are huge:

| Variant | Public Key | Signature |
|---------|------------|-----------|
| SLH-DSA-128s | 32 bytes | 7,856 bytes |
| SLH-DSA-128f | 32 bytes | 17,088 bytes |
| SLH-DSA-192s | 48 bytes | 16,224 bytes |
| SLH-DSA-192f | 48 bytes | 35,664 bytes |
| SLH-DSA-256s | 64 bytes | 29,792 bytes |
| SLH-DSA-256f | 64 bytes | 49,856 bytes |

The "s" variants are "small" (smaller signatures, slower signing). The "f" variants are "fast" (faster signing, larger signatures). Pick your poison.

7KB for the smallest secure signature. 50KB for the largest. This is... a lot.

### When to Use SLH-DSA

Honestly? As a backup.

ML-DSA is your primary post-quantum signature scheme. But if you're doing something like signing firmware that will exist for 20 years, having an SLH-DSA signature alongside gives you insurance. If lattices somehow break, your hash-based signatures remain valid.

> The NSA's CNSA 2.0 guidance requires SLH-DSA for software and firmware signing in National Security Systems. They're not betting everything on lattices.
{: .prompt-info }

---

## Implementation: Where Theory Meets Reality

The cryptographic theory is elegant. Implementation is where things get messy.

### Side-Channel Resistance

Every operation touching secret data must be constant-time. For lattice schemes, this includes:
- Polynomial arithmetic with secret coefficients
- Centered binomial sampling
- NTT (Number Theoretic Transform) butterflies
- Comparison operations in rejection sampling
- Error decoding during decapsulation

The NTT deserves special attention. It's essentially an FFT over a finite field, and it's where most of ML-KEM's computation happens. Naive implementations branch on coefficient values. Good implementations use constant-time conditional swaps.

```c
// BAD: Timing depends on coefficients
if (a[i] >= q) a[i] -= q;

// GOOD: Constant-time conditional subtraction
int32_t mask = (a[i] - q) >> 31;  // All 1s if a[i] < q, all 0s otherwise
a[i] = (a[i] & mask) | ((a[i] - q) & ~mask);
```

For ML-DSA, the rejection sampling loop is a constant-time nightmare. You can't just return when you find a valid signature—that leaks iteration count. Instead, you:
1. Always run the maximum expected iterations
2. Accumulate valid signatures
3. Return the first valid one using constant-time selection

### Library Options

As of late 2024, your options are:

**LibOQS (Open Quantum Safe)**: Reference implementations of everything. Not optimized for production but well-tested. Good for prototyping.

**PQClean**: Clean, portable implementations. Focused on readability and correctness over speed.

**liboqs-rust / pqcrypto crate**: Rust bindings. If you're in Rust, this is your path.

**OpenSSL 3.5+**: Native ML-KEM support coming. This is probably where most deployments will land.

**BoringSSL**: Google's fork has experimental PQC support. Chrome uses this for the X25519+ML-KEM-768 hybrid.

### Real Deployments Already Happening

Chrome 124+ negotiates X25519MLKEM768 automatically. I've seen it in wireshark captures. The hybrid construction concatenates an X25519 shared secret with an ML-KEM-768 shared secret, then derives the TLS keys from both.

Cloudflare's been running ML-KEM in production for over a year. Their data shows about 1.6KB additional handshake overhead and 15.5ms additional latency on cold connections. On persistent connections with session resumption, the impact is negligible.

Signal's PQXDH protocol combines X25519 with Kyber-1024 (pre-standardization ML-KEM-1024). They went aggressive on parameters because their threat model includes nation-state adversaries harvesting traffic now for decryption later.

AWS KMS supports ML-KEM key encapsulation. You can generate post-quantum KEKs today.

---

## Migration Strategy: What to Do Right Now

If you're reading this in 2024-2025, here's the pragmatic path:

**Phase 1 (Now)**: Inventory your cryptographic dependencies. Everything using RSA, DH, ECDH, or ECDSA will eventually need replacement. You need to know where those calls are.

**Phase 2 (2025)**: Deploy hybrid schemes. X25519+ML-KEM-768 for key exchange. ECDSA+ML-DSA-65 for signatures where you can tolerate the size. Security comes from both—if either remains secure, you're protected.

**Phase 3 (2026-2027)**: Begin migrating high-value, long-lived keys to pure post-quantum. CA root keys, firmware signing keys, anything with a 10+ year lifespan.

**Phase 4 (2028+)**: Accelerate based on quantum computing progress. IBM's roadmap shows 200 fault-tolerant logical qubits by 2029. IonQ claims cryptographically-relevant machines by 2028-2033. These timelines are aggressive but not impossible.

### The Harvest Now, Decrypt Later Problem

This is why migration is urgent even though large-scale quantum computers don't exist yet.

Adversaries with resources—and we know who we're talking about—are capturing encrypted traffic today. TLS session keys are typically ECDH-derived. When a sufficiently large quantum computer exists, they can derive those session keys and decrypt the stored traffic.

> If your traffic has a 20-year sensitivity window, and quantum computers might exist in 10 years, you needed post-quantum encryption 10 years ago. Since time travel isn't available, start today.
{: .prompt-danger }

---

## What I'm Actually Worried About

My concern isn't the mathematics. Lattice cryptography has solid theoretical foundations and survived NIST's brutal selection process.

I'm worried about implementation quality.

The OpenSSL Heartbleed bug wasn't a cryptographic flaw—it was a bounds-checking error. The ROCA vulnerability in Infineon TPMs wasn't a mathematical break—it was a bad random number generator. EFAIL wasn't about AES—it was about how email clients handled decryption failures.

Post-quantum implementations are new. They haven't had the same adversarial attention as OpenSSL's RSA code. The constant-time requirements are harder to verify. The side-channel attack surface is larger because the algorithms do more complex things with secret data.

The next few years will be a period of discovery. We'll find implementation bugs. Some will be severe. This is normal for new cryptographic deployments. The alternative—waiting for "mature" implementations while nation-states harvest your traffic—is worse.

Use hybrid schemes. Deploy post-quantum where you can. Accept that you'll be doing updates. That's the reality of cryptographic transition.

---

## Technical Reference

For a comprehensive parameter reference covering all NIST PQC standards, see our [**PQC Technical Reference**](/assets/resources/pqc-reference.html)—a single-page summary of ML-KEM, ML-DSA, and SLH-DSA parameters, performance benchmarks, and implementation considerations.

---

*If you want to play with this yourself, LibOQS has reasonable documentation at [openquantumsafe.org](https://openquantumsafe.org). The NIST standards documents are dense but precise—read FIPS 203 if you want to understand exactly what "IND-CCA2" security means in this context. And if you're implementing from scratch: don't. Use a library. I mean it.*
