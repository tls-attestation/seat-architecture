---
title: "Secure Evidence and Attestation Transport (SEAT) Architecture"
abbrev: "SEAT Architecture"
category: info

docname: draft-mihalcea-seat-architecture-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: SEC
# workgroup: SEAT Working Group
keyword:
 - remote attestation
 - TLS
 - confidential computing
 - transport security
venue:
#  group: SEAT
#  type: Working Group
#  mail: seat@ietf.org
#  arch: https://example.com/WG
  github: "telephonicrobotics/seat-architecture"
  latest: "https://telephonicrobotics.github.io/seat-architecture/draft-seat-architecture.html"

author:
  - fullname: Ionuț Mihalcea
    organization: Arm
    email: ionut.mihalcea@arm.com
  - fullname: Thomas Fossati
    organization: Linaro
    email: thomas.fossati@linaro.org
  - fullname: Tirumaleswar Reddy
    organization: Nokia
    email: "kondtir@gmail.com"
  - fullname: Nathanael Ritz
    organization: Independent
    email: "ietf@nritz.com"

normative:
  RFC2119:
  RFC8174:
  RFC9334:
  I-D.mihalcea-seat-use-cases:

informative:
  I-D.ounsworth-rats-privacy-framework:
  I-D.reddy-rats-key-binding:
  IAB-Attestation-Risks:
    title: "IAB Statement on the Risks of Attestation of Software and Hardware on the Open Internet"
    date: 2023-09-25
    target: https://datatracker.ietf.org/doc/statement-iab-statement-on-the-risks-of-attestation-of-software-and-hardware-on-the-open-internet/
    author:
      - org: Internet Architecture Board (IAB)
  NIEME2021: DOI.10.1007/978-3-030-91625-1_10

...

--- abstract


This document defines an architectural framework for composing Remote
ATtestation procedureS (RATS) with Secure Evidence and Attestation
Transport (SEAT).  The document establishes normalized terminology
for SEAT, aligns RATS roles to transport endpoints, outlines
topological patterns for attestation delivery timing, characterizes
the abstract cryptographic pattern by which Evidence is bound to a
given transport connection, and describes the architectural
requirements for end-to-end trust in deployments traversing
intermediaries such as proxies.


--- middle

# Introduction

## Establishing Trust in Secure Communications

> "Cryptography [without system integrity] is like investing in an
  armored car to carry money between a customer living in a cardboard
  box and a person doing business on a park bench."
>
> — Gene Spafford
{: =aside}

Traditional secure channel protocols, such as Transport Layer
Security (TLS), primarily establish trust in a peer's identity. This
is typically achieved through mechanisms like a Public Key
Infrastructure (PKI), where a trusted Certification Authority (CA)
vouches for the binding between a public key and an identifier (e.g.,
a hostname).

However, this model has a core limitation: identity authentication
provides no assurance about the peer's internal state or the
integrity of its software stack.  A compromised server, for instance,
can still present a valid X.509 certificate and be considered
"trusted" by a client.  This gap allows compromised endpoints to
maintain network access and the trust of their peers, posing a
significant security risk in many environments.

## The Role of Remote Attestation

Remote Attestation (RA), as described in the RATS architecture
{{RFC9334}}, is a mechanism designed to fill this gap.  RA allows an
entity (the "Attester") to produce verifiable "Evidence" about its
current runtime state.  This Evidence covers the Attester's TCB, and
can thus include measurements of its firmware, operating system, and
application code, as well as the configuration of its hardware and
software security features (e.g., secure boot status, memory
isolation).  A "Relying Party" can then use this Evidence, often with
the help of a trusted "Verifier", to appraise the Attester's
trustworthiness.

By integrating RA into a secure channel establishment protocol, a
second dimension of trust—trustworthiness—is added to complement
regular peer authentication.  This allows a peer to make
authorization decisions based not just on who the other party is, but
also on what it is (e.g., an AMD SEV-SNP-based server running in some
known datacenter) and whether its state is acceptable.

## Purpose and Scope

This document is intended as an input to the design of protocol
solutions within the SEAT working group.  A key goal is to define
requirements for a solution that is agnostic to any specific
attestation technology (e.g., Trusted Platform Modules (TPMs), Intel
TDX, AMD SEV, Arm CCA, etc.).

For the scope of this architecture, the term "transport" is used
interchangeably with "secure transport" to refer to secure channel
establishment protocols.

