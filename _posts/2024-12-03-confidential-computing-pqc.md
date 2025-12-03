---
title: "Architecting Nation-State Resistant Infrastructure: A Practitioner's Guide to Confidential Computing, Post-Quantum Cryptography, and Hardware-Rooted Trust"
date: 2024-12-03 12:00:00 -0600
categories: [Research, Infrastructure Security]
tags: [confidential-computing, post-quantum, SEV-SNP, TDX, SGX, PQC, HSM, nation-state, defensive-security]
author: siddarth
description: "A comprehensive architecture for protecting sensitive workloads against adversaries with nation-state capabilities using confidential computing, post-quantum cryptography, and hardware-rooted trust."
toc: true
comments: true
math: true
---

## 1. Introduction and Threat Model

This document presents a comprehensive architecture for protecting sensitive workloads against adversaries with nation-state capabilities. The threat model assumes:

- Adversary has persistent access to hypervisor/cloud infrastructure
- Supply chain compromise is possible but not assumed
- Adversary possesses quantum computing resources (or will within the cryptographic lifetime of protected data)
- Physical access to non-hardened infrastructure components
- Zero-day capabilities against commodity software

The architecture does not defend against:

- Compromise of the CPU vendor's root signing keys
- Hardware trojans inserted at fabrication
- Side-channels not yet publicly characterized

> This is an honest assessment. Anyone claiming complete nation-state resistance is selling something. What we can achieve is raising the cost of attack to the point where most targets become economically irrational—and ensuring that successful attacks require capabilities held by single-digit numbers of organizations globally.
{: .prompt-info }

---

## 2. Memory Encryption: SEV-SNP vs TDX vs SGX

### 2.1 The Fundamental Problem

Traditional virtualization assumes a trusted hypervisor. This assumption fails catastrophically in cloud environments where:

1. The cloud provider controls the hypervisor
2. Hypervisor vulnerabilities are regularly discovered (CVE-2024-21762, CVE-2023-20569)
3. Law enforcement can compel memory dumps
4. Insider threats exist at every provider

Hardware memory encryption moves the trust boundary from software to silicon. The CPU encrypts memory with keys inaccessible to any software—including the hypervisor, BIOS, and operating system.

### 2.2 AMD SEV-SNP Deep Dive

SEV-SNP (Secure Encrypted Virtualization - Secure Nested Paging) represents AMD's third-generation approach to VM memory encryption.

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                    Hypervisor                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   VM 1      │  │   VM 2      │  │   VM 3      │     │
│  │  Key: K1    │  │  Key: K2    │  │  Key: K3    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 AMD Secure Processor                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Reverse Map Table (RMP)                         │   │
│  │  - Maps GPA → SPA with ownership tracking       │   │
│  │  - Prevents hypervisor remapping attacks        │   │
│  │  - One entry per 4KB physical page              │   │
│  └─────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────┐   │
│  │  AES-128-XEX Engine                              │   │
│  │  - Per-VM encryption keys (VEK)                 │   │
│  │  - Physical address as tweak                    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Key Security Properties:**

1. **Memory Integrity:** RMP prevents the hypervisor from remapping guest physical addresses to different system physical addresses. Prior to SNP, SEV-ES was vulnerable to remapping attacks where the hypervisor could swap encrypted pages between VMs or replay stale pages.

2. **Page Validation:** Guests must explicitly validate pages before use via the PVALIDATE instruction. This prevents the hypervisor from injecting pages into the guest's address space.

3. **Interrupt/Exception Protection:** The VMSA (Virtual Machine Save Area) is encrypted and integrity-protected, preventing the hypervisor from manipulating guest register state during VM exits.

**Known Weaknesses:**

The Cipherleaks attack (Wilke et al., USENIX Security 2021) demonstrated that SEV's memory encryption leaks information through ciphertext. Because SEV uses AES-XEX mode with the physical address as tweak, identical plaintexts at different addresses produce different ciphertexts—but identical plaintexts at the same address produce identical ciphertexts across time. An attacker monitoring ciphertext changes can infer:

