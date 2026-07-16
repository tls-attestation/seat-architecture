---
title: "Secure Evidence and Attestation Transport (SEAT) Architecture"
abbrev: "SEAT Architecture"
category: info

docname: draft-many-seat-architecture-latest
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
  github: "tls-attestation/seat-architecture"
  latest: "https://tls-attestation.github.io/seat-architecture/draft-seat-architecture.html"

author:
  - fullname: Nathanael Ritz
    organization: Independent
    email: "ietf@nritz.com"
  - fullname: Thomas Fossati
    organization: Independent
    email: "tho.ietf@gmail.com"
  - fullname: Tirumaleswar Reddy
    organization: Nokia
    email: "kondtir@gmail.com"
  - fullname: Ionuț Mihalcea
    organization: Arm
    email: "ionut.mihalcea@arm.com"

normative:
  RFC2119:
  RFC8174:
  RFC9334:
  I-D.mihalcea-seat-use-cases:

informative:
  I-D.ounsworth-rats-privacy-framework:
  I-D.reddy-rats-key-binding:
  I-D.ietf-rats-epoch-markers:
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
given transport connection.


--- middle

# Introduction

## Establishing Trust in Secure Communications

> "Cryptography *without system integrity* is like investing in an
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

Connection Signing Key (`CSK`):
: The asymmetric identity key pair used by the endpoint to authenticate 
  the transport handshake (e.g., signing the TLS CertificateVerify message). 
  A `CSK` may be natively generated and maintained exclusively within the 
  Target Environment (providing non-exportability guarantees), or it MAY be 
  provisioned into the Target Environment from an external certificate manager 
  or HSM. Evidence produced by the Attesting Environment MUST include a binding 
  to the `CSK`.

Ephemeral KEM Key (`EKK`):
: An asymmetric Key Encapsulation Mechanism (KEM) key pair generated
  and maintained exclusively within the Target Environment. The
  Attester uses the `EKK` to decapsulate session-specific challenges.

  As a per-session liveness anchor, the `EKK` is ephemeral and closely tied
  to the lifecycle of a single Attestation Result; its rotation is
  governed by freshness boundaries (e.g., temporal nonces such as Epoch
  Markers distributed by an Epoch Bell {{I-D.ietf-rats-epoch-markers}}).
  Regularly rotating the `EKK` in this role strictly bounds the
  vulnerability window of an exfiltrated key to the current validity
  period.

> The Verifier includes the public `EKK` within the Attestation Result,
> allowing the Relying Party to cryptographically bind the credential
> to the current session and enforce per-session freshness
> (see {{per-session-freshness}}).

Long-Term Key (`LTK`):
: A long-lived asymmetric key pair representing the host or hardware-level 
  authority, typically residing outside the Target Environment in a Hardware 
  Security Module (HSM). In multi-tenant or containerized environments, an 
  `LTK` MAY delegate transport signing authority to an ephemeral, per-workload 
  `CSK` via a Delegated Credential.

Attesting Environment Key (`AEK`):
: The asymmetric key used by the Attesting Environment
  to sign Evidence. The Verifier trusts the `AEK` through an
  Endorsement chain that typically roots in a hardware manufacturer
  or a device‑specific CA. The `AEK` is used solely for attestation
  and is distinct from any key used for transport authentication.

Attestation Result Key (`ARK`):
: The asymmetric key used by a Verifier to sign
  Attestation Results. The Relying Party must possess the
  corresponding trust anchor for the `ARK` so that it can verify
  the integrity and authenticity of received Attestation Results.

Object Security Key (`OSK`):
: An asymmetric public key belonging to the Verifier, used by the 
  Attesting Environment to apply object-level encryption (e.g., HPKE) 
  to Evidence payloads before transport transmission.

Attestation Credential:
: The attestation payload conveyed by the Attester to the Relying
  Party across the transport connection. Depending on the RATS
  conveyance model in use, this payload consists of either Evidence
  (Background-Check Model) or an Attestation Result (Passport Model).
  Where a statement applies specifically to one but not the other,
  this document uses the more specific term.

