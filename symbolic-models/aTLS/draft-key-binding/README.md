# Symbolic Modeling for Identity Workload Policy and Evidence Oracle Mitigation

### Threat Model: The Evidence Oracle & UCCS Proxy Forwarding

In confidential computing architectures, a TEE instance uses an Attestation Key (AK) endorsed by the platform vendor or Cloud Service Provider (CSP) to sign measurements and session keys (`rdata`).

> NOTE: pubEK and pubLTK have been renamed to pubTSK (TLS Signing Key) and is akin to TIK (draft-early) and AIK (draft-expat).

The **Evidence Oracle** models a signing oracle vulnerability (specifically the UCCS proxy-forwarding attack described in `draft-reddy-rats-key-binding` §8.3):

* **The Vulnerability**: An untrusted caller passes arbitrary key material into a TEE-hosted service. If the enclave blindly forwards this caller-supplied key to the local attestation runtime, the hardware generates a valid attestation quote over rogue keys. An attacker could use this oracle to masquerade an arbitrary external endpoint as a secure enclave.

* **Universal Vulnerability to Proxy Forwarding**: Changing transport placement (moving from intra-handshake `log_SH` binding to post-handshake `ems` exporter binding) does not eliminate the need for provenance appraisal. In both modes, an attestation interface that quotes arbitrary input will produce a validly bound session for an attacker unless the client explicitly rejects `ExternalOrExportable` evidence.

* **The Defense Mechanism**: Hardware-enforced key provenance flags differentiate keys generated within the cryptographic boundary from imported keys. When the oracle signs over externally supplied data, the hardware tags the quote claims with `ExternalOrExportable` rather than `LocalNonExportable`.

---

### Baseline Client Security Invariants

To guarantee session integrity while the `EvidenceOracle` is active, the client appraisal engine enforces three invariants:

* **Rejection of Cryptographic Downgrades**: The client strictly rejects weak or collision-vulnerable hash algorithms negotiated during the handshake (`WeakHash`), preventing transcript collision attacks.

* **Rejection of Degenerate Key Shares**: The client rejects small-subgroup Diffie-Hellman parameters (`BadElement`), preventing key-exchange manipulation and forced shared-secret predictability.

* **Strict Key Provenance Filtering**: The client verifies the `key-attributes` claim in the evidence quote and immediately aborts if tagged with `ExternalOrExportable`. Only keys generated strictly within the enclave (`LocalNonExportable`) are accepted.

---

#### Option A: Direct Attestation Anchoring

Attestation replaces Web PKI entirely. The client resolves the expected target identity directly to expected enclave launch measurements. The TLS handshake key (`pubTSK`) is self-signed, and the attestation quote serves as the sole proof of authentication and session binding.

#### Option B: Compound Additive Authentication

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

## Acknowledgements

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