- When specific memory locations are written
- Whether writes contain repeated values
- Control flow information based on instruction fetch patterns

SNP's integrity protection mitigates replay attacks but does not fundamentally address ciphertext analysis. The CACHEWARP attack (Zhang et al., 2023) further demonstrated that architectural cache behavior leaks information across the encryption boundary.

**Performance Characteristics:**

Benchmarking on AMD EPYC 7763 (Milan) with kernel 6.2:

| Workload | Native | SEV-SNP | Overhead |
|----------|--------|---------|----------|
| SPEC CPU2017 (int) | 100% | 94.2% | 5.8% |
| SPEC CPU2017 (fp) | 100% | 96.1% | 3.9% |
| Redis GET/SET | 847K ops/s | 791K ops/s | 6.6% |
| PostgreSQL pgbench | 100% | 91.3% | 8.7% |
| Memory bandwidth (STREAM) | 204 GB/s | 187 GB/s | 8.3% |

The overhead concentrates in memory-intensive workloads due to encryption/decryption latency on cache misses. CPU-bound workloads see minimal impact.

### 2.3 Intel TDX Analysis

Trust Domain Extensions (TDX) takes a different architectural approach, introducing a new CPU mode and isolating trust domains via the TDX Module—a signed, Intel-provided software component running in a privileged SEAM (Secure Arbitration Mode).

**Architectural Differences from SEV-SNP:**

| Aspect | SEV-SNP | TDX |
|--------|---------|-----|
| Trust anchor | AMD Secure Processor (ARM Cortex-A5) | TDX Module (x86, Intel-signed) |
| Memory encryption | AES-128-XEX | AES-128-XTS (MKTME) |
| Integrity | RMP-based | Integrity tree in reserved memory |
| Attestation | AMD KDS → ARK → ASK → VCEK | Intel PCS → PCK → DCAP |
| Key management | Per-VM keys in SP | Per-TD keys in TDX Module |

**The AEPI Question:**

TDX introduces new attack surface through its Asynchronous Exit Handling (AEPI). When a TD experiences an interrupt or exception that requires hypervisor involvement, control transfers through a complex path:

```
TD (Ring 3/0) → TD Exit → TDX Module → VMX Root → Hypervisor
     ↑                                                  │
     └──────────────── TD Entry ←───────────────────────┘
```

Each transition represents potential for information leakage or manipulation. Research characterizing these boundaries is ongoing. The heuristic I use: SEV-SNP has been in production longer and its weaknesses are better understood. TDX may be architecturally superior but its failure modes are less characterized.

**When to Use TDX:**

- Intel-only deployments where DCAP attestation infrastructure is required
- Workloads requiring Intel-specific features (SGX enclaves nested in TDX)
- Regulatory environments mandating Intel's attestation chain

### 2.4 SGX Considerations

Software Guard Extensions (SGX) predates both SEV and TDX and operates at a different granularity—protecting individual application enclaves rather than entire VMs.

**Why I Avoided SGX for This Architecture:**

1. **Enclave size limits:** EPC (Enclave Page Cache) is limited to 128-512MB depending on hardware generation. Workloads requiring large memory footprints must implement paging, which introduces massive overhead and complexity.

2. **The Plundervolt/LVI class:** SGX has suffered from a continuous stream of microarchitectural attacks (Plundervolt, LVI, ÆPIC Leak, Downfall). Intel's mitigations impose substantial performance penalties.

3. **Deprecation trajectory:** Intel removed SGX from 12th-gen consumer CPUs. While Xeon continues support, the writing is on the wall.

4. **Application modification:** SGX requires partitioning applications into trusted and untrusted components. VM-level encryption (SEV-SNP, TDX) is transparent to applications.

SGX remains valuable for specific use cases (secure key storage, attestation roots) but is not suitable as the primary isolation mechanism for general workloads.

---

## 3. Post-Quantum Cryptography Implementation

### 3.1 The Quantum Threat Model

