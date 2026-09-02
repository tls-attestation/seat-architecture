# Multi-Agent/Multi-Modal Attestation model: for Intra- and Post- handshake Attested TLS 1.3 

## Overview

This ProVerif model formalizes an attested TLS 1.3 handshake between clients and confidential-computing server VMs (cVMs), layered on top of the standard TLS 1.3 key schedule covering both attestation timing models for intra-handshake and post-handshake attestation. The symbolic model evaluates multi-tenant/cloud deployments, where many physical machines each host many independently-launched guest cVMs or workloads.

## Established Security Properties

The automated verification of the model demonstrates several core cryptographic and architectural guarantees:

* **Cryptographic Evidence-to-Handshake Binding:** The attestation evidence (`quote`/`ev`) cannot be transplanted or replayed across sessions:
  * **Intra-handshake model:** Attestation evidence is embedded directly into the handshake transcript, establishing unconditional transcript and session correlation `(kc1, ev1) = (kc2, ev2)`.
  * **Post-handshake model:** Evidence binding is derived from post-handshake exported key material (`ClientStateEvKc`), ensuring agreement on secrets (`gxy`, `kch`, `kc`) while session correlation `(kc1, ev1) = (kc2, ev2)` holds strictly on the condition that degenerate key share elements (`BadElement`) are rejected.

* **Orthogonal Trust-Root Redundancy:** Server authentication and remote attestation achieve independent survivability under single-root compromise:

* **CA Compromise Resilience:** If the WebPKI CA is fully compromised (`CompromisedCA`) or the TLS signing key is leaked (`LeakedEK`), injective agreement on the handshake (`ClientFinishedWithID`) and remote attestation parameters (`ClientRA`) still holds, provided the hardware attestation key (`pubAK`) and CSP endorsement root remain intact.

* **Attestation Compromise Resilience:** If the CSP root is compromised (`CompromisedCSP`) or an attestation key is leaked (`LeakedAK`), TLS session authentication and server identity integrity (`idS = idXS`) still hold via the standard PKI trust path.

* **Mitigation of Identity Diversion and Relay Attacks:** Injective agreement over composition parameters (`ClientComp ==> PreServerComp`) prevents adversary-in-the-middle relay and splicing attacks. The client's perceived server identity is guaranteed to match the actual server (`idS = idXS`), precluding DNS/SNI diversion attacks.

* **Per-Session Attestation Freshness:** Replay of pre-recorded quotes or platform measurement claims (`dev_status`) is prevented. Injective agreement on accepted measurements (`AcceptedRdata ==> SentRdata`) guarantees that measurements correspond to the active, live session instance.

* **Application Key Confidentiality:** The client application traffic secret (`kc`) remains completely confidential from active Dolev-Yao attackers. Secrecy is breached only in the explicit presence of underlying root compromise (`CompromisedCSP` / `CompromisedCA`), key compromise (`LeakedAK` / `LeakedEK`), or negotiation downgrades to weak primitives (`BadElement` / `WeakHash`).

## Enhanced Threat Modeling

**Adversary-controlled handshake secrets (Unconditional Leakage).** Immediately upon handshake key derivation, the server outputs the keys directly onto the public channel:
```proverif
   out(io, (kch, ksh));
   event AttackerControlsHSK(kch, ksh);
```

**Certificates as attacker-observable network data.** Servers receive their certificate over the network before validating it against internal state, rather than treating it as pre-trusted local data. This models certificates and their signatures as data the attacker can intercept, replay, or attempt to forge.

**Attacker-influenced server selection.** Clients receive their target server identity (SNI) from the network rather than from a pre-validated source, modeling realistic DNS/SNI-based routing and the possibility of client misdirection.

**Attacker-driven attestation measurement quotes.** Maintains existing Evidence generation process that collects platform measurement claims directly from adversary control (`in(io, dev_status1: bitstring)`). This enables the adversary to drive quote creation with arbitrary or tampered guest VM states signed by an authentic attestation key (`privAK`). The model evaluates whether the client's endorsement verification pipeline (`dev_status1 = dev_statusRef` validated against `pubISV` signatures) successfully detects and rejects malicious or misconfigured VM launches.

**Selective, per-party compromise.** Each key class—TLS key (`EK`), attestation key (`AK`), CA root key, and ISV/CSP root key—has its own independent leakage process, each tagged with a distinct event (`LeakedEK`, `LeakedAK`, `CompromisedCA`, `CompromisedCSP`). This lets the model reason about partial-compromise scenarios rather than all-or-nothing trust, distinguishing two distinct trust paths:

