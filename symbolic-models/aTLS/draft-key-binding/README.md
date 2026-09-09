# Symbolic Modeling for Identity Workload Policy and Evidence Oracle Mitigation

## Threat Model: The Evidence Oracle & UCCS Proxy Forwarding

In confidential computing architectures, a TEE instance uses an Attestation Key (AK) endorsed by the platform vendor or Cloud Service Provider (CSP) to sign measurements and session keys (`rdata`).

> NOTE: pubEK and pubLTK have been renamed to pubTSK (TLS Signing Key) and is akin to TIK (draft-early) and AIK (draft-expat).

The **Evidence Oracle** models a signing oracle vulnerability (specifically the UCCS proxy-forwarding attack described in `draft-reddy-rats-key-binding` §8.3):

* **The Vulnerability**: An untrusted caller passes arbitrary key material into a TEE-hosted service. If the enclave blindly forwards this caller-supplied key to the local attestation runtime, the hardware generates a valid attestation quote over rogue keys. An attacker could use this oracle to masquerade an arbitrary external endpoint as a secure enclave.

* **Universal Vulnerability to Proxy Forwarding**: Changing transport placement (moving from intra-handshake `log_SH` binding to post-handshake `ems` exporter binding) does not eliminate the need for provenance appraisal. In both modes, an attestation interface that quotes arbitrary input will produce a validly bound session for an attacker unless the client explicitly rejects `ExternalOrExportable` evidence.

* **The Defense Mechanism**: Hardware-enforced key provenance flags differentiate keys generated within the cryptographic boundary from imported keys. When the oracle signs over externally supplied data, the hardware tags the quote claims with `ExternalOrExportable` rather than `LocalNonExportable`.

---

## Baseline Client Security Invariants

To guarantee session integrity while the `EvidenceOracle` is active, the client appraisal engine enforces three invariants:

* **Rejection of Cryptographic Downgrades**: The client strictly rejects weak or collision-vulnerable hash algorithms negotiated during the handshake (`WeakHash`), preventing transcript collision attacks.

* **Rejection of Degenerate Key Shares**: The client rejects small-subgroup Diffie-Hellman parameters (`BadElement`), preventing key-exchange manipulation and forced shared-secret predictability.

* **Strict Key Provenance Filtering**: The client verifies the `key-attributes` claim in the evidence quote and immediately aborts if tagged with `ExternalOrExportable`. Only keys generated strictly within the enclave (`LocalNonExportable`) are accepted.

---

### Option A: Direct Attestation Anchoring

Attestation replaces Web PKI entirely. The client resolves the expected target identity directly to expected enclave launch measurements. The TLS handshake key (`pubTSK`) is self-signed, and the attestation quote serves as the sole proof of authentication and session binding.

### Option B: Compound Additive Authentication

Attestation composes additively with standard Web PKI. The server presents a standard CA-signed X.509 certificate matching the target domain or service identity. In parallel, it presents an attestation quote and a signed **Workload Manifest** issued by the workload policy owner.

* **Additive Trust Anchor**: The manifest cryptographically links the X.509 identity (`WID / ID_S`) to the allowed runtime measurements (`dev_statusRef`).
* **Compromise Resilience**:
  - If a Web PKI CA is compromised or untrusted DNS redirects traffic, the attacker cannot spoof the instance without presenting measurements that match the owner's signed manifest and valid hardware endorsements.
  - If an enclave host is compromised, the attacker cannot impersonate the domain without a CA-signed certificate and owner-signed manifest.

---

### Misc

* **Handshake Placement**: Evidence can be transmitted within the TLS 1.3 handshake via the Certificate message (intra-handshake aTLS) or post-handshake using Exported Authenticators (RFC 9261 / ALTEA).

* **Session Binding Exporter**: For post-handshake exchanges, the attestation quote binds to the TLS 1.3 exporter secret (`ems`) combined with a client-supplied request context, ensuring quotes cannot be replayed across distinct TLS sessions.

#### Project layout

```
├── conf
│   ├── branchA.pvl
│   ├── branchB.pvl
│   ├── noBadElementHash.pvl
│   └── noExternalOrExportable.pvl
├── logs
│   ├── logs-intra-a.txt
│   ├── logs-intra-b.txt
│   ├── logs-post-a.txt
│   └── logs-post-b.txt
├── models
│   ├── ClientStateEvKcBinding.ascii-art
│   ├── atls-lib-intra.pvl
│   └── atls-lib-post.pvl
├── queries.pvl
└── tls13-multiagent.pv
```

To run: 

```
proverif205 -lib models/atls-lib-<intra|post>.pvl -lib conf/noBadElementHash.pvl -lib queries.pvl -lib conf/branch<A|B>.pvl [-lib conf/noExternalOrExportable.pvl] -html traces tls13-multiagent.pv 2>&1 | tee logs/logs-<intra|post>-<a|b>.txt
```

e.g. for intra-handshake with Option A with RPK:
`proverif205 -lib models/atls-lib-intra.pvl -lib conf/noBadElementHash.pvl -lib queries.pvl -lib conf/branchA.pvl -html traces tls13-multiagent.pv 2>&1 | tee logs/logs-intra-a.txt`

## Driver File: tls13-multiagent.pv