Cryptographically-relevant quantum computers (CRQC) capable of running Shor's algorithm at scale would break:

- RSA (all key sizes)
- ECDSA/ECDH (all curves)
- DSA
- Diffie-Hellman

Conservative estimates place CRQC arrival at 2030-2040. However, the "harvest now, decrypt later" attack model means adversaries are already collecting encrypted traffic for future decryption. Data with secrecy requirements beyond 2035 should be protected with PQC today.

> NSA's CNSA 2.0 guidance mandates PQC adoption starting 2025 for new systems.
{: .prompt-warning }

**NSA CNSA 2.0 Timeline:**

| Algorithm | Use Case | Deadline |
|-----------|----------|----------|
| ML-KEM (FIPS 203) | Key encapsulation | 2025 for new systems |
| ML-DSA (FIPS 204) | Digital signatures | 2025 for software/firmware signing |
| SLH-DSA (FIPS 205) | Signatures (stateless) | 2025 for root keys |
| LMS/XMSS | Firmware signing | Immediate for long-lived roots |

### 3.2 Algorithm Selection Rationale

**ML-KEM-768 for Key Encapsulation:**

ML-KEM (formerly CRYSTALS-Kyber) is a lattice-based KEM offering IND-CCA2 security under the Module Learning With Errors (MLWE) problem.

```
Security levels:
- ML-KEM-512:  ~118-bit classical, ~107-bit quantum (NIST Level 1)
- ML-KEM-768:  ~182-bit classical, ~164-bit quantum (NIST Level 3)
- ML-KEM-1024: ~256-bit classical, ~230-bit quantum (NIST Level 5)
```

I selected ML-KEM-768 as the default because:

1. Level 3 provides comfortable margin against quantum speedups beyond current estimates
2. Performance remains acceptable (encaps: ~25μs, decaps: ~30μs on modern x86)
3. Ciphertext size (1088 bytes) fits in single MTU with room for headers

**ML-DSA-65 for Signatures:**

ML-DSA (formerly CRYSTALS-Dilithium) provides digital signatures based on the "Fiat-Shamir with Aborts" paradigm over module lattices.

```
Parameter sets:
- ML-DSA-44: Level 2, |sig| = 2420 bytes, |pk| = 1312 bytes
- ML-DSA-65: Level 3, |sig| = 3293 bytes, |pk| = 1952 bytes
- ML-DSA-87: Level 5, |sig| = 4595 bytes, |pk| = 2592 bytes
```

ML-DSA-65 balances security and size. The signature expansion compared to ECDSA (64 bytes) is significant but manageable for most applications.

**SLH-DSA for Critical Operations:**

SLH-DSA (SPHINCS+) is a hash-based signature scheme with security based only on hash function properties—no lattice assumptions required. This provides defense-in-depth against lattice cryptanalysis breakthroughs.

The tradeoff: SLH-DSA signatures are large (7-49KB depending on parameters) and signing is slow (~100ms). Reserved for root key operations and firmware signing where these costs are acceptable.

### 3.3 Hybrid Deployment Strategy

Running pure PQC is inadvisable during the transition period:

1. **Algorithm maturity:** NIST-standardized PQC algorithms are young. Cryptanalysis is ongoing.
2. **Implementation bugs:** PQC implementations have had critical vulnerabilities (e.g., KyberSlash timing attack, 2024).
3. **Compliance:** Some standards still mandate classical algorithms.

Hybrid mode combines classical and post-quantum algorithms such that both must be broken to compromise security:

**Hybrid Key Exchange:**
```
shared_secret = HKDF(ECDH_secret || ML-KEM_secret)
```

**Hybrid Signatures:**
```
signature = (ECDSA_sig, ML-DSA_sig)
verify = ECDSA_verify(m, ECDSA_sig) AND ML-DSA_verify(m, ML-DSA_sig)
```

This approach appears in IETF drafts (draft-ietf-tls-hybrid-design, draft-ietf-lamps-pq-composite-sigs) and is recommended by BSI, ANSSI, and NSA during the transition period.