* **WebPKI Trust Path:** Evaluates compromises along the standard TLS certificate hierarchy through `CompromisedCA` (compromise of the Certificate Authority signing key) and `LeakedEK` (exfiltration of the cVM's TLS private key). This isolates transport-layer identity failures to test whether remote attestation evidence and platform endorsements can preserve session integrity even if the WebPKI authority or TLS key is completely undermined.

* **ISV / Cloud Service Provider (CSP) Trust Path:** Evaluates compromises along the confidential computing endorsement chain through `CompromisedCSP` (compromise of the independent software vendor or Cloud Service Provider endorsement key) and `LeakedAK` (leakage of the physical machine's attestation key). This isolates hardware platform trust to determine whether the standard WebPKI identity pipeline remains sufficient to authenticate the server and protect session data even when platform measurement endorsements are untrusted or forged.

**Negative testing via `insecureSkipVerify` variant.** A dedicated model configuration strips out the attestation binder (`rdata = zero`) and omits reference measurement verification (e.g., `dev_status1 = dev_statusRef`), modeling an insecure verification bypass often introduced for debugging or testing. This control serves as a sanity check. Under this variant, the verification summary confirms a complete collapse of session security: 
  - evidence-to-handshake binding breaks entirely across Diffie-Hellman secrets (`gxy`), handshake keys (`kch`), and application keys (`kc`), 
  - session correlation fails, and 
  - client application secrets (`kc`) leak unconditionally to the attacker without requiring root key or authority compromise.

## Additional Modeling Features

**Zero Key-Schedule Modification.** The intra-handshake protocol strictly preserves the standard TLS 1.3 key schedule (RFC 8446) without introducing custom HKDF labels, modified extractors, or handshake-secret derivations:
  - **No Secret-Binding Dependency:** Security does not rely on binding attestation evidence to ephemeral handshake traffic secrets (`htsc`, `htss`, `kch`, `ksh`) or intermediate key schedule states (`hs`).
  - **Soundness Under Handshake Key Exposure:** Handshake write keys (`kch`, `ksh`) are modeled as fully compromised and published directly to the adversary (`out(io, (kch, ksh))`). The formal verification proves that client application traffic key confidentiality (`kc`) and mutual session binding (`ClientComp`) remain unbroken even when the adversary actively possesses the handshake traffic keys.
  - **Transcript-Enforced Authenticity:** Handshake binding is achieved entirely through the public cryptographic transcript (`log_SH = (ch, SH)`) signed over inside attestation claims (`rdata = (log_SH, pubEK)`), demonstrating that altering the TLS 1.3 key schedule is completely unnecessary for formal session integrity.

**Two independent trust roots.** A WebPKI-style Certificate Authority (CA) certifies the binding between a server's identity and its TLS key (`ID_S`, `pubEK`). A separate Independent Software Vendor / Cloud Service Provider (ISV/CSP) endorsement authority independently certifies the binding between an attestation key and its expected launch measurements (`pubAK`, `dev_statusRef`). These are modeled as distinct signers with distinct keys, reflecting that certificate issuance and attestation endorsement are handled by different real-world organizations.

* **Relocation of the enforcement boundary to the client:** Prior models treated cryptographic downgrade and parameter selection as server-driven behaviors (`ServerChoosesKEX`, `ServerChoosesHash`). Because Confidential Computing assumes the server executes within an untrusted host or hypervisor that actively proposes malicious parameters, the new evaluation reframes these failure modes around client-side acceptance (`ClientAcceptsElement`, `ClientAcceptsHash`). Verification proves that the protocol remains secure across hostile server environments unless the client implementation actively fails to enforce basic parameter and subgroup validation.

* **Factorized single-root resilience versus monolithic disjunctions:** Rather than combining all possible leakages into a single, catch-all failure disjunction where any compromised key collapses the model, queries are split into contrasting pairs. This factorizes the proof space into two distinct survivability paths—WebPKI compromise (`CompromisedCA` / `LeakedEK`) versus platform attestation compromise (`CompromisedCSP` / `LeakedAK`)—proving that identity and session security hold independently under single-root failure rather than demanding all-or-nothing trust.

* **Isolation of algebraic attacks from binder logic:** The queries decouple low-level Diffie-Hellman small-subgroup exploits (`ClientID(..., BadElement)`) from the core attestation binder. This isolates whether an attack represents an algebraic weakness or a protocol design flaw, directly exposing why intra-handshake transcript integration inherently neutralizes invalid points while post-handshake key-exporter binding leaves session correlation conditional on client curve validation.

**Per-launch measurement granularity.** Each server launch generates its own fresh reference measurement value (`dev_statusRef`) rather than checking against one shared global reference. This allows the model to represent many different, independently-measured launches — correctly configured or otherwise — coexisting under the same hardware root of trust.


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