## Use Cases

The use cases motivating this architecture are defined in
{{I-D.mihalcea-seat-use-cases}}.  Readers are directed there for the
full enumeration of deployment scenarios, requirements, and
properties that protocol work in the SEAT working group is expected
to satisfy.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

The following terms are used in this document.  Terms defined in
{{RFC9334}} are used with the meanings established there; the
definitions below extend or specialize those terms for the transport
context.

This document adopts terms of art such as `intra-` and `post-`
as coined by {{NIEME2021}}.

Attested Channel:
: A transport session in which at least one endpoint
  has produced Evidence that has been appraised, and in which that
  Evidence is cryptographically bound to the session such that the
  appraisal cannot be replayed to a different session or transferred
  to a different endpoint.

Attestation Timing Model:
: The temporal relationship between Evidence
  conveyance and connection establishment time.  This document
  defines two timing models: Intra-Handshake Attestation and Post-
  Handshake Attestation. See {{timing-models}}.

Evidence Generation Time:
: The point at which an Attester's Claims
  are signed to produce Evidence. Depending on the internal workings
  of the Attester, the Evidence reflects the reported state at the
  time the underlying Claims were collected and may not represent a
  snapshot of state at the exact moment of signing the evidence. In all cases, it
  makes no representation about the Attester's state at any later time.

Connection Establishment Time:
: The point at which a transport
  handshake completes and the session becomes usable for application
  data exchange.

Lifetime of Connection:
: The period from Connection Establishment
  Time until the session is torn down.  Post-handshake re-
  attestation operates during the Lifetime of Connection, allowing
  Evidence to reflect the Attester's current state rather than its
  state at Connection Establishment Time.

Re-attestation:
: The production and appraisal of fresh Evidence during an
  established session's Lifetime of Connection.

Intra-Handshake Window:
: The interval during transport connection
  establishment in which Evidence is conveyed within the handshake
  messages themselves, prior to the transition to application data
  exchange.

Post-Handshake Window:
: The interval following connection establishment in which Evidence
  is conveyed to the Relying Party using post-handshake protocol
  mechanisms (e.g., Exported Authenticators or application-layer
  exchanges).

Session Binding Value:
: A value, uniquely determined by a specific transport
  session, from which Attestation Binders are derived.  A Session
  Binding Value may be public or secret depending on the topology;
  what is required is that it cannot be known before the session is
  initiated.  See {{channel-binding-pattern}}.

Attestation Binder:
: A cryptographic value derived from a Session Binding
  Value and committed to by the Attesting Environment into its
  Evidence payload. This value binds the Evidence to a specific
  session guaranteed under typical cryptographic assumptions.

Transmission Anchor:
: The point in the protocol at which an
  Attestation Binder is included in a protocol message.  A binder
  may be computed and transmitted before peer authentication is
  complete.

Verification Anchor:
: The protocol mechanism by which the integrity
  of a transmitted Attestation Binder is established. Depending on
  the Attestation Timing Model, this may be achieved via a MAC
  that authenticates the handshake transcript (e.g., the TLS Finished
  message), or through post-handshake cryptographic binding (e.g.,
  Exported Authenticators).

Split Deployment:
: A deployment in which the Attesting Environment
  and the transport stack reside in different execution contexts.
  The transport stack is in the Target Environment; the Attesting
  Environment (e.g., a TEE) must receive the attestation binder
  input — typically a handshake transcript hash or exported key —
  from the transport stack via a trusted interface.

Participating Intermediary:
: A TLS-terminating proxy that operates
  outside the trusted network topology and is traversed by a client
  seeking to attest an origin endpoint behind it.  A Participating
  Intermediary is not a RATS role; it participates in Evidence
  routing but is not authorised to appraise Evidence or to access
  Evidence payloads.  This term applies specifically to the proxy-
  traversal topology, in which the attested properties belong to the
  origin rather than to the intermediary itself.

Layered Attestation:
: In layered attestation, Claims are collected
  from multiple execution layers beginning with a foundational
  Attesting Environment that is typically immutable or difficult to
  modify.  A common example is a TEE executing within a virtual
  machine, which is itself executing on a host platform.  In the
  transport context, the critical question is whether the layer
  holding the transport session's keying material is the same layer
  that signs the Evidence, or whether the attestation binder input
  must traverse a layer boundary.

# Roles and Entities