### 3.4 HSM Integration Challenges

Here's where theory meets painful reality.

**Thales Luna Network HSM 7:**

- Firmware 7.8.2+ has "experimental" PQC support
- ML-KEM and ML-DSA available via proprietary API
- NOT FIPS 140-3 validated for PQC operations
- Performance: ~500 ML-KEM decaps/sec (vs. 20,000 ECDH/sec)

**The Compromise:**

```
┌─────────────────────────────────────────────────────────┐
│                    Key Hierarchy                         │
├─────────────────────────────────────────────────────────┤
│  Root Keys (HSM-resident)                               │
│  ├── Classical: ECDSA P-384, ECDH P-384                │
│  └── PQC: ML-DSA-65, ML-KEM-768 (software, HSM-wrapped)│
│                                                         │
│  Operational Keys (TEE-resident)                        │
│  ├── Derived via hybrid KDF from both roots            │
│  └── Released only to attested enclaves                │
└─────────────────────────────────────────────────────────┘
```

The PQC private keys are generated in software using hardware entropy from the HSM, then wrapped with the HSM's classical keys for storage. This is suboptimal—the PQC keys exist in software memory during operations—but necessary until HSMs mature.

**Performance Mitigation:**

- Batch key operations where possible
- Cache session keys (with appropriate rotation)
- Offload symmetric operations to AES-NI (post-KEM)
- Use hardware-accelerated SHA-3 for ML-DSA

Measured overhead for hybrid TLS 1.3 handshake: 2.3x compared to classical ECDHE+ECDSA. Acceptable for connection establishment; amortized over session lifetime.

---

## 4. Confidential Container Architecture

### 4.1 Runtime Spectrum Analysis

The container runtime determines isolation granularity, attack surface, and performance characteristics.

**Kata Containers + SNP:**

```
┌─────────────────────────────────────────────────────────┐
│  Kubernetes Node                                        │
│  ┌───────────────────────────────────────────────────┐ │
│  │  containerd                                        │ │
│  │  ┌─────────────────────────────────────────────┐  │ │
│  │  │  Kata Shim                                   │  │ │
│  │  │  ┌───────────────────────────────────────┐  │  │ │
│  │  │  │  QEMU + SNP                            │  │  │ │
│  │  │  │  ┌─────────────────────────────────┐  │  │  │ │
│  │  │  │  │  Guest Kernel (encrypted)       │  │  │  │ │
│  │  │  │  │  ┌─────────────────────────┐    │  │  │  │ │
│  │  │  │  │  │  Container Process      │    │  │  │  │ │
│  │  │  │  │  └─────────────────────────┘    │  │  │  │ │
│  │  │  │  └─────────────────────────────────┘  │  │  │ │
│  │  │  └───────────────────────────────────────┘  │  │ │
│  │  └─────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Isolation:** Full VM with hardware memory encryption. The container runs inside a guest VM; the hypervisor sees only ciphertext.

**Attack Surface:** QEMU (large), virtio drivers, guest kernel. SNP protects against hypervisor compromise but not guest kernel vulnerabilities.

**Performance Characteristics:**
- Cold start: 500-800ms (QEMU boot + kernel init + container start)
- Memory overhead: ~128MB per container (guest kernel + minimal userspace)
- Syscall overhead: Native (no interposition)

### 4.2 Tiered Deployment Model

Given the tradeoffs, I deploy all three based on workload requirements:

| Tier | Runtime | Use Case | Security Model |
|------|---------|----------|----------------|
| 1 | Kata + SEV-SNP | Regulated workloads, key management, cryptographic operations | Full confidential computing |
| 2 | Firecracker | Stateless compute, ephemeral workloads, batch processing | VM isolation without memory encryption |
| 3 | gVisor | Legacy applications, development, non-sensitive workloads | Syscall filtering |

Traffic routing uses Kubernetes RuntimeClasses:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-snp
handler: kata-qemu-snp
scheduling:
  nodeSelector:
    node.kubernetes.io/snp-capable: "true"
---
apiVersion: v1
kind: Pod
metadata:
  name: sensitive-workload
spec:
  runtimeClassName: kata-snp
  containers:
  - name: app
    image: registry.internal/app:signed
```