`tls13-multiagent.pv` serves as the top-level ProVerif entry point. Protocol cryptography, equational theories, and message constructors are specified in the paired library files (`atls-lib-intra.pvl` or `atls-lib-post.pvl`). This file instantiates the system principals, defines honest protocol workflows, specifies adversarial compromise interfaces, and composes all active roles into a single top-level parallel process.

---

### Structural Extensions to the Baseline TLS 1.3 Model

The driver extends the baseline single-path TLS 1.3 specification (Bhargavan et al.) through five architectural modifications:

* **Consolidated Workload Keying**: Replaces the separate long-term identity key (`pubLTK`) and ephemeral TLS handshake key (`pubEK`) with a unified TLS Signing Key (`pubTSK`). Under Option A, `pubTSK` is self-signed; under Option B, it is certified by a CA. The legacy `LeakedLTK` event is eliminated.
* **Additional External Trust Anchors**: Introduces Cloud Service Provider / Independent Software Vendor (`CSP / ISV`) and Workload Policy Owner (`Owner`) principals, accompanied by respective compromise events (`CompromisedCSP`, `UncheckedPolicies`) to model compound authorization architectures.
* **Provisioning and Lifecycle Instrumentation**: Introduces setup-phase events (`CSRSigned`, `ProvisionedTargetEnv`, `EndorsementsIssued`, and `IdentityManifestIssued`). These events establish baseline reachability invariants and support correspondence lemmas verifying Target Environment Engagement.
* **Adversarial Session Seeding**: Enables adversary-controlled inputs (certificates or target launch measurements) to initialize `run_Server` prior to internal key matching, modeling misconfigured server provisioning and malicious tenant dispatching.
* **Hardware Identifier Anonymization**: Adds a Universal Entity ID (`UEID`) attribute to `agent_keys`, which `gen_endorsements` overwrites with `NULL_ID` to model CSP-enforced pseudonymization of physical hardware platform identifiers.

---

### Identity Anchoring Variants

The driver parameterizes identity appraisal via `idOption: branch_option`, which is evaluated across key generation, manifest issuance, and endpoint execution:

* **Option A (Direct Attestation Anchoring)**: The workload identity is derived directly from target measurements via `dev2id(dev_statusRef)`. `pubTSK` is self-signed without Web PKI certificates. Attestation evidence serves as the sole root of authentication and session binding.
* **Option B (Compound Additive Authentication)**: Identity (`ID_S`) is bound to `pubTSK` through a CA-signed X.509 certificate, while an Owner-signed manifest binds `ID_S` to authorized launch measurements (`dev_statusRef`). Authentication requires concurrent appraisal across both Web PKI and platform endorsement chains.

---

### Threat Model: Evidence Oracle

The `TEE_Oracle_Vuln` process models the UCCS proxy-forwarding attack (`draft-reddy-rats-key-binding` §8.3). The adversary provides arbitrary runtime data (`rdata`) and launch measurements to the attestation interface, prompting an endorsed `privAK` to sign over attacker-selected material. The resulting quote is tagged with the `ExternalOrExportable` key provenance attribute, modeling platform hardware assertions that the underlying key material originated outside the physical enclave boundary.

### Compromise Events

Compromise events are scoped to individual entities via `agent_keys` table lookups rather than leaking global signing material:

| Event | Compromised Subject |
| --- | --- |
| `LeakedTSK` | TLS signing key of an individual workload instance |
| `LeakedAK` | Attestation key of an individual cVM instance |
| `CompromisedCA` | Web PKI certification authority private key |
| `CompromisedCSP` | CSP / platform endorser private key |
| `UncheckedPolicies` | Workload policy owner private signing key |

These five events form the predicate base for the verification queries in `queries.pvl`, enabling evaluation of security guarantees under varying assumptions of partial infrastructure compromise.

---

### Principals

* **CA**: Web PKI trust anchor that signs X.509 certificates binding `ID_S` to `pubTSK`.
* **CSP / ISV**: Platform hardware endorser that certifies `pubAK` associations with platform launch measurements.
* **Owner**: Workload policy authority (active in Option B) that signs manifests binding workload identifiers to expected runtime measurements.
* **Client / Server**: TLS protocol endpoints. Each confidential VM (cVM) server instance hosts an attestation key (`pubAK`) and one or more workload TLS signing keys (`pubTSK`).
* **Adversary**: Active Dolev-Yao attacker operating over the public channel `io`, capable of intercepting and injecting traffic, instantiating server processes with arbitrary parameters, and triggering selective key compromises.


## Acknowledgements

Thank you to all participants of the SEAT working group for the discussions and contributions that helped shape this project.

---

This work is a unique adaptation and extension incorporating input from the following prior art:

* [Identity Crisis in Confidential Computing: Formal analysis of attested TLS protocols](https://github.com/CCC-Attestation/formal-spec-id-crisis/tree/main)
* [Intra-handshake.fail (CVE-2026-33697): High-severity CVE in Attested TLS](https://www.researchgate.net/publication/408219182_Intra-handshakefail_CVE-2026-33697_High-severity_CVE_in_Attested_TLS)
* [Verified Models and Reference Implementations for the TLS 1.3 Standard Candidate](https://ieeexplore.ieee.org/document/7958594)

Thank you to all involved authors and researchers!

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