The SEAT architecture maps the roles defined in [RFC9334] to standard
transport protocol entities.  The subsections below describe each
role and its specific character in the transport context.

## Attester

The Attester produces Evidence about its current state for
consumption by a Verifier.  In the transport context, the Attester is
a network endpoint — either the Client or the Server — that possesses
an Attesting Environment (such as a Trusted Execution Environment)
capable of securely collecting Claims and signing them with an
attestation key.

The Attester's transport stack provides the attestation binder input
to the Attesting Environment so that Evidence can be bound to the
specific session.  In a Split Deployment, the transport stack is in
the Target Environment and the interface between the transport stack
and the Attesting Environment is a security-critical boundary: the
Attesting Environment MUST NOT treat an attestation binder input
received from an untrusted host as authoritative without first
verifying it has not been substituted.

In mutual attestation deployments, both the Client and the Server
simultaneously act as Attesters.  Each endpoint's Attesting
Environment independently generates Evidence bound to the session.

## Relying Party

The Relying Party consumes Attestation Results or Evidence and uses
them to make authorization decisions about the transport connection.
In the transport context, the Relying Party is typically the endpoint
opposite the Attester — the Server when the Client attests, or the
Client when the Server attests.

## Verifier

The Verifier appraises Evidence by applying an Appraisal Policy for
Evidence and produces Attestation Results.  In the transport context,
two deployment variants arise, corresponding to the two RATS
conveyance models:

Co-located Verifier:
: The Verifier is implemented within the Relying
  Party endpoint. This variant is typical of intra-handshake
  attestation, where Evidence is evaluated inline during the
  handshake and the Relying Party requires a real-time appraisal
  result before finalizing the connection.
  This deployment corresponds to the Background-Check Model of
  {{RFC9334}}.

Remote Verifier:
: The Verifier is an independent service. The
  Attester contacts the Verifier prior to or during the session and
  obtains Attestation Results, which it then presents to the Relying
  Party.  This variant may be more common in post-handshake
  attestation flows.
  This deployment corresponds to the Passport Model of {{RFC9334}}.

{{fig-roles}} illustrates how the RATS roles map onto the Client and
Server transport endpoints under the two conveyance models (see {{Section 5 of RFC9334}}).

~~~ ascii-art
 Background-Check Model (Verifier co-located with Relying Party)

  +---------------------+  Evidence   +---------------------------+
  |      Attester       |------------>|      Relying Party        |
  | (Client or Server)  |             | +-----------------------+ |
  |                     |             | | Verifier (co-located) | |
  |                     |             | +-----------------------+ |
  +---------------------+             +---------------------------+

 Passport Model (Remote Verifier)

  +---------------------+  Evidence   +---------------------+
  |      Attester       |------------>|   Remote Verifier   |
  | (Client or Server)  |<------------|                     |
  +---------------------+ Att.Results +---------------------+
            |
            | Attestation Results
            v
  +---------------------+
  |    Relying Party    |
  | (Server or Client)  |
  +---------------------+