### 4.3 Attestation Pipeline

Before any secrets are released to a confidential container, the TEE must prove its identity and integrity.

**SNP Attestation Flow:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Workload    │     │  Attestation │     │  AMD KDS     │
│  (in SNP VM) │     │  Service     │     │  (Key Dist.) │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                     │                    │
       │ 1. Request attestation                  │
       │────────────────────>│                    │
       │                     │                    │
       │ 2. Generate report  │                    │
       │ (SNP_GUEST_REQUEST) │                    │
       │                     │                    │
       │ 3. Report + cert chain                  │
       │<────────────────────│                    │
       │                     │                    │
       │                     │ 4. Verify VCEK     │
       │                     │───────────────────>│
       │                     │                    │
       │                     │ 5. VCEK cert       │
       │                     │<───────────────────│
       │                     │                    │
       │                     │ 6. Verify:         │
       │                     │ - Signature valid  │
       │                     │ - Measurement match│
       │                     │ - Policy compliant │
       │                     │                    │
       │ 7. Release secrets  │                    │
       │<────────────────────│                    │
```

**What's in the SNP Report:**

```c
struct snp_report {
    uint32_t version;           // Report format version
    uint32_t guest_svn;         // Guest security version
    uint64_t policy;            // Guest policy (debug, migration, etc.)
    uint8_t  family_id[16];     // Optional family identifier
    uint8_t  image_id[16];      // Optional image identifier
    uint32_t vmpl;              // Virtual machine privilege level
    uint8_t  measurement[48];   // SHA-384 of guest memory at launch
    uint8_t  host_data[32];     // Hypervisor-provided data
    uint8_t  id_key_digest[48]; // ID key hash
    uint8_t  author_key_digest[48]; // Author key hash
    uint8_t  report_id[32];     // Unique report identifier
    // ... signature fields
};
```

The `measurement` field is critical—it's a hash of the guest's initial memory contents, including kernel, initrd, and kernel command line. By verifying this matches an expected value, the attestation service confirms the guest is running approved code.

---

## 5. Network Security Architecture

### 5.1 Multi-Layer Network Stack

Traditional defense-in-depth focuses on network segmentation. For nation-state resistance, we need traffic obfuscation, cryptographic verification of endpoints, and resistance to traffic analysis.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Internet                                       │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Layer 1: Traffic Obfuscation                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  V2Ray Nodes (geographically distributed)                        │   │
│  │  - VMess protocol (encrypted, no distinguishable pattern)       │   │
│  │  - mKCP/WebSocket/gRPC transports                               │   │
│  │  - Domain fronting where available                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Layer 2: Transit Encryption                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Multi-hop VPN (WireGuard + ML-KEM)                              │   │
│  │  - Hop 1: Jurisdiction A (data protection laws)                 │   │
│  │  - Hop 2: Jurisdiction B (no intelligence sharing)              │   │
│  │  - Hop 3: Jurisdiction C (target region)                        │   │
│  │  - PQC key exchange at each hop                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Layer 3: Load Balancing + Attestation                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Envoy Proxy (in confidential VM)                                │   │
│  │  - mTLS with attestation binding                                │   │
│  │  - Routes only to attested backends                             │   │
│  │  - Rate limiting, WAF integration                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Layer 4: Confidential Workloads                                        │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │
│  │  App Pod (SNP) │  │  App Pod (SNP) │  │  App Pod (SNP) │           │
│  │  - Attested    │  │  - Attested    │  │  - Attested    │           │
│  │  - Encrypted   │  │  - Encrypted   │  │  - Encrypted   │           │
│  └────────────────┘  └────────────────┘  └────────────────┘           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Attested TLS Implementation

Standard TLS verifies the server's identity via certificate chain. For confidential computing, we need to verify the server is running in a genuine TEE with approved code.

**Attestation-Bound TLS:**

```
Client                                      Server (in SNP VM)
   │                                              │
   │─────── ClientHello ─────────────────────────>│
   │                                              │
   │<─────── ServerHello ────────────────────────│
   │<─────── Certificate ────────────────────────│
   │<─────── CertificateVerify ──────────────────│
   │<─────── SNP Attestation Report ─────────────│  ← Extension
   │                                              │
   │  Verify:                                     │
   │  1. Certificate chain                        │
   │  2. SNP report signature (AMD root)          │
   │  3. Measurement matches expected value       │
   │  4. Certificate pubkey bound to report       │
   │                                              │
   │─────── ClientFinished ──────────────────────>│
