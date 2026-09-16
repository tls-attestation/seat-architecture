# Multi-Agent/Multi-Modal Attestation Model: Intra- and Post-Handshake Attested TLS 1.3

### Symbolic Modeling for Identity Workload Policy and Evidence Oracle Mitigation

## Overview

This ProVerif model formalizes an attested TLS 1.3 handshake between clients and confidential-computing server VMs (cVMs), layered on top of the standard TLS 1.3 key schedule covering both attestation timing models for intra-handshake and post-handshake attestation. The symbolic model evaluates multi-tenant/cloud deployments, where many physical machines each host many independently-launched guest cVMs or workloads.

---

## Architecture

### Key Hierarchy and Split-Model Attestation

Per `draft-reddy-rats-key-binding-02` §6, attestation is split across two keys and two tokens rather than a single AK signing a quote directly over `pubTSK`:

* **Key Attestation Key (`pubKAK` / `privKAK`)**: A per-cVM signing key generated within the enclave cryptographic boundary.
* **Platform Attestation Token (PAT)**: Signed by hardware `privAK`, certifying `pubKAK`, platform measurements (`dev_statusRef`), and provenance attributes.
* **Key Attestation Token (KAT)**: Signed by `privKAK`, certifying the ephemeral TLS signing key (`pubTSK`).

The PAT and KAT together form the Composite Attestation Token / Conceptual Message Wrapper (`cmw = (pat, kat)`) per RFC 9999 and `draft-ietf-rats-msg-wrap`.

The manifest and endorsement structures implement `draft-ietf-rats-corim-11` CoRIM/CoMID triples:

* `AttestKeyTriple(NULL_ID, pubAK)` for CSP hardware endorsements.
* `RefValTriple(ID_S, dev_statusRef, workloadAttr)` for reference value appraisal.

### Structural Extensions to the Baseline TLS 1.3 Model

The model extends the baseline single-path TLS 1.3 specification (Bhargavan et al.) through five architectural modifications:

* **Consolidated Workload Keying**: Replaces the separate long-term identity key (`pubLTK`) and ephemeral TLS handshake key (`pubEK`) with a unified TLS Signing Key (`pubTSK`). Under Option A, `pubTSK` is self-signed; under Option B, it is certified by a CA. The legacy `LeakedLTK` event is eliminated.
* **Additional External Trust Anchors**: Introduces a Cloud Service Provider (`CSP`, hardware/platform endorser, keys `pubCSP`/`privCSP`) and a Verifier Owner (`VOK`, workload manifest signer, keys `pubVOK`/`privVOK`) as separate principals, accompanied by respective compromise events (`CompromisedCSP`, `UncheckedPolicies`) to model compound authorization architectures.
* **Provisioning and Lifecycle Instrumentation**: Introduces setup-phase events (`CSRSigned`, `ProvisionedTargetEnv`, `EndorsementsIssued`, and `IdentityManifestIssued`). These events establish baseline reachability invariants and support correspondence lemmas verifying Target Environment Engagement.
* **Adversarial Session Seeding**: Enables adversary-controlled inputs (certificates or target launch measurements) to initialize `run_Server` prior to internal key matching, modeling misconfigured server provisioning and malicious tenant dispatching.
* **Hardware Identifier Anonymization**: Adds a Universal Entity ID (`UEID`) attribute to `agent_keys`, which `gen_endorsements` overwrites with `NULL_ID` to model CSP-enforced pseudonymization of physical hardware platform identifiers.

### Identity Anchoring: Option A vs. Option B

Identity appraisal is parameterized via `idOption: branch_option`, evaluated across key generation, manifest issuance, and endpoint execution:

* **Option A — Direct Attestation Anchoring**: Attestation replaces Web PKI entirely. The workload identity is derived directly from target measurements via `dev2id(dev_statusRef)`. The TLS handshake key (`pubTSK`) is self-signed without Web PKI certificates, and the attestation quote serves as the sole proof of authentication and session binding.
* **Option B — Compound Additive Authentication**: Attestation composes additively with standard Web PKI. Identity (`ID_S`) is bound to `pubTSK` through a CA-signed X.509 certificate, while a VOK-signed **Workload Manifest** binds `ID_S` to authorized launch measurements (`dev_statusRef`). Authentication requires concurrent appraisal across both Web PKI and platform endorsement chains.
  * **Compromise Resilience**:
    - If a Web PKI CA is compromised or untrusted DNS redirects traffic, the attacker cannot spoof the instance without presenting measurements that match the Verifier Owner's signed manifest and valid hardware endorsements.
    - If an enclave host is compromised, the attacker cannot impersonate the domain without a CA-signed certificate and VOK-signed manifest.
    - If an Evidence Oracle signs arbitrary attacker key material, the attack fails under Option B without relying on key provenance claims (`ExternalOrUntrusted`), provided the CA, CSP, AK, TSK, and Verifier Owner policy anchors remain sound.