~~~
{: #fig-roles title="RATS Roles Mapped to Transport Endpoints"}

# Trust Model

This section describes the trust relationships required to establish
an Attested Channel.  The general trust model of {{RFC9334}} Section 7
applies; the subsections below specialise it for the transport
context.

## Relying Party Trust

The Relying Party must trust that the Attestation Results it receives
accurately reflect the Attester's state, which depends on its trust
in the Verifier and in the Endorsement chain for the Attesting
Environment.

The Relying Party must additionally satisfy itself that
the Evidence is bound to the current session — that it has not been
replayed from a different session or transferred from a different
endpoint.  This assurance is provided by the session binding mechanism
described in {{channel-binding-pattern}}; the check may be performed by
the Relying Party itself or delegated to the Verifier, but it cannot
be pre-computed independently of the session.

## Attester Trust

For an Attesting Environment to be trustworthy to a Verifier, the
Verifier must be able to establish trust in the signing key the
Attesting Environment uses to produce Evidence.  This is accomplished
via an Endorsement chain from a hardware manufacturer or certificate
authority that attests to the Attesting Environment's properties and
the provenance of its attestation key.  In the transport context,
Endorsements may be conveyed alongside Evidence in the same transport
message, or fetched out-of-band by the Verifier prior to or during
appraisal.

## Verifier Trust

The Relying Party must have a trust relationship with the Verifier
commensurate with the sensitivity of the authorization decision.  In
the co-located Verifier deployment, this relationship is implicit:
the Verifier's logic is part of the Relying Party's own
implementation.  In the remote Verifier deployment, the Relying Party
must authenticate the Verifier and confirm that the Verifier's
Appraisal Policy for Evidence is consistent with the Relying Party's
own requirements before accepting Attestation Results.

## Participating Intermediary Trust

The Participating Intermediary is explicitly not trusted for the
security properties of the attested channel: not for Evidence
confidentiality (hence object-level encryption), not for session
integrity after eviction (hence key rotation that excludes it from
deriving post-eviction traffic keys), and not for Evidence appraisal.

The Participating Intermediary is trusted solely to correctly execute
its transport-layer routing obligations.  A Participating
Intermediary that fails to honour those obligations or that
suppresses eviction signalling causes session termination, providing
fail-secure behaviour without requiring the endpoints to detect
adversarial intent.

# Timing Models {#timing-models}

The timing and conveyance of attestation data relative to the
transport handshake define the two Attestation Timing Models used in
this architecture.

## Intra-Handshake Attestation

Evidence is conveyed by the Attester during the transport connection
establishment to the Relying Party within the handshake messages
themselves, prior to the transition to application data exchange.

The Relying Party, which may be deployed with a co-located Verifier,
appraises the Evidence in real time and makes an authorization
decision before the transport state machine permits application data
to flow.

Evidence is carried within standard transport protocol extensions
during the authentication phase of the handshake.

## Post-Handshake Attestation

Evidence is conveyed by the Attester after transport connection
establishment to the Relying Party following the transition to
application data exchange.

The Attestation Binder is derived after handshake completion,
tying the Evidence to the completed session.

This deployment can be localized to a sidecar 
that withholds application data until attestation completes, decoupling attestation
protocol from application logic.
the attestation protocol.

## Combining Timing Models

The two timing models are not mutually exclusive and their
combination is the natural architecture for deployments requiring
both immediate trust establishment and durable session integrity over
long-lived connections.

In this composition, intra-handshake attestation establishes baseline
trust before the session becomes usable: the Relying Party's Verifier
must accept the Attester's Evidence before application data can flow.
The combined model suits constrained device and IoT deployments where
a single attestation protocol handles both initial session trust and
ongoing re-verification, avoiding separate code paths for onboarding
and normal operation.

Protocol specifications building on this architecture MAY support one
or both timing models.

# Attestation Session Binding {#channel-binding-pattern}

Regardless of which timing model is used or which transport protocol
is in use, a correctly bound attested channel requires that three
conditions hold in sequence.

The first condition is Session Binding Value establishment.  The
endpoints must derive or obtain a shared, session-specific Session
Binding Value from which Attestation Binders can be derived.  The
Session Binding Value is bound to the specific session instance by
construction, and may be public (for example, a handshake transcript)
or secret (for example, an exporter-derived value).

The second condition is directional Binder derivation.  From the
Session Binding Value, the protocol derives distinct Attestation
Binders for the initiator and the responder.  The binders are directional:
the initiator's binder cannot be substituted for the responder's and vice
versa.  This ensures that Evidence produced by one endpoint cannot
satisfy the verification requirement for the opposite endpoint, even
within the same session.

The third condition is channel binding to Evidence.  The Attesting
Environment signs its directional Attestation Binder into its
Evidence payload, committing that Evidence to this specific session.
This condition addresses two distinct concerns.

The first is replay across sessions.  Because the Session Binding
Value is unique to the session, Evidence committed to a binder
derived from it cannot be presented in a different session.  Where
the Session Binding Value is secret, only the session participants can
derive it.  Where it is public, for example, a handshake transcript,
its uniqueness follows from the ephemeral keying material that the
transport establishes per session, so the transcript, and hence the
binder, cannot recur across sessions.

The second is a Key Substitution Attack: valid Evidence produced by a
genuine attested execution environment is presented while the Subject
Key used for authentication was not generated or protected
within that environment.  Session binding alone does not bind the
Subject Key to the attested environment; this is handled at the RATS
layer, as discussed under Key Non-exportability in
{{security-considerations}}.

The Evidence itself plays a critical role in verifying that these three
session binding conditions have been successfully achieved. Beyond the
cryptographic inclusion of the Attestation Binder, strict requirements
for the internal structure and the application of logical safeguards
protecting the Evidence are necessary to provide assurance that the
Evidence could not have been generated through alternative means such
as side-channel exploits.

When all three conditions are met, the channel-binding check may be
performed either by the Relying Party itself or by the Verifier.  As a
session participant, the Relying Party holds the Session Binding Value
and can derive the expected Attestation Binder locally, rejecting
Evidence whose binder does not match.  Alternatively, the Relying
Party conveys the binder to the Verifier, which checks it against the
binder in the Evidence.

# Freshness

The freshness of Evidence is critical to its value as a
trustworthiness signal.  In the transport context, freshness has
several distinct scopes that must be addressed separately.

## Per-session freshness

Per-session freshness ensures that Evidence is bound to the specific
session being evaluated and cannot be replayed from a prior session.
This property is addressed directly by the session binding mechanism
of {{channel-binding-pattern}}.  The Session Binding Value is specific
to the session
and cannot be known before the session is initiated, providing
nonce-style freshness in the sense of {{RFC9334}} Section 10.
Evidence committed to an Attestation Binder derived from the Session
Binding Value is therefore intrinsically fresh with respect to the
session: a replay from a different session will carry an Attestation
Binder derived from a different Session Binding Value, and appraisal
will fail.

## Session resumption

Session resumption introduces a specific freshness consideration.
When a transport session is resumed, previously obtained Attestation
Results may no longer reflect the Attester's current state.

## Re-Attestation in Long-Running Sessions

Initial attestation at Connection Establishment Time addresses
the architectural invariants the Relying Party's policy
requires before application data may flow.  Re-attestation
addresses the dynamic reality that established sessions may
outlast the validity of a single trust assessment.  Protocol
specifications building on this architecture SHOULD treat
these as distinct concerns.

Per-session freshness ensures Evidence cannot be replayed across
sessions but does not address changes in the Attester's state
during the Lifetime of Connection. A Relying Party MAY require
Re-attestation before continuing to transmit sensitive data to a
peer whose trust assessment has expired or whose deployment
environment may have changed in ways material to its policy.

Re-attestation does not retroactively protect data transmitted
before a state change occurred.  It bounds further exposure by
conditioning continued sensitive data transmission on a current
trust assessment.  Whether to terminate a session upon
re-attestation failure or continue with reduced privilege is
a matter of Relying Party policy.

# Privacy Considerations

## Evidence Payload Confidentiality

The Evidence payload carries Claims about the Attester's state and is
the most privacy-sensitive artifact in the protocol.  In deployments
involving a Participating Intermediary, the intermediary has Layer 7
visibility into the transport connection during the handshake phase
and would ordinarily be able to read unprotected message content.
Evidence payloads SHOULD be protected using object-level encryption
to a key held exclusively by the intended recipient, ensuring that
the Evidence content is inaccessible to the Participating
Intermediary regardless of its network position.  This protects
against the eavesdropper and intermediary threat surface; object-
level encryption provides this protection even when TLS is terminated
at the proxy boundary.

The complementary control for the Relying Party surface is
Attestation Result minimization: the Attestation Result returned to
the Relying Party SHOULD NOT re-expose sensitive Claims that were
protected in the encrypted Evidence.  Encrypting Evidence for the
Verifier without minimizing the Attestation Result shifts rather than
eliminates the disclosure risk.  A framework for consistent handling
of sensitive Evidence across RATS roles, including claim
classification, Trusted Verifier management, and Attestation Result
minimization, is provided in {{I-D.ounsworth-rats-privacy-framework}}.

## Transport Metadata

The transport connection discloses metadata — IP addresses, server
name indications, and connection timing — that is visible to the
Participating Intermediary and to passive network observers.  This
disclosure is inherent to the transport protocol and is not specific
to the attestation layer.

## Attestation Key Correlation

When the same attestation signing key is used across multiple
sessions, any party with access to Evidence from more than one of
those sessions can correlate the sessions to the same Attesting
Environment.  This linkability consideration is particularly relevant
for client Attesters where privacy of individual connections is a
concern.

## Anonymous Client Attestation

The SEAT architecture supports deployments where a client Attester
attests to the trustworthiness of its Attesting Environment without
presenting a TLS client identity certificate, enabling anonymous
client attestation.  In this deployment, the Relying Party's
appraisal policy applies to the client's hardware and software state
rather than to a disclosed identity.

## Scope Boundary and Internet Openness

The IAB has issued a statement cautioning that using client
attestation as a barrier to access for otherwise open protocols and
services risks undermining Internet openness {{IAB-Attestation-Risks}}.
The statement distinguishes services with intentionally restricted
access — for which client attestation is recognized as a valuable
security measure — from openly accessible services, for which
imposing hardware or software requirements on participating
implementations is inappropriate.  SEAT is scoped to the former
category: the use cases motivating this work involve confidential
workloads, enterprise-controlled environments, and TEE-backed
services where access is explicitly conditioned on verified platform
state.

The IAB statement further identifies the disclosure of vendor-
specific hardware and software information as a distinct risk:
attestation evidence that reveals which specific implementations are
in use can restrict access and enable tracking in ways that undermine
the open internet.  Protocol designs building on this architecture
should minimize vendor-specific claim disclosure consistent with the
Attestation Result minimization controls described in this section
and in {{I-D.ounsworth-rats-privacy-framework}}.

# Security Considerations {#security-considerations}

This section enumerates the security properties and considerations
of the SEAT architecture.  Security goals state outcomes the
architecture is designed to achieve; they carry no normative
mandates.  Security properties state technical characteristics the
protocol is expected to exhibit and may carry normative
requirements.  Implementations MUST also consider the Security
Considerations of {{RFC9334}} and of any protocol specification that
instantiates this architecture.

**Cryptographic Session Binding and Relay Prevention.** Evidence
presented on a session MUST be cryptographically bound to that
session and to the endpoint role in which it is presented. This is
achieved by binding the Evidence to an Attestation Binder derived
from a Session Binding Value that is specific to the session and
cannot be known before the session is initiated. Consequently,
valid Evidence from one session cannot satisfy a Verifier on a
different session or from a different endpoint; a replay carries
a Binder derived from a different Session Binding Value and MUST
be rejected. See {{channel-binding-pattern}}.

**Evidence Freshness.** Evidence reflects the Attester's state at or
near the Evidence Generation Time for the session in which it is
presented.  Per-session freshness ensures Evidence from a prior
session cannot be replayed against a new one.  When re-attestation
occurs during a session's Lifetime of Connection, the re-attestation
Evidence reflects the Attester's state at the time of re-attestation,
not at Connection Establishment Time.

**Evidence Confidentiality from Intermediaries.** When a Participating
Intermediary is present, Evidence payloads SHOULD be protected by
object-level encryption to a key held exclusively by the intended
recipient.  See {{I-D.ounsworth-rats-privacy-framework}}.

**Key Non-exportability (informative).** The specific concern of
demonstrating that the Subject Key used for transport authentication
is physically confined within the attested execution environment is
addressed at the RATS layer by {{I-D.reddy-rats-key-binding}} and is
not re-specified here.

**Session Resumption.** When a transport session is resumed, previously
obtained Attestation Results may no longer reflect the Attester's
current state.  Attestation from a prior session does not carry over
to a resumed session.

**Directional Endpoint Binding.** Distinct Attestation Binders MUST be
derived for the initiator and the responder from the same Session
Binding Value using distinct inputs.  Evidence produced by one
endpoint MUST NOT
satisfy the verification requirement for the opposite endpoint.  See
{{channel-binding-pattern}}

**Transmission and Verification Anchor Soundness.** An Attestation
Binder may be included in a transport message before peer
authentication is complete (the Transmission Anchor).
Implementations MUST ensure the transport protocol's integrity
guarantee covers the message carrying the Attestation Binder; for
example, the TLS 1.3 handshake MAC (the Verification Anchor)
retroactively guarantees the Binder's integrity at handshake
completion.

**Downgrade Prevention.** Two endpoints that both support attestation
cannot be caused by an active adversary to negotiate a connection
without it.  The negotiation of attestation capabilities is protected
against suppression.

**Dynamic Verification Code Integrity.** When client-side attestation
verification logic is dynamically delivered by the endpoint under
appraisal (such as browser-based JavaScript), a circular trust
dependency exists.  Unless the client's execution environment
enforces an independent, orthogonal guarantee of code integrity
and binary transparency, Application-layer attestation cannot
provide security assurance, as the Attester may serve malicious
code that bypasses cryptographic validation.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The authors wish to thank Usama Sardar, Yuning Jiang, and Meiling Chen
for their thoughtful input and contributions that influenced this document.

TODO