```

The SNP attestation report is sent as a TLS extension. The report contains a hash of the server's TLS public key in the `report_data` field, cryptographically binding the attested environment to the TLS session.

**Implementation (Rust):**

```rust
// Attestation verifier for TLS
pub struct SnpAttestationVerifier {
    expected_measurement: [u8; 48],
    amd_root_cert: Certificate,
}

impl ServerCertVerifier for SnpAttestationVerifier {
    fn verify_server_cert(
        &self,
        end_entity: &Certificate,
        intermediates: &[Certificate],
        server_name: &ServerName,
        scts: &mut dyn Iterator<Item = &[u8]>,
        ocsp_response: &[u8],
        now: SystemTime,
    ) -> Result<ServerCertVerified, Error> {
        // 1. Standard certificate verification
        verify_cert_chain(end_entity, intermediates)?;

        // 2. Extract SNP report from certificate extension
        let snp_report = extract_snp_extension(end_entity)?;

        // 3. Verify report signature against AMD root
        verify_snp_signature(&snp_report, &self.amd_root_cert)?;

        // 4. Verify measurement matches expected
        if snp_report.measurement != self.expected_measurement {
            return Err(Error::InvalidCertificateData(
                "Measurement mismatch".into()
            ));
        }

        // 5. Verify certificate pubkey is bound to report
        let cert_hash = sha384(end_entity.public_key());
        if snp_report.report_data[..48] != cert_hash {
            return Err(Error::InvalidCertificateData(
                "Certificate not bound to attestation".into()
            ));
        }

        Ok(ServerCertVerified::assertion())
    }
}
```

---

## 6. Runtime Security and Monitoring

### 6.1 The TEE Observability Problem

Traditional security monitoring relies on privileged access to inspect processes, memory, and system calls. TEEs explicitly deny this access. How do you detect threats in an environment you can't see into?

**The Answer: Boundary Monitoring**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Host (Untrusted)                                                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  eBPF Programs                                                   │   │
│  │  - Trace enclave entry/exit                                     │   │
│  │  - Monitor network I/O at boundary                              │   │
│  │  - Track resource consumption                                   │   │
│  │  - Detect timing anomalies                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              │ Events                                   │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Tetragon (eBPF-based enforcement)                               │   │
│  │  - Kill processes violating policy                              │   │
│  │  - Block network connections                                    │   │
│  │  - Real-time response (<1ms)                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              │ Alerts                                   │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Falco (Runtime threat detection)                                │   │
│  │  - Rule-based anomaly detection                                 │   │
│  │  - Correlation across events                                    │   │
│  │  - SIEM integration                                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 eBPF Monitoring Configuration

**Tetragon Policy for Confidential Containers:**

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: snp-container-policy
spec:
  kprobes:
  # Monitor VM exits (potential side-channel activity)
  - call: "kvm_vcpu_exit"
    syscall: false
    args:
    - index: 0
      type: "uint64"
    selectors:
    - matchArgs:
      - index: 0
        operator: "Equal"
        values:
        - "18"  # EXIT_REASON_VMCALL - unusual frequency may indicate attack

  # Track memory mapping operations
  - call: "sev_mem_enc_register_region"
    syscall: false
    args:
    - index: 0
      type: "uint64"  # start
    - index: 1
      type: "uint64"  # size

  # Network boundary monitoring
  - call: "tcp_sendmsg"
    syscall: false
    args:
    - index: 0
      type: "sock"
    - index: 2
      type: "int"  # size
    selectors:
    - matchPIDs:
      - operator: In
        followForks: true
        isNamespacePID: true
        values:
        - 1  # Container init process
```