---

## Established Security Properties

The automated verification of the model demonstrates several core cryptographic and architectural guarantees:

* **Cryptographic Evidence-to-Handshake Binding:** The attestation evidence (`pat`/`kat`/`ev`) cannot be transplanted or replayed across sessions:
  * **Intra-handshake model:** Attestation evidence is embedded directly into the handshake transcript, establishing unconditional transcript and session correlation `(kc1, ev1) = (kc2, ev2)`.
  * **Post-handshake model:** Evidence binding is derived from post-handshake exported key material (`ClientStateEvKc`), ensuring agreement on secrets (`kem_ss`, `kch`, `kc`) while session correlation `(kc1, ev1) = (kc2, ev2)` holds strictly on the condition that a downgraded KEM parameter set (`WeakKEM`) is rejected.

* **Orthogonal Trust-Root Redundancy:** Server authentication and remote attestation achieve independent survivability under single-root compromise across three anchors:

  * **CA Compromise Resilience:** If the Web PKI CA is fully compromised (`CompromisedCA`) or the TLS signing key is leaked (`LeakedTSK`), injective agreement on the handshake and remote attestation parameters (`ClientFinRecentAgr`, `ClientRANew`) still holds, provided the hardware attestation key (`pubAK`), Key Attestation Key (`pubKAK`), and CSP endorsement root remain intact.
  * **Attestation Compromise Resilience:** If the CSP root is compromised (`CompromisedCSP`) or an attestation key is leaked (`LeakedAK`/`LeakedKAK`), TLS session authentication and server identity integrity still hold via the standard PKI trust path.
  * **Policy Compromise Resilience:** If Verifier Owner policy validation is bypassed (`UncheckedPolicies`), enclave identity assurance can still hold, provided the CSP, AK, KAK, and (in Option B) CA anchors remain sound.

* **Mitigation of Identity Diversion and Relay Attacks:** Injective agreement over composition parameters (`ClientComp ==> PreServerComp`) prevents adversary-in-the-middle relay and splicing attacks. The client's perceived server identity is guaranteed to match the actual server (correlated `ClientID`/`ServerID` on the shared session key `kem_ss`), precluding DNS/SNI diversion attacks.

* **Per-Session Attestation Freshness:** Replay of pre-recorded quotes or platform measurement claims (`dev_status`) is prevented. Reachability of `AcceptedRdata` together with the `eat_nonce = expectedNonce` freshness check in appraisal guarantees that accepted measurements correspond to the active, live session instance.

* **Application Key Confidentiality:** The client application traffic secret (`kc`) remains completely confidential from active Dolev-Yao attackers. Secrecy is breached only in the explicit presence of underlying root compromise (`CompromisedCSP` / `CompromisedCA` / `UncheckedPolicies`), key compromise (`LeakedAK` / `LeakedKAK` / `LeakedTSK`), or negotiation downgrades to weak primitives (`WeakKEM` / `WeakHash`) elsewhere in the model.

### Security Equivalence of Post-Handshake Attestation

Intra-handshake attestation (e.g., Early Attestation) embeds the Conceptual Message Wrapper (CMW) directly into the TLS transcript by including it within the Certificate message—thereby directly feeding into the HKDF derivation of the application traffic keys. In contrast, post-handshake attestation (e.g., via RFC 9261 Exported Authenticators and ALTEA transport framing) derives the application traffic keys *before* attestation evidence is generated or exchanged.

Despite this structural difference, the formal model proves that post-handshake attestation preserves the exact same compound authentication and security properties as intra-handshake attestation. This equivalence is achieved through the **late attestation of the session binding value**:

1. **Exporter-Derived Binding:** Following the baseline connection establishment, the protocol derives an exported attestation binder from the TLS Exporter Master Secret (`ems`) and the authenticator request context.