> While Evidence is exclusively generated by the Attester and
> Attestation Results are generated by a Verifier, in the SEAT
> transport context, the Attestation Credential is always presented
> by the Attester directly to the Relying Party over the secure channel.

> An Attestation Result MAY be cached by the Attester and reused
> across multiple subsequent sessions rather than obtained anew for
> each session. Where per-session freshness is required for a cached
> Attestation Result, this is established separately from the
> Result's issuance; see {{interactive-passport}}.

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
  snapshot of state at the exact moment of signing the evidence.
  In all cases, it makes no representation about the Attester's
  state at any later time.

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

# Roles and Entities

The SEAT architecture maps the roles defined in {{RFC9334}} to standard
transport protocol entities.  The subsections below describe each
role and its specific character in the transport context.

The overarching SEAT goal is to establish an Attested Channel between two
entities.  {{fig-atls}} shows the TLS and RATS roles that are involved in
achieving this goal, and how they interact.

~~~ aasvg
{::include img/arch.ascii-art}
~~~
{: #fig-atls artwork-align="center" title="Attested Secure Channel"}

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
and the Attesting Environment is a security-critical boundary.
See {{security-considerations}}.

In mutual attestation deployments, both the Client and the Server
simultaneously act as Attesters.  Each endpoint's Attesting
Environment independently generates Evidence bound to the session.

## Relying Party

The Relying Party consumes an Attestation Result and uses
it to make authorization decisions about the transport connection.
In the transport context, the Relying Party is typically the endpoint
opposite the Attester — the Server when the Client attests, or the
Client when the Server attests.

## Verifier

The Verifier appraises the validity of Evidence and produces
Attestation Results, as defined in {{Section 4 of RFC9334}}.

The appraisal is driven by an Appraisal Policy for Evidence,
a set of rules that determines which Endorsements and Reference
Values are required, which Claims must be present, and under what
conditions Evidence is considered acceptable.

The Appraisal Policy may be configured as part of the Verifier’s
trust anchors or supplied by a Relying Party in a
deployment-specific manner. When Attestation Results are produced,
they reflect the outcome of applying that policy.

How Evidence reaches the Verifier follows one of the two RATS
conveyance models ({{Section 5 of RFC9334}}):

Background-Check Model:
: The Relying Party conveys the Attester's Evidence to the Verifier
  and receives Attestation Results in return.  The Verifier may be
  co-located with the Relying Party, appraising Evidence inline, for
  example during an intra-handshake exchange that requires a real-time
  result before the connection is finalized or operated as a remote
  service.

Passport Model:
: The Attester conveys its Evidence to a remote Verifier, obtains
  Attestation Results, and presents those Results to the Relying
  Party.

Verifier location is an independent deployment choice: a co-located
Verifier operates under the Background-Check Model, whereas a remote
Verifier may operate under either model.

{{fig-roles}} illustrates how Evidence and Attestation Results flow
under the two conveyance models.

~~~ aasvg
 Background-Check Model
 (the Relying Party conveys Evidence to the Verifier; the
  Verifier may be co-located with the Relying Party or remote)

  +----------+ Evidence  +---------------+ Evidence  +----------+
  | Attester |---------->| Relying Party |---------->| Verifier |
  |          |           |               |<----------|          |
  +----------+           +---------------+ Att.Res.  +----------+

 Passport Model
 (the Attester conveys Evidence to the Verifier and presents
  the resulting Attestation Results to the Relying Party)

  +----------+ Evidence  +----------+
  | Attester |---------->| Verifier |
  |          |<----------|          |
  +----------+ Att.Res.  +----------+
        |
        | Attestation Results
        v
  +---------------+
  | Relying Party |
  +---------------+
~~~
{: #fig-roles title="RATS Conveyance Models in the Transport Context"}

# Trust Model

This section describes the trust relationships required to establish
an Attested Channel.  The general trust model of {{RFC9334}} Section 7
applies; the subsections below specialise it for the transport
context.

## Relying Party Trust

The Relying Party must trust that the Attestation Credential it receives
accurately reflects the Attester's state, which depends on its trust
in the Verifier and in the Endorsement chain for the Attesting
Environment.

The Relying Party must additionally satisfy itself that
the Attestation Credential is bound to the current session — that it
has not been replayed from a different session or transferred from a
different endpoint.  This assurance is provided by the session binding
mechanism described in {{channel-binding-pattern}}; the check may be
performed by the Relying Party itself or delegated to the Verifier,
but it cannot be pre-computed independently of the session.

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
own requirements before accepting any Attestation Credentials.

# Timing Models {#timing-models}

The timing and conveyance of Attestation Credentials relative to the
transport handshake define the two Attestation Timing Models used in
this architecture.

Depending on the approach, an Attestation Credential may be conveyed
during Intra-Handshake Window or conveyed at the application layer in
the Post-Handshake Window.

If the credential is Evidence, the Relying Party acts as or forwards
it to a Verifier to appraise the Evidence. If the credential is an
Attestation Result, the Relying Party evaluates it against its own
Appraisal Policy for Attestation Results.

In both cases, an authorization decision must be made before the
transport state machine permits application data to flow.

## Intra-Handshake Attestation

An Attestation Credential is conveyed by the Attester **during** the
transport connection establishment to the Relying Party within the
handshake messages themselves, prior to the transition to application
data exchange. Upon receipt, the Relying Party processes the
Attestation Credential.

The Relying Party, which may be deployed with a co-located Verifier,
appraises the Evidence in real time and makes an authorization
decision before the transport state machine permits application data
to flow.

## Post-Handshake Attestation

An Attestation Credential is conveyed by the Attester **after** transport
connection establishment to the Relying Party following the transition to
application data exchange.

The Attestation Binder is derived after handshake completion,
tying the Attestation Credential to the completed session.

This deployment can be localized with the sidecar pattern, which
withholds application data until the attestation procedure completes,
decoupling the attestation protocol from application logic.

## Combining Timing Models

The two timing models may also be used together and their combination
is the natural architecture for deployments requiring both immediate
trust establishment and durable session integrity over long-lived
connections.

In this composition, intra-handshake attestation establishes baseline
trust before the session becomes usable: the Relying Party's Verifier
must accept the Attester's Attestation Credential before application
data can flow. The combined model suits constrained device and IoT
deployments where a single attestation protocol handles both initial
session trust and ongoing periodic re-attestion, avoiding separate
code paths for onboarding and normal operation.

Protocol specifications building on this architecture MAY support one
or both timing models.

# Failure handling considerations {#failure-handling}

## Failure handing within Intra-Handshake Window

When remote attestation occurs within the Intra-Handshake Window,
the transport handshake withholds progression to application data
exchange until the Attestation Result is available: application
data exchange has not yet begun. A Verifier rejection, or a
Relying Party policy rejection of an otherwise valid Attestation
Result, MUST result in a fatal error consistent with the transport
protocol's existing handshake-failure handling.

It is RECOMMENDED that the failure mode be interpretable by the
application as a remote-attestation-related fault. Remote attestation
specificity provides greater flexibility to apply application-layer
policies, and assists in auditing and general debugging.

## Failure handing within Post-Handshake Window

When attestation occurs within the Post-Handshake Window, or when
Re-attestation fails during the Lifetime of Connection, the
transport session already exists and application data may already
be flowing.  {{RFC9334}} expects a failed Attester appraisal to
result in reduced access or privileges rather than outright
rejection.  In the event of failures occurring within the
Post-Handshake Window, this behaviour MAY be handled at
the transport layer.

As the Relying Party's enforcement point sits outside the
transport handshake, operating on already-established
application-layer traffic, the Appraisal Policy determines
whether the connection is torn down, or restricted to a subset
of application-layer functionality. Failure handling of
Post-Handshake Attestation does not retroactively protect
application data already exchanged prior to the failed appraisal;
it bounds further exposure going forward.

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

The third condition is channel binding to an Attestation Credential. The
Attesting Environment signs its directional Attestation Binder into its
Evidence payload, committing that Evidence to this specific session.

For this condition to hold when using the Passport model, the Verifier
must propagate this binding into the resulting Attestation Result,
ensuring the final Attestation Credential presented to the Relying Party
remains committed to the specific transport session.

The first is replay across sessions.  Because the Session Binding
Value is unique to the session, an Attestation Credential committed
to a binder derived from it cannot be presented in a different session.
Where the Session Binding Value is secret, only the session participants
can derive it.  Where it is public, for example, a handshake transcript,
its uniqueness follows from the ephemeral keying material that the
transport establishes per session, so the transcript, and hence the
binder, cannot recur across sessions.

The second is a Key Substitution Attack: a valid Attestation Credential
produced by a genuine attested execution environment is presented while
the Subject Key used for authentication was not generated or protected
within that environment.  Session binding alone does not bind the
Subject Key to the attested environment; this is handled at the RATS
layer, as discussed under Key Non-exportability in
{{security-considerations}}.

The Attestation Credential itself plays a critical role in verifying
that these three session binding conditions have been successfully
achieved. Beyond the cryptographic inclusion of the Attestation Binder,
strict requirements for the internal structure and the application of
logical safeguards protecting the Attestation Credential are necessary
to provide assurance that the Attestation Credential could not have been
generated through alternative means such as side-channel exploits.

When all three conditions are met, the channel-binding check may be
performed either by the Relying Party itself or by the Verifier.  As a
session participant, the Relying Party holds the Session Binding Value
and can compute the binder locally and MAY send it to the Verifier
which compares it with the binder in the Evidence, avoiding the
need requiring that the Relying Party decode the Evidence first.

## Background-Check Conveyance Model

If the Relying Party is directly consuming Evidence (Background-Check
model), it rejects Evidence whose binder does not match.

A Background-Check exchange MAY also be used to provision an `EKK`
for subsequent use under the Passport model, by minting a cached
Attestation Result that binds a Trust-On-First-Use `EKK` for later
Interactive Passport challenges (see {{interactive-passport}}).

## Passport Conveyance Model

If the Relying party is consuming an Attestation Result (Passport model)
and expects per-session freshness (see {{per-session-freshness}}), it
MUST reject the Attestation Result if it cannot affirmatively evaluate
that the Verifier explicitly tied the Attestation Result to the current
session's Attestation Binder.

### Interactive Passport Conveyance {#interactive-passport}

To fulfill these binding conditions without requiring the Attester to
contact the Verifier during connection establishment, deployments
utilizing the Passport model MAY employ an interactive challenge.
By encapsulating a session-specific challenge to an `EKK` whose public
component is bound within the Attestation Result, the Relying Party
can verify the Attester's active control over the Target Environment.
The decapsulated secret is subsequently mixed into the Session Binding
Value, cryptographically tying the cached credential to the active
transport session.

~~~ aasvg
{::include img/interactive-passport.ascii-art}
~~~
{: #fig-passport artwork-align="center" title="Per-Session Freshness with Interactive Passport Challenge"}


# Freshness

The freshness of Evidence is critical to its value as a
trustworthiness signal.  In the transport context, freshness has
several distinct scopes that must be addressed separately.

## Per-session freshness {#per-session-freshness}

Per-session freshness ensures that Evidence is bound to the specific
session being evaluated and cannot be replayed from a prior session.
This property is addressed directly by the session binding mechanism
of {{channel-binding-pattern}}.  The Session Binding Value is specific
to the session and cannot be known before the session is initiated,
providing nonce-style freshness in the sense of {{Section 10 of RFC9334}}.
Evidence committed to an Attestation Binder derived from the Session
Binding Value is therefore intrinsically fresh with respect to the
session: a replay from a different session will carry an Attestation
Binder derived from a different Session Binding Value, and appraisal
will fail.

For deployments utilizing cached Attestation Results (the Passport model),
per-session freshness of the transport connection must be decoupled from
the generation time of the credential itself. This is achieved by binding
the lifecycle of the `EKK` to a strict freshness boundary, such as an Epoch
Marker {{I-D.ietf-rats-epoch-markers}}. While the transport nonces provide
network freshness, the successful decapsulation of an `EKK` challenge proves
the Attesting Environment is operating within the acceptable epoch window,
mitigating the risk of indefinitely replaying exfiltrated credentials.

Where the `EKK` instead serves as a per-machine continuity anchor pinned
via Trust-On-First-Use, rotation occurs at an operator-configured interval
rather than solely in response to an Epoch Marker. In this case, each
successive `EKK` MUST be attested by the outgoing `EKK` at the point of
rotation, so that the Relying Party can verify an unbroken chain of
succession back to the originally pinned `EKK` without requiring a fresh
Background-Check exchange at every rotation boundary.

## Session resumption

Session resumption introduces a specific freshness consideration.
When a transport session is resumed, a previously obtained
Attestation Credential may no longer reflect the Attester's
current state.

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
a matter of Relying Party policy; see {{failure-handling}}.

# Privacy Considerations

## Evidence Payload Confidentiality

The Evidence payload carries Claims about the Attester's state and is
the most privacy-sensitive artifact in the protocol.  It is
RECOMMENDED that Evidence payloads be encrypted to a key
held exclusively by the intended recipient (typically the Verifier),
so that the Evidence content is disclosed only to that recipient and
not to the Relying Party or to other parties on the path.

The complementary control for the Relying Party surface is minimization:
the Attestation Results returned to the Relying Party SHOULD NOT
re-expose sensitive Claims that were protected in any encrypted Evidence.
A framework for consistent handling of sensitive Evidence across RATS roles,
including claim classification, Trusted Verifier management, and
Attestation Credential minimization, is provided in
{{I-D.ounsworth-rats-privacy-framework}}.

## Transport Metadata

The transport connection discloses metadata — IP addresses, server
name indications, and connection timing — that is visible to passive
network observers.  This disclosure is inherent to the transport
protocol and is not specific to the attestation layer.

## Attestation Key Correlation

When the same Attesting Environment Key (`AEK`) is used across multiple
transport sessions, it acts as a stable Attester Identifier. As
defined in {{I-D.ounsworth-rats-privacy-framework}}, any party with
access to Evidence from more than one of those sessions can use this
public key to correlate the sessions to the same Attesting
Environment. This linkability consideration is particularly relevant
for client Attesters where the privacy of individual connections is a
concern.

This linkability risk natively extends to the public Ephemeral KEM
Key (`EKK`) when an Attestation Result is cached and reused across
multiple sessions under the Passport conveyance model. In such deployments,
the static public `EKK` acts as a temporary cross-session tracking beacon,
necessitating mitigation controls like short epoch validity periods or
pairwise key generation to preserve client anonymity.

To mitigate this correlation risk, deployments SHOULD
consider the controls described in {{I-D.ounsworth-rats-privacy-framework}},
such as utilizing pairwise identifiers, short validity periods, or
unlinkable presentations for the `AEK`.

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
Attestation Credential minimization controls described in this section
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

**Cryptographic Session Binding and Relay Prevention.** An Attestation
Credential presented on a session MUST be cryptographically bound to that
session and to the endpoint role in which it is presented. This is
achieved by binding the Attestation Credential to an Attestation Binder
derived from a Session Binding Value that is specific to the session and
cannot be known before the session is initiated. Consequently, a valid
Attestation Credential from one session cannot satisfy a Verifier or
Relying Party on a different session or from a different endpoint;
a replay carries a Binder derived from a different Session Binding
Value and MUST be rejected. See {{channel-binding-pattern}}.

**Split-Deployments.** Because a compromised host could attempt to
use the Attesting Environment as a signing oracle by substituting
the attestation binder input, the architecture relies on
cryptographic binding rather than continuous state monitoring.
The Attesting Environment MUST bind the Attestation Credential
to the private identity key it holds to authenticate a connection
(for example, by including a hash of the associated public key in the
signed payload).

The Relying Party then verifies that this claim matches the
identity key presented in the transport handshake, preventing
an untrusted host from successfully substituting the binder.

Where the transport authentication key is a non-confidential `CSK`
provisioned directly into the Target Environment rather than being
natively generated, binding the Attestation Credential to that
key's public component alone does not establish that the key is
exclusive to the attested environment.

**Compound Authentication and Key Exclusivity.** A Relying Party's
appraisal policy dictates the required security properties of the
transport authentication key. An Attester uses a Connection Signing 
Key (`CSK`) to authenticate the transport session. If a deployment 
utilizes an injected `CSK` (lacking native non-exportability guarantees) 
AND the Relying Party's appraisal policy strictly requires Compound 
Authentication, then an Ephemeral KEM Key (`EKK`) generated natively 
within the Target Environment is REQUIRED to anchor the session and 
prevent binder substitution by an untrusted host holding the same 
injected `CSK`. Otherwise, standalone transport authentication via `CSK` 
is sufficient.

**Key Non-exportability (informative).** The specific concern of
demonstrating that the Subject Key used for transport authentication
is physically confined within the attested execution environment is
addressed at the RATS layer by {{I-D.reddy-rats-key-binding}} and is
not re-specified here.

**Evidence Freshness.** Evidence reflects the Attester's state at or
near the Evidence Generation Time for the session in which it is
presented.  Per-session freshness ensures Evidence from a prior
session cannot be replayed against a new one.  When re-attestation
occurs during a session's Lifetime of Connection, the re-attestation
Evidence reflects the Attester's state at the time of re-attestation,
not at Connection Establishment Time.

**Evidence Confidentiality.** Evidence payloads SHOULD be protected by
object-level encryption to an Object Security Key (`OSK`) held exclusively 
by the intended recipient. While `OSK` ensures privacy against passive 
observers, the cryptographic integrity and session-binding resistance 
of the attestation channel do not rely on `OSK` secrecy. See 
{{I-D.ounsworth-rats-privacy-framework}}.

**Session Resumption.** When a transport session is resumed, previously
obtained Attestation Credential may no longer reflect the Attester's
current state.

**Directional Endpoint Binding.** Distinct Attestation Binders MUST be
derived for the initiator and the responder from the same Session
Binding Value using distinct inputs.  Evidence produced by one endpoint
MUST NOT satisfy the verification requirement for the opposite endpoint.
See {{channel-binding-pattern}}.

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

# Implementing Transport Integration (informational) {#integration-patterns}

The Timing Models of {{timing-models}} describe when an Attestation
Credential is conveyed relative to connection establishment.  This
section describes two structural implementation examples by which a
transport protocol conveys an Attestation Credential to the Relying
Party without requiring the transport specification itself to encode
RATS semantics.

Depending on the conveyance model, the Relying Party either forwards
Evidence to a Verifier to receive an authorization decision
(Background-Check Model) or validates an Attestation Result directly
(Passport Model).

## Extension-Based Conveyance

In this pattern, the transport protocol's existing identity or
authentication structures (such as an X.509 certificate extension, or
a comparable protocol-specific extension point) are reused to carry an
Attestation Credential.  The transport stack itself remains unaware of
{{RFC9334}} semantics: it recognizes only that an extension it is
configured to process is present, and delegates interpretation of the
extension's contents to an external callback.

The transport state machine suspends progress at the point the
extension is processed, invokes the callback with the extension
payload, and resumes or aborts the handshake based on the callback's
return value.  The callback interface is transport-external: it need
not be specified by the transport protocol itself, only supported by
it as an extension point.

## Structured Payload Conveyance

In this pattern, the transport protocol defines a dedicated, opaque
field for authorization-related data as part of its handshake or
key-exchange messages, distinct from the identity structures used for
peer authentication.  The Attestation Credential, and any associated
attestation-specific protocol elements, are carried within this field.

The transport stack extracts the field's contents and passes them
to an adjacent component responsible for {{RFC9334}} semantics,
without needing to parse or understand the contents itself.  As in
Extension-Based Conveyance, the transport state machine halts
pending the outcome of this processing.  The distinction between
the two patterns is where the extension point is anchored: an
existing identity structure being overloaded (Extension-Based
Conveyance) versus a field purpose-defined by the transport
protocol for authorization data (Structured Payload Conveyance).

Both patterns satisfy the requirement that an Attestation Credential
be conveyed prior to the transition to application data exchange; the
choice between them depends on the target transport protocol's
extension model and is otherwise architecturally equivalent from an
{{RFC9334}} perspective.

# Acknowledgments
{:numbered="false"}

The authors wish to thank all SEAT WG participants for their
thoughtful input and contributions that have helped influence
this document.