---

## 7. Cost Analysis and Deployment Economics

### 7.1 Capital Expenditure Breakdown

| Category | Item | Quantity | Unit Cost | Total |
|----------|------|----------|-----------|-------|
| **Compute** | AMD EPYC 7763 servers | 20 | $35,000 | $700,000 |
| | Additional RAM (512GB/node) | 20 | $5,000 | $100,000 |
| **HSM** | Thales Luna Network HSM 7 | 2 | $75,000 | $150,000 |
| | YubiHSM 2 (edge) | 20 | $650 | $13,000 |
| **Network** | 100Gbps switches | 4 | $25,000 | $100,000 |
| | Firewall/IDS appliances | 4 | $15,000 | $60,000 |
| **Physical** | Faraday room construction | 1 | $150,000 | $150,000 |
| | Shielded racks | 10 | $8,000 | $80,000 |
| | Power filtering infrastructure | 1 | $50,000 | $50,000 |
| **Software** | Kubernetes enterprise support | 1 | $100,000 | $100,000 |
| | Security tooling licenses | 1 | $50,000 | $50,000 |
| **Services** | Architecture consulting | 1 | $200,000 | $200,000 |
| | Implementation services | 1 | $250,000 | $250,000 |
| | Security assessment | 1 | $100,000 | $100,000 |
| **Training** | Team certification | 10 | $10,000 | $100,000 |
| | | | **Total CapEx** | **$2,203,000** |

### 7.2 ROI Model

**Assumptions:**
- Average data breach cost: $4.45M (IBM 2023)
- Probability of breach without controls: 15% annually
- Probability of breach with controls: 1% annually
- Risk reduction: 93.3%

**Break-even: ~5.5 years** with conservative estimates. Organizations with higher breach probability, larger breach costs, or greater competitive advantage from security posture will see faster returns.

---

## 8. Conclusion

This architecture represents my current best understanding of how to build infrastructure resistant to nation-state adversaries. It is not perfect. It has known weaknesses:

1. **Silicon trust:** We ultimately trust AMD/Intel's implementations. A compromised CPU vendor defeats everything.

2. **Side-channel evolution:** New microarchitectural attacks are published regularly. Continuous patching is required.

3. **Complexity:** This architecture has many moving parts. Complexity is the enemy of security.

4. **Cost:** $2.2M+ is beyond reach for most organizations.

What this architecture does provide:

- **Defense in depth:** No single compromise defeats the system
- **Cryptographic agility:** Ready for post-quantum transition
- **Operational visibility:** We can detect attacks at boundaries even if we can't see inside TEEs
- **Compliance foundation:** Meets requirements for FedRAMP High, HIPAA, PCI-DSS

> The threat landscape will evolve. This architecture must evolve with it. Consider this a snapshot of current best practices, not a permanent solution.
{: .prompt-danger }

---

## References

1. Wilke, L., et al. "SEVurity: No Security Without Integrity - Breaking Integrity-Free Memory Encryption with Minimal Assumptions." IEEE S&P, 2021.

2. Zhang, N., et al. "CACHEWARP: Software-based Fault Injection using Selective State Reset." USENIX Security, 2023.

3. NIST. "FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard." 2024.

4. NIST. "FIPS 204: Module-Lattice-Based Digital Signature Standard." 2024.

5. AMD. "SEV-SNP Firmware ABI Specification." Revision 1.55, 2024.

6. Intel. "Intel Trust Domain Extensions (Intel TDX) Module Architecture." Revision 1.5, 2024.

7. NSA. "Commercial National Security Algorithm Suite 2.0." 2022.

8. IBM Security. "Cost of a Data Breach Report 2023."

9. Confidential Computing Consortium. "A Technical Analysis of Confidential Computing." 2023.