2. **Cryptographic Commitment:** This binder, along with the TLS signing key (`pubTSK`), is committed directly into the attestation challenge (the EAT nonce) and signed by the hardware root of trust within the Platform Attestation Token (PAT).

3. **Retroactive Authentication:** Because the `ems` is cryptographically bound to the entire baseline TLS 1.3 handshake transcript, incorporating this binder into the hardware-signed quote retroactively authenticates the established connection.

Consequently, once the Relying Party successfully appraises the post-handshake evidence, the late attestation comprehensively authenticates the entire dual-phase exchange. This mathematically proves that the actively established application keys (`kc`, `ks`) terminate strictly within the genuine, provisioned enclave, delivering identity and authorization guarantees identical to intra-handshake integration—provided application data release is gated on appraisal completion.

> **Note:** These equivalence properties are formally proven to hold under baseline TLS 1.3 security assumptions. Specifically, relying parties must correctly validate handshake negotiation, reject degenerate or weak key exchange shares, and enforce transcript integrity checks as outlined in Appendix F. of RFC 9846.

---

## Enhanced Threat Modeling

**Adversary-controlled handshake secrets (Unconditional Leakage).** Immediately upon handshake key derivation, the server outputs the keys directly onto the public channel:
```proverif
   out(io, (kch, ksh));
   event AttackerControlsHSK(kch, ksh);
```

**Certificates as attacker-observable network data.** Servers receive their certificate over the network before validating it against internal state, rather than treating it as pre-trusted local data. This models certificates and their signatures as data the attacker can intercept, replay, or attempt to forge.

**Attacker-influenced server selection.** Clients receive their target server identity (SNI) from the network rather than from a pre-validated source, modeling realistic DNS/SNI-based routing and the possibility of client misdirection.

### Evidence Oracle

The `Attestation_Environment` process models both genuine attestation requests and an untrusted external API oracle — the UCCS proxy-forwarding attack described in `draft-reddy-rats-key-binding-02` §8.3:

* **The Vulnerability**: An untrusted caller passes arbitrary key material and device-state claims into a TEE-hosted service. If the enclave blindly forwards this caller-supplied input to the local attestation runtime, the hardware generates a valid PAT over rogue keys/measurements. An attacker could use this oracle to masquerade an arbitrary external endpoint as a secure enclave. The model evaluates whether the client's endorsement verification pipeline — comparing the appraised device state against a VOK-signed `RefValTriple` — successfully detects and rejects such malicious or misconfigured VM launches.
* **Scope of Proxy Forwarding Vulnerability**: Changing transport placement (moving from intra-handshake `log_SH` binding to post-handshake `ems` exporter binding) does not alter the underlying trust dependencies of evidence appraisal. In direct attestation architectures (Option A), an attestation interface that quotes arbitrary input produces a validly bound session unless the relying party explicitly filters out `ExternalOrUntrusted` evidence. Under compound additive authentication (Option B), the oracle attack is neutralized under standard trust assumptions (uncompromised AK, TSK, CSP, CA, and Verifier Owner policy) even without evaluating key provenance attributes.
* **The Defense Mechanism**: Hardware-enforced key provenance flags differentiate keys generated within the cryptographic boundary from imported keys. When the oracle signs over externally supplied data, the hardware tags the quote claims with `ExternalOrUntrusted` rather than `TrustedNonExportable`.

**Selective, per-party compromise.** Each key class — TLS signing key (`TSK`), attestation key (`AK`), Key Attestation Key (`KAK`), CA root key, CSP root key, and Verifier Owner policy key (`VOK`) — has its own independent leakage process, each tagged with a distinct event. This lets the model reason about partial-compromise scenarios rather than all-or-nothing trust, distinguishing three independent trust paths:

* **WebPKI Trust Path:** Evaluates compromises along the standard TLS certificate hierarchy through `CompromisedCA` (compromise of the Certificate Authority signing key) and `LeakedTSK` (exfiltration of the cVM's TLS private key). This isolates transport-layer identity failures to test whether remote attestation evidence and platform endorsements can preserve session integrity even if the Web PKI authority or TLS key is completely undermined.
* **Platform Endorsement (CSP) Trust Path:** Evaluates compromises along the confidential computing endorsement chain through `CompromisedCSP` (compromise of the Cloud Service Provider endorsement key) and `LeakedAK`/`LeakedKAK` (leakage of the platform attestation key or the per-cVM key attestation key). This isolates hardware platform trust to determine whether the standard Web PKI identity pipeline remains sufficient to authenticate the server and protect session data even when platform measurement endorsements are untrusted or forged.
* **Verifier Owner (Policy) Trust Path:** Evaluates the compromise or bypass of Verifier Owner appraisal policy (`UncheckedPolicies`, `privVOK` via `LVOK`) independently of the hardware and Web PKI roots, isolating whether reference-value/manifest policy is a single point of failure.

| Event | Compromised Subject |
| --- | --- |
| `LeakedTSK` | TLS signing key of an individual workload instance |
| `LeakedAK` | Attestation key of an individual cVM instance |
| `LeakedKAK` | Key Attestation Key of an individual cVM instance |
| `CompromisedCA` | Web PKI certification authority private key (`privCA`) |
| `CompromisedCSP` | CSP / platform endorser private key (`privCSP`) |
| `UncheckedPolicies` | Verifier Owner private signing key (`privVOK`) |

Each compromise event is triggered by a matching adversarial leak process in the driver file: `LCA` leaks `privCA`, `LCSP` leaks `privCSP` (firing `CompromisedCSP`), `LKAK` leaks `privKAK` (firing `LeakedKAK`), and `LVOK` leaks `privVOK` (firing `UncheckedPolicies`).

---

## Additional Modeling Features

**Zero Key-Schedule Modification (Intra-Handshake).** The intra-handshake protocol strictly preserves the standard TLS 1.3 key schedule (RFC 9846) without introducing custom HKDF labels, modified extractors, or handshake-secret derivations:
  - **No Secret-Binding Dependency:** Security does not rely on binding attestation evidence to ephemeral handshake traffic secrets (`htsc`, `htss`, `kch`, `ksh`) or intermediate key schedule states (`hs`).
  - **Soundness Under Handshake Key Exposure:** Handshake write keys (`kch`, `ksh`) are modeled as fully compromised and published directly to the adversary (`out(io, (kch, ksh))`). The formal verification proves that client application traffic key confidentiality (`kc`) and mutual session binding (`ClientComp`) remain unbroken even when the adversary actively possesses the handshake traffic keys.
  - **Transcript-Enforced Authenticity:** Handshake binding is achieved entirely through the public cryptographic transcript (`log_SH = (ch, SH)`), hashed into the session nonce `sessionNonce = hash(StrongHash, (log_SH, pubTSK))` and embedded as the attestation nonce inside the PAT claims signed by the hardware attestation key — demonstrating that altering the TLS 1.3 key schedule is unnecessary for formal session integrity on this timing path. (The post-handshake path binds evidence via the TLS 1.3 exporter interface instead — see "Key Hierarchy and Split-Model Attestation" and the exporter-derivation labels in `atls-library.pvl`.)

**Three independent trust roots.** A Web PKI-style Certificate Authority (CA) certifies the binding between a server's identity and its TLS key (`ID_S`, `pubTSK`). A Cloud Service Provider (CSP) endorsement authority independently certifies the binding between an attestation key and its expected launch measurements (`pubAK`, `dev_statusRef`). A Verifier Owner (VOK) independently signs the reference-value manifest binding workload identity to expected measurements. These are modeled as distinct signers with distinct keys, reflecting that certificate issuance, hardware endorsement, and policy/reference-value authorship are handled by different real-world organizations.

* **Relocation of the enforcement boundary to the client:** Prior models treated cryptographic downgrade and parameter selection as server-driven behaviors (`ServerChoosesKEX`, `ServerChoosesHash`). Because Confidential Computing assumes the server executes within an untrusted host or hypervisor that actively proposes malicious parameters, the evaluation reframes these failure modes around client-side acceptance (`ClientAcceptsKEM`, `ClientAcceptsHash`). Verification proves that the protocol remains secure across hostile server environments unless the client implementation actively fails to enforce basic KEM parameter and hash-algorithm validation.
* **Factorized single-root resilience versus monolithic disjunctions:** Rather than combining all possible leakages into a single, catch-all failure disjunction where any compromised key collapses the model, queries are split into contrasting groups. This factorizes the proof space across three distinct survivability paths — Web PKI compromise (`CompromisedCA` / `LeakedTSK`), platform attestation compromise (`CompromisedCSP` / `LeakedAK` / `LeakedKAK`), and policy compromise (`UncheckedPolicies`) — proving that identity and session security hold independently under single-root failure rather than demanding all-or-nothing trust.
* **Isolation of algebraic attacks from binder logic:** The queries decouple low-level ML-KEM parameter-downgrade exploits (`ClientAcceptsKEM(..., WeakKEM)`) from the core attestation binder. This isolates whether an attack represents an algebraic weakness or a protocol design flaw, directly exposing why intra-handshake transcript integration inherently neutralizes weak-KEM parameter sets while post-handshake key-exporter binding leaves session correlation conditional on client-side KEM parameter validation.

**Per-launch measurement granularity.** Each server launch generates its own fresh reference measurement value (`dev_statusRef`) rather than checking against one shared global reference. This allows the model to represent many different, independently-measured launches — correctly configured or otherwise — coexisting under the same hardware root of trust.

---

## Reference Material

### Principals

* **CA**: Web PKI trust anchor that signs X.509 certificates binding `ID_S` to `pubTSK`.
* **CSP**: Cloud Service Provider / infrastructure endorser (keys `pubCSP`/`privCSP`) that certifies `pubAK` associations with platform launch measurements via `gen_endorsements`.
* **VOK**: Verifier Owner (keys `pubVOK`/`privVOK`, active in Option B) that signs the workload manifest and baseline measurements (`dev_statusRef`) via `gen_reference_manifest`, binding workload identifiers to expected runtime measurements.
* **Client / Server**: TLS protocol endpoints. Each confidential VM (cVM) server instance hosts an attestation key (`pubAK`), a per-cVM Key Attestation Key (`pubKAK`), and one or more workload TLS signing keys (`pubTSK`).
* **Adversary**: Active Dolev-Yao attacker operating over the public channel `io`, capable of intercepting and injecting traffic, instantiating server processes with arbitrary parameters, and triggering selective key compromises.

### Project Layout

```
../
├── atls-multiagent-driver.pv
├── queries.pvl
├── conf/branchA.pvl
├── conf/branchB.pvl
├── conf/noBadKEXHash.pvl
├── models/atls-library.pvl
├── models/evidence.pvl
├── models/verifier.pvl
├── models/hs-cv-fin.pvl
├── models/intra-hs.pvl
└── models/post-hs.pvl
```

To run:

```
 proverif205 \
 -lib conf/branch{A,B}.pvl \
 -lib conf/noBadKEXHash.pvl \
 -lib models/atls-library.pvl \
 -lib models/hs-cv-fin.pvl \
 -lib models/evidence.pvl \
 -lib models/verifier.pvl \
 -lib models/{intra,post}-hs.pvl \
 -lib queries.pvl \
 atls-multiagent-driver.pv 2>&1 | tee logs/log-{intra,post}-{a,b}.txt
```

`idOption` (Option A/B) and `timingOption` (Intra/Post) are not selected via separate config-file variants — both are adversary-supplied inputs read at runtime inside `atls-multiagent-driver.pv` and `verifier.pvl` respectively.

### Normative References

* RFC 9846 (TLS 1.3)
* RFC 9334 (RATS Architecture)
* `draft-ietf-seat-use-cases-01`
* `draft-reddy-rats-key-binding-02`
* `draft-ietf-rats-corim-11`
* `draft-ietf-tls-mlkem-10`
* `draft-fossati-seat-early-attestation-06`
* `draft-fossati-seat-expat-03`
* `draft-reddy-seat-expat-transport-01` (ALTEA)

---

## Acknowledgements

Thank you to all participants of the SEAT working group for the discussions and contributions that helped shape this project.

This work is a unique adaptation and extension incorporating input from the following prior art:

* [Identity Crisis in Confidential Computing: Formal analysis of attested TLS protocols](https://github.com/CCC-Attestation/formal-spec-id-crisis/tree/main)
* [Intra-handshake.fail (CVE-2026-33697): High-severity CVE in Attested TLS](https://www.researchgate.net/publication/408219182_Intra-handshakefail_CVE-2026-33697_High-severity_CVE_in_Attested_TLS)
* [Verified Models and Reference Implementations for the TLS 1.3 Standard Candidate](https://ieeexplore.ieee.org/document/7958594)

Thank you to all involved authors and researchers!

### Diff from upstream models

> A high-level overview of changes from the Sardar et al. prior art is available [here](https://notes.ietf.org/s/WA5Zt9-D4V)

## Copyright and License

Copyright 2017 Bhargavan et al.
Copyright 2026 Sardar et al.
Copyright 2026 Nathanael Ritz.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
