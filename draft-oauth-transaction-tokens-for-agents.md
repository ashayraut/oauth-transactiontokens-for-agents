---
title: "Transaction Tokens For Agents"
category: info

docname: draft-araut-oauth-transaction-tokens-for-agents-03
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - transaction tokens
 - agentic AI
 - multi-agent authorization
 - zero trust
venue:
#  group: WG
#  type: Working Group
#  mail: TBD
#  arch: TBD
  github: "ashayraut/oauth-transactiontokens-for-agents"
  latest: "https://ashayraut.github.io/oauth-transactiontokens-for-agents/draft-oauth-transaction-tokens-for-agents.html"

author:
 -
    fullname: ASHAY RAUT
    email: asharaut@amazon.com

normative:
  RFC2119:
  RFC8174:
  RFC6749:
  RFC7519:
  RFC7515:
  RFC8693:
  RFC9068:
  RFC9396:
  OAUTH-TXN-TOKENS:
    title: "OAuth Transaction Tokens"
    target: https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html
    author:
      - name: Atul Tulshibagwale
        org: SGNL
      - name: George Fletcher
      - name: Pieter Kasselman

informative:

...

--- abstract

This document specifies an extension to OAuth Transaction Tokens
[OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) to support agent context propagation within
Transaction Tokens for agent-based workloads. The extension defines
the `agentic_ctx` claim, a structured JWT claim that carries
chain-level metadata required for multi-agent flow integrity.

The extension addresses two deployment patterns: single-agent flows
where one agent acts on behalf of a user within a transaction, and
multi-agent flows where multiple agents collaborate across a call
chain.  In multi-agent scenarios, the Transaction Token is replaced
at each agent transition by the Transaction Token Service, updating
the `agentic_ctx` claim while preserving the immutable identity
context (`sub` and `act` claims) established at transaction
initiation.

By propagating agent chain context to every service in the call path,
this extension enables Resource Servers to implement authorization
policies that distinguish agentic execution paths from direct human
calls, and provides the transaction lineage required for fine-grained
audit trails at the point of data access.  Security and compliance
teams gain visibility into the downstream operational footprint of an
agentic workflow — the full set of services and data stores reached
under a single transaction — enabling post-incident reconstruction
and bounded assessment of agentic reach.

This specification leverages the existing `act` claim as defined in
[RFC8693](https://datatracker.ietf.org/doc/html/rfc8693) and the 
`sub` claim as defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).  It
does not define new token types or grant types; it operates entirely
within the existing Transaction Token issuance and replacement
mechanisms defined by the base specification.

--- middle

# 1. Introduction

AI agents operate autonomously across multi-step workflows, making
tool calls and accessing data stores without direct human oversight
at each step.  This creates three specific gaps that existing
authorization infrastructure does not address.  First, authorization
services at the data layer have no mechanism to distinguish a request
that originated from a direct human API call from one that traveled
through an autonomous agent chain several hops deep.  Second, audit
logs capture individual operations but lose the execution context
that explains why those operations occurred and on whose behalf.
Third, security and compliance teams have no way to bound or
reconstruct the downstream operational footprint — the agentic reach
— of a given agentic workflow after the fact.

Transaction Tokens (Txn-Tokens) are short-lived, signed JSON Web
Tokens [RFC7519] that convey identity and authorization context.
However, the current Txn-Token format lacks sufficient context for
services within the call chain to implement fine-grained access
control policies for agent-based workflows.  Specifically, it does
not carry the agent chain-level context needed to distinguish agentic
execution paths from direct calls, limiting both authorization
granularity and audit traceability in multi-agent deployments.

This document defines the `agentic_ctx` claim, a structured extension
within a Transaction Token that carries chain-level metadata for
multi-agent flow integrity.  It also specifies how the existing `act`
claim (defined in [RFC8693]) and `sub` claim (defined in
[OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html)) are used in agent-based transaction flows.

The extension introduces a two-layer model for agent transaction
context:

*  Identity Layer: The `sub` claim (representing the subject of the
   transaction) and the `act` claim (representing the agent that
   obtained the original access token) are immutable throughout the
   transaction lifetime.  These claims are populated per the rules
   defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) and [RFC8693](https://datatracker.ietf.org/doc/html/rfc8693) respectively.

*  Context Layer: The `agentic_ctx` claim carries chain-level
   metadata required for multi-agent flow integrity: the originating
   and proximate agent identities, hop tracking, and an optional
   minimum assurance level.  Resource Servers requiring richer context
   (such as operational posture or provenance) SHOULD query the Agent
   Attribute Store directly using the agent identifiers carried in
   the token, rather than relying on token-embedded attributes.

By propagating this context to every service in the call path, the
extension enables three capabilities:

*  Audit closer to the data: Internal data stores and microservices
   can log fine-grained audit trails that reflect whether a specific
   operation originated from a direct human call or from an autonomous
   agentic execution path.  The `originator.id`, `proximate_agent.id`,
   and `hop_count` fields carried in every Txn-Token provide the
   execution context needed for meaningful audit entries at the point
   of data access, not just at the API gateway.

*  Agentic reach visibility: By preserving transaction lineage across
   the full call chain, the `txn` claim combined with the TTS audit
   log (Section 3.9) enables post-incident reconstruction of the
   complete execution path and the set of resources accessed under a
   single transaction.  This gives security and compliance teams a
   bounded, auditable view of the operational footprint of any agentic
   workflow.

*  Context-aware authorization: Resource Servers can enforce policies
   that evaluate the nature of the calling chain, not just the
   identity of the immediate caller.  Policies that permit a direct
   human call but restrict or audit an equivalent call from an
   autonomous agent chain are expressible using `originator.id`,
   `proximate_agent.id`, and `min_assurance_level` without changes to
   the underlying resource implementation.

To support richer context population, the TTS SHOULD have access to
an Agent Attribute Store or equivalent mechanism that enables it to
distinguish agent callers from non-agent workload callers and to
source agent metadata within the trust domain.  When the TTS
identifies a caller as an agent, it applies agent-specific token
issuance rules as defined in this specification.  When the caller is
not identified as an agent, the TTS follows the base specification
rules without modification.  Deployments without an Agent Attribute
Store MAY still populate `agentic_ctx` using information derivable
from the token exchange flow itself (e.g., `proximate_agent.id` from
`client_id`, `hop_count` from replacement count).

This extension leverages the existing Txn-Token infrastructure to
enable secure propagation of AI agent context throughout the
service graph.


## 1.1. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119) [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174) when, and only when, they appear in all capitals, as shown here.

# 2. Terminology

Agentic-AI: AI Agentic applications are software applications that utilize
Large Language Models (LLM)s and plans, reasons,and takes actions independently
to achieve complex, multi-step goals with minimal human oversight.

Workload:
An independent computational unit that can autonomously receive and process
invocations, and can generate invocations of other workloads.
Examples of workloads include containerized microservices,
monolithic services and infrastructure services such as managed databases.

Trust Domain:
A collection of systems, applications, or workloads that share a
common security policy. In practice this may include a virtually or
physically separated network, which contains two or more workloads.
The workloads within a Trust Domain may be invoked only through published
interfaces.

Call Chain:
A sequence of synchronous invocations that results from the invocation of an external endpoint.

External Endpoint:
A published interface to a Trust Domain that results in the invocation
of a workload within the Trust Domain. This is the first service in the
call chain where request starts.

Transaction Token (Txn-Token):
A signed JWT with a short lifetime, providing immutable information about the user or workload,
certain parameters of the call, and specific contextual attributes of the call.
The Txn-Token is used to authorize subsequent calls in the call chain.

Transaction Token Service (Txn-Token Service):
A special service within the Trust Domain that issues Txn-Tokens to requesting
workloads. Each Trust Domain using Txn-Tokens MUST have exactly one logical
Txn-Token Service.

Agent Attribute Store:
A logical role — not a defined protocol interface — that MAY be realized as
an agent registry, service catalog, policy engine, or any equivalent
mechanism accessible within the trust domain.  It serves two functions: an
identity catalog (recording which workloads are agents and their assigned
assurance levels) and an attribute store (recording additional attributes
such as operational posture, attestation status, and provenance metadata).
The TTS MAY use the Agent Attribute Store to determine whether a requesting
workload is an agent and to source the assurance level used in
`min_assurance_level` calculation.  Resource Servers MAY query the Agent
Attribute Store using the agent identifier from `proximate_agent.id` or
`originator.id` to obtain richer context for fine-grained authorization
decisions.  The presence of an Agent Attribute Store is RECOMMENDED but
not required.

Note on terminology: In IETF and IANA practice, the term "registry"
conventionally refers to a namespace or schema catalog of permitted
identifiers and values (e.g., an IANA registry of claim names), not a
runtime instance store.  This specification uses "Agent Attribute Store"
to avoid that collision and to describe the runtime, instance-level nature
of this component.  Deployments that use a system commonly referred to as
an "agent registry" or "agent catalog" MAY treat it as fulfilling this role
provided it supports the attribute query functions described in
Section 3.7.2.

Proximate Agent:
The agent whose request caused the current Transaction Token to be
issued or most recently replaced.  In a multi-agent chain, this is
the agent at the current hop — the one closest to the resource
being accessed at the time of the token exchange.  Represented in
the `proximate_agent` field of `agentic_ctx`.

Originating Agent:
The agent that initiated the transaction chain by causing the
initial Transaction Token to be issued.  This agent's identity is
preserved immutably throughout all replacement flows.  Represented
in the `originator` field of `agentic_ctx`.

Inference Layer:
The component of an agent system that runs the AI model (e.g., an
LLM), processes inputs, generates plans, and produces tool calls or
outputs.  The inference layer operates on data presented to it and
has no inherent awareness of the security context of the surrounding
system.

Infrastructure Layer:
The component of an agent system that executes tool calls, manages
network I/O, enforces security policies, and interacts with external
services including the Transaction Token Service.  The infrastructure
layer is responsible for token acquisition and exchange and operates
independently of the inference layer's outputs.


# 3. Protocol overview

## 3.1 Transaction Flow

   This section describes the process by which an agent application
   obtains a Transaction Token, either acting autonomously or on behalf
   of a principal. The external endpoint requests a Txn-Token following
   the procedures defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), augmented with additional
   context for agent identity and, when applicable, principal identity.

## 3.2 Agent Application Transaction Flows

   The Transaction Token creation process varies depending on the
   presence of a principal.

### 3.2.1 Subject-Initiated Flow

When a human subject initiates the workflow, the following steps
occur:

1. The subject invokes the agent application to perform a task.

2. The agent application calls an external endpoint.  The external
   endpoint returns an OAuth challenge.

3. The agent application authenticates using an OAuth 2.0
   Authorization Code flow [RFC6749](https://tools.ietf.org/html/rfc6749) access token.  The access token
   contains `sub` and `client_id` claims as per [RFC9068].  If the
   access token represents a delegated flow (human delegating to
   agent), it MAY contain an `act` claim per [RFC8693].

4. The external endpoint submits the received access token along with
   its subject token to the Txn-Token Service.  Subject token
   requirements are specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).

5. The Txn-Token Service validates the access token.

6. The Txn-Token Service populates the Txn-Token's `sub` claim
   following the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).  The `sub`
   claim is determined based on the `subject_token` provided in the
   request.

7. If the access token contains an `act` claim, the Txn-Token Service
   copies the `act` claim value to the Txn-Token's `act` field.  The
   value is copied as-is without modification.

8. If the TTS has access to an Agent Attribute Store or equivalent mechanism,
   it performs a lookup using the `client_id` from the access token.
   If the `client_id` resolves to a registered agent, the TTS
   populates the `agentic_ctx` claim. This is the case when agent is within trust domain and
   user start request using agent within trust domain. If no Agent Attribute Store is
   available, the TTS MAY still populate `agentic_ctx` using
   information derivable from the token exchange (e.g., `client_id`
   as `proximate_agent.id`).

10. The Txn-Token Service issues the Txn-Token to the requesting
   workload.


```
+--------+     +-------+     +----------+     +-----+     +----------+
| Human  |     |  3P   |     | External |     | TTS |     |  Agent   |
| Subject|     | Agent |     | Endpoint |     |     |     |Attr Store|
+---+----+     +---+---+     +----+-----+     +--+--+     +----+-----+
    |              |               |              |              |
    | 1. Invoke    |               |              |              |
    |  task        |               |              |              |
    |------------->|               |              |              |
    |              |               |              |              |
    |              | 2. Call with   |              |              |
    |              |  access_token |              |              |
    |              |  (sub, act)   |              |              |
    |              |-------------->|              |              |
    |              |               |              |              |
    |              |               | 3. Request   |              |
    |              |               |  Txn-Token   |              |
    |              |               |  (access_token + subject_token)
    |              |               |------------->|              |
    |              |               |              |              |
    |              |               |              | 4. Lookup    |
    |              |               |              |  client_id   |
    |              |               |              |............->|
    |              |               |              |              |
    |              |               |              |  agent=true  |
    |              |               |              |<.............|
    |              |               |              |              |
    |              |               |              | 5. Issue     |
    |              |               |              |  Txn-Token:  |
    |              |               |              |  sub=user    |
    |              |               |              |  act={agent} |
    |              |               |              |  agentic_ctx |
    |              |               |              |              |
    |              |               | 6. Txn-Token |              |
    |              |               |<-------------|              |
    |              |               |              |              |
```

Legend:
- `-->` Solid: Primary request/response flow
- `...>` Dotted: Registry lookup (optional, per §3.7)


### 3.2.2. Autonomous Flow

When the agent application operates autonomously, the following
steps occur:

1. The agent application initiates a task based on an event or
   scheduled assignment.

2. The agent application calls an external endpoint.  The OAuth
   challenge flow starts.

3. The agent application authenticates using an OAuth 2.0 Client
   Credentials Grant [RFC6749](https://tools.ietf.org/html/rfc6749).

4. The agent application uses the access token to call the external
   endpoint.

5. The external endpoint submits the received access token along with
   its subject token to the Txn-Token Service.  Subject token
   requirements are specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).

6. The Txn-Token Service validates the access token.

7. The Txn-Token Service populates the Txn-Token's `sub` claim
   following the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).  For
   autonomous agents, the `sub` claim is determined based on the
   `subject_token` provided in the request, which typically
   represents the agent's own identity.

8. If the access token contains an `act` claim, the Txn-Token Service
   copies it to the Txn-Token.  For Client Credentials Grant flows,
   the access token typically does not contain an `act` claim, and
   therefore the Txn-Token MUST NOT contain an `act` claim.

9. If the TTS has access to an Agent Attribute Store, it performs a lookup
   and populates `agentic_ctx` if the caller is identified as an
   agent.  If no Agent Attribute Store is available, the TTS MAY still
   populate `agentic_ctx` using information derivable from the token
   exchange.


```
+-------+     +----------+     +-----+     +----------+
|  1P   |     | External |     | TTS |     |  Agent   |
| Agent |     | Endpoint |     |     |     |Attr Store|
+---+---+     +----+-----+     +--+--+     +----+-----+
    |               |              |              |
    | 1. Event/     |              |              |
    |  schedule     |              |              |
    |  triggers     |              |              |
    |  task         |              |              |
    |               |              |              |
    | 2. Client Credentials Grant  |              |
    |  (obtains access_token)      |              |
    |----------------------------->|              |
    |<-----------------------------|              |
    |               |              |              |
    | 3. Call with  |              |              |
    |  access_token |              |              |
    |  (no act)     |              |              |
    |-------------->|              |              |
    |               |              |              |
    |               | 4. Request   |              |
    |               |  Txn-Token   |              |
    |               |  (access_token + subject_token)
    |               |------------->|              |
    |               |              |              |
    |               |              | 5. Lookup    |
    |               |              |  client_id   |
    |               |              |............->|
    |               |              |              |
    |               |              |  agent=true  |
    |               |              |<.............|
    |               |              |              |
    |               |              | 6. Issue     |
    |               |              |  Txn-Token:  |
    |               |              |  sub=agent   |
    |               |              |  (no act)    |
    |               |              |  agentic_ctx |
    |               |              |              |
    |               | 7. Txn-Token |              |
    |               |<-------------|              |
    |               |              |              |
```

Note: In autonomous flows, `act` is absent (no human delegation).
The `originator.id` in `agentic_ctx` is set from `client_id`.

## 3.4. Replacement tokens

Txn-Token Service provides capability to get a replacement Txn-Token as defined in the [OAUTH-TXN-TOKENS.replacement flow](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html#name-creating-replacement-txn-to). This specification
extends the replacement flow with agent-specific behavior.

When a workload identified as an agent receives a Transaction Token
and intends to invoke a downstream workload, it SHOULD request a
replacement Txn-Token from the TTS before making the downstream call.
Failure to do so results in stale `agentic_ctx` metadata that does
not reflect the actual execution path; deployments enforcing chain
integrity policies based on `hop_count` or `min_assurance_level`
MUST treat such tokens as non-compliant (see Section 3.4 paragraph
below).

If a deployment enforces chain integrity policies based on
`hop_count` or `min_assurance_level`, then agents MUST request
replacement before downstream invocation.  Failure to replace results
in stale chain metadata that does not reflect the actual execution
path.

### 3.4.1. Replacement When Caller is Identified as an Agent

When the workload requesting a replacement Txn-Token is identified as
an agent (per Section 3.7), the following rules apply in addition to
the base specification replacement rules:

*  The `sub`, `txn`, and `aud` claims MUST NOT be modified (per base
   specification).

*  The `act` claim, if present, MUST NOT be modified.  The `act`
   claim is immutable throughout the entire transaction lifetime.

*  The `scope` claim can only be attenuated (per base specification).

*  The `req_wl` claim MUST be updated by appending the current
   requesting workload's identifier, using the comma (,) character as
   separator per the base specification.

*  The `agentic_ctx` claim MUST be updated as follows:

   -  The `proximate_agent.id` field MUST be set to the identifier of
      the new requesting agent.

   -  If `proximate_agent.session_id` is present, it MUST be updated
      alongside `proximate_agent.id` to reflect the session identifier
      of the new requesting agent.

   -  The `originator.session_id` field, if present, MUST NOT be
      modified.

   -  The `chain_metadata.hop_count` MUST be incremented by one.

   -  If `chain_metadata.min_assurance_level` is present, it MUST be
      recalculated.  The value MUST be the minimum of the existing
      `min_assurance_level` and the new agent's assurance level as
      recorded in the Agent Attribute Store (monotonic attenuation).

### 3.4.2. Replacement When Caller is Not Identified as an Agent

When the workload requesting a replacement Txn-Token is NOT identified
as an agent, the TTS MUST follow the base specification
[OAUTH-TXN-TOKENS] replacement rules without modification.  The
`agentic_ctx` claim, if present in the incoming Txn-Token, MUST be
carried forward without modification.  The `act` claim, if present,
MUST NOT be modified. This means, even if agents are anywhere in the call chain, 
the internal workloads receiving a Txn-Token now gets 
claims in Txn-Token that inform if one or more agents were involved in the call chain
and they can do additional authorization checks.

### 3.4.3. Token State Transitions 
```
       INITIAL ISSUANCE                 REPLACEMENT (hop 2)
       ================                 ===================

+------------------+                    +------------------+
| sub: user        |  --- immutable --> | sub: user        |
| act: {3P}        |  --- immutable --> | act: {3P}        |
| txn: abc-123     |  --- immutable --> | txn: abc-123     |
| aud: trust-dom   |  --- immutable --> | aud: trust-dom   |
| scope: billing   |  --- attenuate -> | scope: billing   |
| req_wl: apigw    |  --- append ----> | req_wl: apigw,1P |
+------------------+                    +------------------+
| agentic_ctx:     |                    | agentic_ctx:     |
|  proximate: 3P   |  --- update ----> |  proximate: 1P   |
|  originator: 3P  |  --- immutable -> |  originator: 3P  |
|  hop_count: 1    |  --- increment -> |  hop_count: 2    |
|  min_assur: (opt)|  --- attenuate -> |  min_assur: (opt)|
+------------------+                    +------------------+
```

## 3.5. Txn-Token Format

No changes to the JWT header from the base specification: `typ` MUST
be `txntoken+jwt`, with a signing key identifier such as `kid`.

The Txn-Token body augments the base claim set with the `act` field
(when present in the input access token) and the `agentic_ctx` claim
for agent context.  Existing claims like `txn`, `sub`, `aud`, `iss`,
`iat`, `exp`, `scope`, `tctx`, and `req_wl` retain identical
semantics, population rules, and immutability guarantees as defined
in [OAUTH-TXN-TOKENS].

In this example, the agent is a 3rd-party agent not part of the trust
domain.  It hits the API Gateway in the trust domain, and the API
Gateway requests a Txn-Token from the Txn-Token Service using the
access token received from the 3P agent and its own subject token (to
authenticate with the Txn-Token Service).  The requesting workload is
the API Gateway.  The agent is identified by the `act` claim copied
from the access token issued to the 3P agent to act on behalf of the
user.

~~~ json
{
  "iat": 1697059200,
  "aud": "trust-domain.example",
  "exp": 1697059500,
  "txn": "c2dc3992-2d65-483a-93b5-2dd9f02c276e",
  "sub": "user:alice@example.com",
  "scope": "trade.stocks",
  "req_wl": "apigateway.trust-domain.example",
  "act": {
    "sub": "agent-identity-1"
  },
  "tctx": {
    "action": "BUY",
    "ticker": "MSFT",
    "quantity": "100"
  },
  "agentic_ctx": {
    "proximate_agent": {
      "id": "agent-identity-1"
    },
    "originator": {
      "id": "agent-identity-1"
    },
    "chain_metadata": {
      "hop_count": 1
    }
  }
}
~~~

## 3.6. Agentic Context

The Txn-Token MAY contain an `agentic_ctx` claim.  Txn-Tokens are
increasingly used in environments where transactions are executed by
or with the assistance of autonomous or semi-autonomous agents (for
example, Large Language Model (LLM)-based agents, workflow
orchestrators, and policy-driven automation components).  In such
deployments, relying exclusively on subject identity and generic
transaction parameters is insufficient to make robust authorization
decisions.  Additional information about the agent chain is required.

The `agentic_ctx` claim provides a container for agent-specific
context within the Transaction Token.  This specification defines a
minimal set of normative fields required for multi-agent flow
integrity.  Resource Servers requiring richer context beyond what is
carried in the token — such as operational posture, attestation
status, or provenance metadata — SHOULD query the Agent Attribute Store
directly using the agent identifiers carried in `proximate_agent.id`
or `originator.id`.  Embedding mutable deployment attributes in an
immutable token is avoided by design: the agent identity already
provides an unambiguous lookup key, and attribute store-sourced data
is always current rather than frozen at token issuance time.

The depth of context available within `agentic_ctx` differs between
external and internal agents.  For external agents entering the trust
domain, the TTS populates the normative fields based on information
available from the token exchange (e.g., `act.sub` or `client_id`)
and assigns context commensurate with the trust domain's policy for
unverified external actors.  For internal agents within the trust
domain, the TTS MAY additionally source the agent's assurance level
from the Agent Attribute Store for use in `min_assurance_level` calculation.
This asymmetry reflects the reality that the trust domain does not
own or control external agents and therefore cannot verify their
operational posture to the same degree.

### 3.6.1. Normative Fields

The following fields within `agentic_ctx` are defined by this
specification:

proximate_agent:
:  REQUIRED.  A JSON object identifying the agent whose request caused
   the current token to be issued or most recently replaced.  The
   object contains the following sub-fields:

   id:
   :  REQUIRED.  A string identifying the proximate agent.  At initial
      issuance, the TTS sets this field to the agent identifier derived
      from the access token (via `act.sub`, Agent Attribute Store lookup, or
      `client_id` — in that priority order).  At each replacement where
      the requesting workload is identified as an agent, the TTS updates
      this field to that agent's identifier.  When the requesting
      workload is not identified as an agent, this field MUST NOT be
      modified.  Resource Servers SHOULD use this field when evaluating
      chain integrity policies and when querying the Agent Attribute Store for
      richer context about the current agent.

   session_id:
   :  OPTIONAL.  A string identifying the session of the proximate
      agent as known to that agent's own runtime or infrastructure
      (e.g., a session identifier assigned by the agent's runtime
      framework, harness, or SDK).  This field is self-asserted by the
      agent's infrastructure layer and is therefore informational only.
      Resource Servers and the TTS MUST NOT use this field as an
      authorization criterion.  Its purpose is operational correlation:
      a workload that observes unexpected behavior may share the `txn`
      claim and this value with the proximate agent's owning team,
      enabling that team to locate the corresponding session in their
      own runtime logs for debugging or revocation.  This field MUST be
      updated alongside `proximate_agent.id` during replacement flows
      so that it always reflects the session of the current proximate
      agent.

originator:
:  REQUIRED.  A JSON object identifying the agent that initiated the
   transaction chain.  The object contains the following sub-fields:

   id:
   :  REQUIRED.  A string identifying the originating agent.  This
      value is set when the initial Txn-Token is issued and MUST NOT be
      modified during replacement flows.  For single-agent flows,
      `originator.id` and `proximate_agent.id` have the same value.
      The `originator.id` field exists to support authorization policy
      evaluation in scenarios where the `act` claim is absent (e.g.,
      autonomous agent flows using Client Credentials Grant where no
      delegation occurs).  In flows where `act` is present, the
      `originator.id` value will typically match `act.sub`.  This field
      ensures that the chain origin is always available within
      `agentic_ctx` regardless of whether the transaction involves
      human delegation.  Resource Servers SHOULD use this field for
      authorization policies that depend on the identity of the chain
      originator, particularly in deployments where the `act` claim may
      be absent.  Resource Servers MAY also use `originator.id` to
      query the Agent Attribute Store for context about the chain origin.

   session_id:
   :  OPTIONAL.  A string identifying the session of the originating
      agent as known to that agent's own runtime or infrastructure.
      This value is set at initial token issuance and MUST NOT be
      modified during replacement flows, mirroring the immutability of
      `originator.id`.  The trust model and acceptable sources are
      identical to `proximate_agent.session_id`: the value is
      self-asserted and informational only.  Resource Servers and the
      TTS MUST NOT use this field as an authorization criterion.  Its
      purpose is to allow a workload or the TTS to cross-reference the
      originating agent's runtime logs independently of the proximate
      agent when investigating a transaction chain.

chain_metadata:
:  REQUIRED.  A JSON object containing chain-level integrity
   metadata.  The following sub-fields are defined:

   hop_count:
   :  REQUIRED.  An integer representing the number of agent
      transitions in the current chain.  Set to 1 at initial
      issuance.  Incremented by 1 at each replacement where the
      caller is identified as an agent.  Resource Servers SHOULD
      evaluate this value against deployment-defined maximum hop
      limits when making authorization decisions.

   min_assurance_level:
   :  OPTIONAL.  A string representing the minimum assurance level
      across all agents in the chain.  When present, the TTS MUST
      set this to the minimum of the existing value and the new
      agent's assurance level during each replacement flow (monotonic
      attenuation).  This field is only meaningful in deployments
      that define an ordered set of assurance levels and maintain an
      Agent Attribute Store from which agent assurance levels can be
      determined.  This specification does not define a fixed
      enumeration of assurance level values.  Deployments MUST
      define their own ordered set and comparison function.  Non-
      normative examples include: "unverified", "low", "medium",
      "high".

attr_store_ref:
:  OPTIONAL.  A URI identifying the Agent Attribute Store endpoint for this
   trust domain.  The TTS MAY include this field to assist Resource
   Servers in locating the Agent Attribute Store without out-of-band
   configuration.  Resource Servers MAY use this URI to query for
   richer agent context (posture, provenance, attestation status)
   using `proximate_agent.id` or `originator.id` as the lookup key.
   This field is informational and does not affect token validity.
   Resource Servers MUST authenticate the Agent Attribute Store response
   independently of this field's presence.

### 3.6.2. Agent Attribute Store Queries for Richer Context

Resource Servers that require richer agent context beyond the
normative fields — such as operational posture, hardware attestation
status, software provenance, or behavioral configuration — SHOULD
query the Agent Attribute Store directly using the agent identifier from
`proximate_agent.id` or `originator.id`.

This approach is preferred over embedding deployment-specific
attributes in the token for two reasons.  First, posture and
provenance data are mutable: a token carries a snapshot of state at
issuance time, whereas an attribute store query returns current state.
Authorization rules based on attribute store-sourced data remain stable
across deployment changes, while rules based on token-embedded fields
must be updated whenever infrastructure changes.  Second, embedding
verbose posture fields increases token size without benefit, since the
agent identity already provides an unambiguous lookup key.

The Agent Attribute Store interface, query protocol, and response format are
out of scope for this specification.  Deployments MUST ensure that
Agent Attribute Store endpoints are authenticated and that store contents
are protected with integrity controls appropriate to their security
requirements (see Security Considerations).

### 3.6.3. Example agentic_ctx

The following is a non-normative example showing `agentic_ctx` in a
multi-agent flow with all normative fields populated:

~~~ json
{
  "agentic_ctx": {
    "proximate_agent": {
      "id": "spiffe://prod.acme.com/billing-agent",
      "session_id": "sess-a3f7c9d2b1e4"
    },
    "originator": {
      "id": "3p-assistant-ext-99",
      "session_id": "sess-7b2e1a9f4c03"
    },
    "chain_metadata": {
      "hop_count": 2,
      "min_assurance_level": "low"
    },
    "attr_store_ref": "https://registry.trust-domain.example/agents"
  }
}
~~~

Note: `proximate_agent.session_id` and `originator.session_id` are
informational only; their values are supplied by the respective
agent's infrastructure layer and MUST NOT be used as authorization
criteria.  Resource Servers requiring posture or provenance data
SHOULD query the Agent Attribute Store at `attr_store_ref` using
`proximate_agent.id` or `originator.id` as the lookup key.

### 3.6.4. Implementation Guidance

The following guidance applies to implementations using `agentic_ctx`:

*  Resource Servers receiving a Txn-Token with `agentic_ctx` SHOULD
   evaluate the normative fields (`proximate_agent.id`, `originator.id`,
   `chain_metadata`) for chain integrity decisions.

*  Resource Servers requiring richer context SHOULD query the Agent
   Registry using `proximate_agent.id` or `originator.id`.  If
   `attr_store_ref` is present, it MAY be used to locate the registry
   endpoint; the response MUST be authenticated independently.

*  Resource Servers MUST NOT use `proximate_agent.session_id` or
   `originator.session_id` as authorization criteria.

*  If a Resource Server does not recognize a field within `agentic_ctx`,
   it MUST ignore that field and MUST NOT reject the token solely on
   that basis.

## 3.7. Agent Identification

The Transaction Token Service needs to determine whether a requesting
workload is an agent in order to apply the appropriate token issuance
and replacement rules defined in this specification.

### 3.7.1. Identification Mechanisms

The TTS SHOULD have access to an Agent Attribute Store or equivalent
mechanism to identify agent callers.  When an Agent Attribute Store is
available, the TTS uses it to determine whether agent-specific token
issuance rules apply and to source metadata for `agentic_ctx`
population.

When the TTS receives a Txn-Token request or replacement request and
has access to an Agent Attribute Store, it SHOULD perform the following
determination:

1. Extract the `client_id` from the presented access token.

2. Query the Agent Attribute Store to determine if the `client_id`
   corresponds to a registered agent.

3. If the `client_id` resolves to a registered agent:

   -  The TTS MUST apply the agent-specific token issuance rules
      defined in this specification.
   -  The TTS MUST populate the `agentic_ctx` claim with at minimum
      the normative fields defined in Section 3.6.1.
   -  If the access token contains an `act` claim, the TTS MUST copy
      it to the Txn-Token without modification.

4. If the `client_id` does NOT resolve to a registered agent:

   -  The TTS MUST follow the base specification [OAUTH-TXN-TOKENS]
      rules without modification.
   -  The TTS MUST NOT populate an `agentic_ctx` claim.

In deployments without an Agent Attribute Store, the TTS MAY use alternative
mechanisms to identify agents, such as:

*  The presence of an `act` claim in the access token (indicating
   delegation to an agent).
*  Convention-based `client_id` patterns.
*  Out-of-band configuration or policy rules.

The specific mechanism for agent identification is deployment-defined.
This specification defines the behavior once the determination is
made, not the mechanism itself.

### 3.7.2. Agent Attribute Store Contents

When an Agent Attribute Store is used, it SHOULD contain, at minimum, the
following information for each registered agent:

*  `client_id`: The OAuth 2.0 client identifier for the agent.
*  `agent_name`: A human-readable name for the agent.

If the deployment uses `min_assurance_level`, the Agent Attribute Store MUST
also contain:

*  `assurance_level`: The assurance level assigned to this agent,
   used for `min_assurance_level` calculation during replacement flows.

Deployments that require Resource Servers to perform posture-based
authorization SHOULD additionally store the following attributes in
the Agent Attribute Store, making them queryable by agent identifier:

*  Operational posture: Hardware attestation status (e.g., TEE type),
   runtime integrity measurements, security tier classification.
*  Provenance: Software version identifiers, manifest references,
   source code references.
*  Attestation metadata: Certificates, endorsements, or other
   cryptographic evidence supporting the above attributes.

These richer attributes are populated and maintained by the trust
domain's deployment pipelines and MUST NOT be sourced from agent
self-assertion.  The Agent Attribute Store SHOULD support authenticated
query interfaces so that Resource Servers can retrieve and verify
these attributes at authorization time using `proximate_agent.id`
or `originator.id` as the lookup key.

The implementation of the Agent Attribute Store is out of scope for this
specification.  Deployments MAY use existing service registries,
dedicated agent catalogs, or policy engines to fulfill this role.


## 3.8. Token Exchange Isolation

Agent systems typically consist of two distinct functional layers:

*  The *inference layer*: The component that runs the LLM or AI
   model, processes natural language, generates plans, and produces
   tool calls or outputs.

*  The *infrastructure layer*: The component that executes tool
   calls, manages network I/O, enforces security policies, and
   interacts with external services including the Transaction Token
   Service.

The token exchange mechanism defined by this specification —
including Txn-Token acquisition and replacement requests — MUST be
implemented in the infrastructure layer and MUST NOT be callable or
bypassable by the inference layer.  This requirement exists because
an LLM can be adversarially influenced (e.g., through prompt
injection or jailbreak attacks) to issue instructions that attempt
to subvert authorization controls.  Placing the token exchange
logic in the infrastructure layer ensures these controls remain
effective regardless of LLM output.

Specifically:

*  Token exchange logic MUST NOT be exposed as a tool, function, or
   capability that the LLM can invoke directly.

*  The LLM's output MUST NOT be used as direct input to Txn-Token
   exchange requests without validation and approval by the
   infrastructure layer.

*  Access tokens, Txn-Tokens, and other credential material MUST
   NOT appear in the LLM's context window, prompt, or generated
   output.

*  The infrastructure layer MUST submit Txn-Token requests for all
   calls to protected resources, regardless of whether the LLM's
   output attempts to indicate otherwise.

The following non-normative architectural patterns satisfy this
requirement:

*  *Agent Harness or Framework SDK*: A runtime wrapper around the
   LLM that intercepts tool calls and performs token exchange
   transparently before executing calls to downstream services.
   The LLM sees only tool results, never credential material.

*  *Agent Gateway or Proxy*: A network-layer component that
   intercepts outbound calls from the agent process and performs
   token exchange before forwarding.  The agent process never
   handles tokens directly.

*  *Sidecar Container*: A co-deployed process responsible for token
   exchange that communicates with the agent container over a local
   interface, keeping credential material out of the agent's
   address space.

The specific pattern is deployment-defined.  What is normatively
REQUIRED is that the token exchange function cannot be skipped or
manipulated by output from the inference layer.  Deployments SHOULD
document and enforce the trust boundary between the inference layer
and the infrastructure layer as part of their security model.


## 3.9. TTS Audit Logging

The Transaction Token Service is the only party that observes every
token exchange in a transaction chain.  At initial issuance and at
each replacement, the TTS receives the requesting workload's
credentials and, when present, the `proximate_agent.session_id`
value supplied by the agent's infrastructure layer.  This positions
the TTS as the authoritative audit ledger for the full agent chain,
complementing the token itself which carries only the proximate and
originating agents.

The TTS SHOULD maintain a persistent audit log.  Without a
persistent log, offline reconstruction of the full agent call chain
for a given transaction is not possible, limiting incident response
to the information visible in individual tokens.  For each token
issuance or replacement request, the log entry SHOULD capture at
minimum:

*  The transaction identifier (`txn` claim value).
*  A timestamp of the exchange request.
*  The resolved agent identifier for the requesting workload (from
   Agent Attribute Store lookup or `client_id`).
*  The `proximate_agent.session_id` value as presented in the
   request, if provided.
*  The `originator.session_id` value, if present in the token
   (relevant at initial issuance when the originating agent provides it).
*  Whether the request was an initial issuance or a replacement.
*  The resulting `hop_count` in the issued token.

This log enables offline reconstruction of the complete agent call
chain for a given `txn`, including all intermediate hops that are
not individually visible in any single token.  A workload that
shares the `txn` value with the TTS operator can obtain the full
chain history, while a workload that also shares a
`proximate_agent.session_id` or `originator.session_id` value allows
the relevant agent's owning team to locate the corresponding session
in their own runtime logs.

The TTS audit log is a security-sensitive resource.  Access to it
MUST be restricted to authorized parties.  The TTS SHOULD protect
the log with tamper-evident storage to preserve its integrity as a
forensic record.

# 4. Multi-Agent Flows

In complex agentic workflows, a transaction often originates from a
3rd-party (3P) agent and propagates through one or more 1st-party
(1P) agents within the local trust domain.  To maintain Zero Trust
integrity, this specification uses the `agentic_ctx` claim to track
the chain of agents involved in the transaction.  This ensures that
downstream Resource Servers can evaluate the chain integrity and
overall assurance level, rather than relying solely on the identity
of the immediate caller.

## 4.1. External vs. Internal Agents

The `agentic_ctx` differentiates between two categories of agents
through the normative `proximate_agent` and `originator` fields:

External Agents (3P):
:  Agents that are not owned or controlled by the trust domain.  They
   enter the trust domain through an external endpoint.  Because their
   hardware and software are outside local control, the TTS has
   limited ability to verify their operational posture.  Their
   identity is typically captured via the `act` claim (from the access
   token) and reflected in the `originator.id` field.  The Agent
   Registry may have limited or no entries for external agents; their
   assurance level is typically "unverified" or "low".

Internal Agents (1P):
:  Agents that operate within the trust domain and are owned or
   controlled by the trust domain operator.  The TTS has full ability
   to verify their assurance level through the Agent Attribute Store.  Their
   identity is reflected in the `proximate_agent.id` field during
   replacement flows.  Resource Servers MAY query the Agent Attribute Store
   using `proximate_agent.id` to obtain posture and provenance
   attributes for these agents.

This distinction is not a protocol-level differentiation — both
external and internal agents are identified through the same
mechanisms (Section 3.7).  The distinction affects the *depth* of
context the Agent Attribute Store can provide for each agent type.

```
+================================================================+
|                      TRUST DOMAIN                               |
|                                                                |
|  +------------------+         +---------------------------+    |
|  | TTS              |         | Agent Attribute Store            |    |
|  |                  |<------->| - client_id               |    |
|  |                  |         | - agent_name              |    |
|  |                  |         | - assurance_level (opt)   |    |
|  +--------+---------+         | - posture attrs (query)   |    |
|           |                   +---------------------------+    |
|           |                                                    |
|           | Issues Txn-Token                                   |
|           |                                                    |
|  +--------v-----------------------------------------+          |
|  | agentic_ctx                                      |          |
|  |                                                  |          |
|  | For EXTERNAL (3P) agent:   For INTERNAL (1P) agent:        |
|  | +----------------------+   +------------------------+      |
|  | | proximate_agent:     |   | proximate_agent:       |      |
|  | |   id: 3P-id          |   |   id: 1P-id            |      |
|  | | originator:          |   | originator:            |      |
|  | |   id: 3P-id          |   |   id: 3P-id            |      |
|  | | chain_metadata:      |   | chain_metadata:        |      |
|  | |   hop_count: 1       |   |   hop_count: 2         |      |
|  | |                      |   |   min_assurance: "low" |      |
|  | | (RS queries store |   |                        |      |
|  | |  for posture — may   |   | (RS queries store   |      |
|  | |  be unverified)      |   |  for verified posture) |      |
|  | +----------------------+   +------------------------+      |
|  +----------------------------------------------------------+  |
|                                                                |
+================================================================+
```

Note: Neither token variant embeds posture data.  Resource Servers
query the Agent Attribute Store using `proximate_agent.id` or `originator.id`
to retrieve posture attributes appropriate to the agent's trust level.

## 4.2. Monotonic Attenuation of Trust

A chain's security posture is only as strong as its weakest link.
When `min_assurance_level` is used, the TTS MUST calculate it during
every token replacement flow.  If a "high" assurance internal agent
is triggered by a "low" assurance 3P originator, the transaction's
overall assurance level remains "low".  This prevents Identity
Laundering, where unverified external agents bypass security
guardrails by proxying requests through internal services.

The Transaction Token Service (TTS) determines the
`min_assurance_level` by comparing the existing value with the
assurance level of the new requesting agent (as recorded in the
Agent Attribute Store) and selecting the minimum.

### 4.2.1. Delegation via Replacement Flow

When an internal agent (the "Delegatee") requires a Transaction Token
to continue a chain initiated by another actor (the "Delegator"), it
SHOULD follow the replacement flow procedures defined in
[OAUTH-TXN-TOKENS] with the following modifications:

*  Subject Immutability: The `txn`, `sub`, and `aud` claims MUST be
   copied from the subject_token to the new Transaction Token without
   modification.

*  Actor Immutability: The `act` claim, if present, MUST be copied
   without modification.  The `act` claim represents the original
   agent delegation and MUST NOT change throughout the transaction
   lifetime.

*  Current Actor Update: The `proximate_agent.id` field MUST be set
   to the Delegatee's identifier.

*  Chain Metadata Update: The TTS MUST increment `hop_count` and,
   if `min_assurance_level` is present, recalculate it.

*  Workload Tracking: The TTS MUST append the Delegatee's workload
   identifier to the `req_wl` claim.

*  Originator Preservation: The `originator` object MUST NOT be
   modified.

If a deployment enforces chain integrity policies, internal agents
MUST request replacement before downstream invocation to ensure
accurate chain metadata.

### 4.2.2. Multi-Agent Example

The following example illustrates a multi-agent flow where a
3rd-party agent ("3p-assistant-ext-99") initiates a transaction on
behalf of a user, and the request is subsequently handled by an
internal 1st-party agent ("1p-billing-svc-v2") within the trust
domain.

Flow description:

1. The 3P agent ("3p-assistant-ext-99") obtains an access token via
   OAuth 2.0 Authorization Code flow on behalf of the user.  The
   access token contains an `act` claim identifying the agent:
   `{"sub": "3p-assistant-ext-99"}`.

2. The 3P agent calls the trust domain's API Gateway with this access
   token.

3. The API Gateway submits the access token and its own subject token
   to the TTS, requesting a Txn-Token.

4. The TTS validates the access token, identifies the caller as an
   agent (via Agent Attribute Store lookup or alternative mechanism), and
   issues the initial Txn-Token with:
   -  `sub` populated from the `subject_token` (the user)
   -  `act` copied from the access token: `{"sub": "3p-assistant-ext-99"}`
   -  `agentic_ctx.proximate_agent.id` set to "3p-assistant-ext-99"
   -  `agentic_ctx.originator.id` set to "3p-assistant-ext-99"
   -  `agentic_ctx.chain_metadata.hop_count` set to 1
   -  `req_wl` set to the API Gateway's identifier

5. The API Gateway forwards the Txn-Token to an internal agent
   ("1p-billing-svc-v2").

6. The internal agent needs to call a downstream Resource Server.  It
   requests a replacement Txn-Token from the TTS, presenting the
   existing Txn-Token as the `subject_token`.

7. The TTS identifies "1p-billing-svc-v2" as an agent.  It issues a
   replacement Txn-Token with:
   -  `sub`, `txn`, `aud` unchanged
   -  `act` unchanged: `{"sub": "3p-assistant-ext-99"}`
   -  `agentic_ctx.proximate_agent.id` updated to "1p-billing-svc-v2"
   -  `agentic_ctx.originator.id` unchanged: "3p-assistant-ext-99"
   -  `agentic_ctx.chain_metadata.hop_count` incremented to 2
   -  `req_wl` appended with the internal agent's identifier


```
+-------+    +---------+    +-------+    +-----+    +----------+    +----------+
|  3P   |    |   API   |    |  1P   |    | TTS |    |  Agent   |    | Resource |
| Agent |    | Gateway |    | Agent |    |     |    |Attr Store|    |  Server  |
+---+---+    +----+----+    +---+---+    +--+--+    +----+-----+    +----+-----+
    |              |             |           |            |               |
    | 1. access_token            |           |            |               |
    |  (sub=user,  |             |           |            |               |
    |   act={3P})  |             |           |            |               |
    |------------->|             |           |            |               |
    |              |             |           |            |               |
    |              | 2. Request Txn-Token    |            |               |
    |              |  (access_token +        |            |               |
    |              |   subject_token)        |            |               |
    |              |----------------------->|            |               |
    |              |             |           |            |               |
    |              |             |           | 3. Lookup  |               |
    |              |             |           |  client_id |               |
    |              |             |           |...........>|               |
    |              |             |           |  agent=3P  |               |
    |              |             |           |<...........|               |
    |              |             |           |            |               |
    |              |             |           | 4. Issue Txn-Token (hop 1) |
    |              |             |           |  sub=user                  |
    |              |             |           |  act={3P}                  |
    |              |             |           |  agentic_ctx:              |
    |              |             |           |    proximate_agent.id=3P   |
    |              |             |           |    originator.id=3P        |
    |              |             |           |    hop_count=1             |
    |              |             |           |            |               |
    |              | 5. Txn-Token|           |            |               |
    |              |<-----------|-----------||            |               |
    |              |             |           |            |               |
    |              | 6. Forward  |           |            |               |
    |              |  Txn-Token  |           |            |               |
    |              |------------>|           |            |               |
    |              |             |           |            |               |
    |              |             | 7. Request Replacement  |               |
    |              |             |  (Txn-Token as          |               |
    |              |             |   subject_token)        |               |
    |              |             |---------->|            |               |
    |              |             |           |            |               |
    |              |             |           | 8. Lookup  |               |
    |              |             |           |  client_id |               |
    |              |             |           |...........>|               |
    |              |             |           |  agent=1P  |               |
    |              |             |           |<...........|               |
    |              |             |           |            |               |
    |              |             |           | 9. Issue Replacement       |
    |              |             |           |  (hop 2)                   |
    |              |             |           |  sub=user (unchanged)      |
    |              |             |           |  act={3P} (unchanged)      |
    |              |             |           |  agentic_ctx:              |
    |              |             |           |    proximate_agent.id=1P   |
    |              |             |           |    originator.id=3P        |
    |              |             |           |    hop_count=2             |
    |              |             |           |            |               |
    |              |             | 10. Replacement        |               |
    |              |             |  Txn-Token |           |               |
    |              |             |<----------|           |               |
    |              |             |           |            |               |
    |              |             | 11. Call with Txn-Token |               |
    |              |             |---------------------------------------->|
    |              |             |           |            |               |
    |              |             |           |            |  12. Validate |
    |              |             |           |            |  Txn-Token:   |
    |              |             |           |            |  - sub=user   |
    |              |             |           |            |  - act={3P}   |
    |              |             |           |            |  - hop=2      |
    |              |             |           |            |  - orig.id=3P |
    |              |             |           |            |  - prox.id=1P |
    |              |             |           |            |               |
    |              |             |           |            | 13. Query     |
    |              |             |           |            |  attr. store  |
    |              |             |           |            |  for posture  |
    |              |             |           |            |  prox.id=1P   |
    |              |             |           |            |............-->|
    |              |             |           |            |               |
```

Legend:
- `-->` Solid: Primary request/response flow
- `...>` Dotted: Agent Attribute Store lookup (optional, per §3.7)
- 
The following is the Txn-Token at the final hop (after step 7),
showing all normative fields:

```json
{
  "iat": 1712850000,
  "aud": "trust-domain.example",
  "exp": 1712850300,
  "txn": "abc-123-xyz",
  "sub": "user_8821@example.com",
  "iss": "https://txn-svc.trust-domain.example",
  "scope": "billing.process",
  "req_wl": "apigateway.trust-domain.example,1p-billing-svc-v2.trust-domain.example",
  "act": {
    "sub": "3p-assistant-ext-99"
  },
  "agentic_ctx": {
    "proximate_agent": {
      "id": "1p-billing-svc-v2"
    },
    "originator": {
      "id": "3p-assistant-ext-99"
    },
    "chain_metadata": {
      "hop_count": 2
    }
  }
}
```

Note: The `act` claim value `{"sub": "3p-assistant-ext-99"}` is
identical to what was set at hop 1.  It represents the original
agent delegation and is never modified during replacement flows.
The `originator.id` field within `agentic_ctx` mirrors this value
for convenience in authorization policy evaluation.

The following is a non-normative example of the same token in a
deployment that uses `min_assurance_level` and includes a
`attr_store_ref` for Resource Servers that need posture context:

```json
{
  "iat": 1712850000,
  "aud": "trust-domain.example",
  "exp": 1712850300,
  "txn": "abc-123-xyz",
  "sub": "user_8821@example.com",
  "iss": "https://txn-svc.trust-domain.example",
  "scope": "billing.process",
  "req_wl": "apigateway.trust-domain.example,1p-billing-svc-v2.trust-domain.example",
  "act": {
    "sub": "3p-assistant-ext-99"
  },
  "agentic_ctx": {
    "proximate_agent": {
      "id": "1p-billing-svc-v2"
    },
    "originator": {
      "id": "3p-assistant-ext-99"
    },
    "chain_metadata": {
      "hop_count": 2,
      "min_assurance_level": "low"
    },
    "attr_store_ref": "https://registry.trust-domain.example/agents"
  }
}
```

The Resource Server evaluates `min_assurance_level` from the token
directly.  For posture details about "1p-billing-svc-v2", it queries
the Agent Attribute Store at `attr_store_ref` using `proximate_agent.id` as
the lookup key.

### 4.2.3. Loop Prevention

To prevent infinite recursion in autonomous agentic loops, the TTS
MUST track the `hop_count` within `chain_metadata` and enforce a
maximum depth.  If the `hop_count` exceeds a deployment-defined
threshold, the replacement request MUST be rejected.

Implementations MAY include additional chain tracking fields within
`agentic_ctx` (such as a path of agent identifiers) but such fields
are not defined by this specification.


# 5. Security Considerations

All the security considerations mentioned in [OAUTH-TXN-TOKENS]
apply.

## Token Replay Protection

Implementations MUST enforce strict token lifetime validation.  The
short-lived nature of Transaction Tokens helps mitigate replay
attacks, but implementations SHOULD also consider:

*  Implementing token tracking mechanisms within trust domains
*  Validating token usage context

## Actor Identity Security

*  Implementations MUST validate `act` claims in tokens.
*  The Txn-Token Service MUST verify the authenticity of actor context
   before token issuance.
*  During replacement flow, the Txn-Token Service MUST NOT modify the
   `act` field in the incoming Txn-Token.  The immutability of `act`
   ensures that the original agent delegation is always visible and
   cannot be overwritten, preventing identity laundering attacks.

## Subject Context Protection

*  Systems MUST prevent unauthorized modifications to the `sub` claim
   during token propagation.  Txn-Tokens are cryptographically signed
   to ensure integrity.
*  During replacement flow, the Txn-Token Service MUST NOT modify the
   `sub` claim in the incoming Txn-Token.
*  The Txn-Token Service MUST follow the subject population rules
   defined in [OAUTH-TXN-TOKENS] to ensure proper subject
   representation.

## Transaction Chain Integrity

*  Services MUST verify the cryptographic signature of every
   Txn-Token before processing its claims.
*  Services MUST validate tokens at trust domain boundaries.
*  Service-to-service communication carrying Txn-Tokens SHOULD use
   mutually authenticated, encrypted channels (e.g., mTLS) to
   prevent tampering in transit.

## AI Agent Specific Controls

*  Implementations MUST enforce the `scope` attenuation rules
   defined in Section 3.4.1, ensuring agent operations cannot
   exceed the scopes granted at initial token issuance.
*  Systems SHOULD implement behavioral monitoring for AI agent
   activities by logging `act`, `sub`, and `agentic_ctx` claims in
   audit logs.
*  Systems SHOULD maintain audit trails of AI agent activities,
   including the `txn` claim and `hop_count`, to support incident
   response.

## Token Transformation Security

*  The Txn-Token Service MUST validate all claims during access token
   to Txn-Token conversion.
*  Implementations MUST verify signatures and formats of all tokens.
*  Systems MUST use authenticated, encrypted channels for token
   exchange requests to prevent interception or manipulation.
*  The Txn-Token Service MUST ensure that the `act` field accurately
   represents the actor identity from the access token.

## Replacement Token Considerations

*  Systems MUST verify the authenticity and validity of original
   tokens before replacement.
*  Systems MUST implement controls to prevent unauthorized replacement
   requests.
*  The immutability of `act`, `sub`, and `originator.id` during
   replacement ensures consistent identity context throughout the
   transaction lifecycle.

## Infrastructure Security

*  All component communications MUST use mutually authenticated,
   encrypted channels (e.g., TLS with certificate validation).
*  The TTS MUST authenticate the requesting workload before issuing
   or replacing a Txn-Token, using the access token presented in
   the exchange request.
*  Implementations SHOULD rotate cryptographic signing keys for
   Txn-Tokens on a regular schedule to limit the exposure window
   of a compromised key.
*  Deployments SHOULD document and enforce trust domain boundaries
   to ensure Txn-Tokens are only accepted within their intended
   scope.

## Prevention of Identity Laundering

*  When `min_assurance_level` is used, implementations MUST enforce
   Monotonic Attenuation.
*  The TTS MUST NOT allow a replacement token to have a higher
   `min_assurance_level` than the incoming subject token, even if the
   current actor has a higher assurance level.
*  This prevents a low-trust 3rd-party originator from "laundering"
   its identity through a high-trust internal agent to bypass security
   guardrails at the Resource Server.
*  The immutability of the `act` claim and `originator.id` field provides
   an additional safeguard: the original delegating agent identity is
   always preserved and visible to downstream Resource Servers
   regardless of how many replacement hops occur.

## Integrity of the Agent Attribute Store

*  When an Agent Attribute Store is used, the security of the agent
   identification mechanism (Section 3.7) relies on the integrity of
   the Agent Attribute Store.
*  If an attacker can add entries to the Agent Attribute Store, they can
   cause the TTS to treat arbitrary workloads as agents, triggering
   agent-specific token issuance rules inappropriately.
*  If an attacker can modify entries, they can alter the assurance
   level assigned to an agent, undermining the monotonic attenuation
   guarantee.  They can also corrupt the posture or provenance
   attributes returned to Resource Servers on attribute store queries.
*  Access to the Agent Attribute Store MUST be restricted to authorized
   deployment pipelines and protected with strong integrity controls.
*  The Agent Attribute Store SHOULD support audit logging of all
   modifications.
*  Implementations SHOULD use cryptographic signatures or content-
   addressable storage to detect unauthorized modifications.
*  Resource Servers querying the Agent Attribute Store MUST authenticate
   the attribute store response.  The presence of `attr_store_ref` in the
   token does not substitute for independent verification of the
   attribute store endpoint's identity.

## Agent Identification Mechanism Security

*  When the TTS relies on `client_id` lookup in the Agent Attribute Store to
   determine whether a caller is an agent, a compromised `client_id`
   could lead to incorrect agent classification.
*  Mitigations include:
   -  Binding `client_id` to transport-layer identity (e.g., mTLS
      certificate or SPIFFE SVID).
   -  Requiring additional verification before accepting agent
      classification.
   -  Implementing rate limiting and anomaly detection on attribute
      store lookups.

## Loop Detection and Recursion Limits

*  The `hop_count` within `chain_metadata` is REQUIRED to prevent
   resource exhaustion in autonomous agentic workflows.
*  The TTS SHOULD enforce a maximum `hop_count` to prevent resource
   exhaustion attacks.  If the count exceeds a defined threshold, the
   replacement request MUST be rejected.

## Data Leakage in Context Propagation

*  When a Txn-Token egresses the trust domain (e.g., a 1st-party
   agent calling an external 3rd-party service), the TTS MUST strip
   the `attr_store_ref` field and any other internal identifiers from
   the token to prevent infrastructure leakage.
*  Only the normative fields and the minimum necessary context should
   be egressed from the trust domain.

## Stale Chain Metadata

*  If agents propagate incoming Txn-Tokens without requesting
   replacement, the `agentic_ctx` will contain stale metadata that
   does not reflect the actual execution path.
*  Deployments that rely on `hop_count` or `min_assurance_level` for
   authorization decisions MUST enforce mandatory replacement for
   agents (see Section 3.4).
*  Resource Servers SHOULD be aware that in deployments without
   mandatory replacement, `hop_count` represents a lower bound on
   the actual number of agent transitions.

## Agent Session Identifiers

The `proximate_agent.session_id` and `originator.session_id` fields
are self-asserted by the respective agent's infrastructure layer and
carry no cryptographic verification.  Both fields MUST NOT influence
authorization decisions.  Their security scope is intentionally
narrow: they exist to enable cross-system operational correlation
between workload audit logs and agent runtime logs during incident
response.

Because these fields cannot be verified by the TTS or Resource
Servers, a malicious agent could report a fabricated or another
session's identifier.  The consequence is limited to misleading
audit trails rather than enabling unauthorized access, since the
fields MUST NOT be used for access control.

Workload operators relying on these fields for incident response
SHOULD treat them as advisory hints, corroborated by the immutable
`txn` claim and other verified claims in the token.  The `txn`
claim and `hop_count` together uniquely identify each token in the
chain and provide the authoritative correlation handle for
security-sensitive analysis.

## Token Exchange Isolation

*  Placing token exchange logic within reach of the LLM's inference
   context (e.g., as a callable tool, a prompt variable, or an
   accessible environment secret) creates an attack surface where
   adversarial inputs — such as prompt injection attacks — can
   manipulate, bypass, or exfiltrate credential material.

*  Deployments MUST enforce the isolation requirement defined in
   Section 3.8.  Failure to do so undermines the authorization
   guarantees provided by the Txn-Token mechanism.

*  A Txn-Token does not carry in-band evidence that token exchange
   isolation was enforced; this remains a deployment-level guarantee.
   Deployments SHOULD use out-of-band attestation mechanisms (e.g.,
   hardware attestation of the agent infrastructure, signed deployment
   manifests) to provide Resource Servers with assurance that the
   isolation policy defined in Section 3.8 was in effect.

# References

## Normative References
[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)
    Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997, <https://www.rfc-editor.org/rfc/rfc2119>.

[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)
     Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174, May 2017, https://www.rfc-editor.org/rfc/rfc8174
    
[RFC6749](https://tools.ietf.org/html/rfc6749)
    Hardt, D., Ed., "The OAuth 2.0 Authorization Framework", RFC 6749, DOI 10.17487/RFC6749, October 2012, <https://www.rfc-editor.org/rfc/rfc6749>.

[RFC7519](https://tools.ietf.org/html/rfc7519)
    Jones, M., Bradley, J., and N. Sakimura, "JSON Web Token (JWT)", RFC 7519, DOI 10.17487/RFC7519, May 2015, <https://www.rfc-editor.org/rfc/rfc7519>.

[RFC7515](https://tools.ietf.org/html/rfc7515)
    Jones, M., Bradley, J., and N. Sakimura, "JSON Web Signature (JWS)", RFC 7515, DOI 10.17487/RFC7515, May 2015, <https://www.rfc-editor.org/rfc/rfc7515>.

[RFC8693](https://tools.ietf.org/html/rfc8693)
    Jones, M., Nadalin, A., Campbell, B., Ed., Bradley, J., and C. Mortimore, "OAuth 2.0 Token Exchange", RFC 8693, DOI 10.17487/RFC8693, January 2020, <https://www.rfc-editor.org/rfc/rfc8693>.

[RFC9068](https://tools.ietf.org/html/rfc9068)
    Bertocci, V., "JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens", RFC 9068, DOI 10.17487/RFC9068, October 2021, <https://www.rfc-editor.org/rfc/rfc9068>.

[RFC9396](https://datatracker.ietf.org/doc/html/rfc9396)
    T. Lodderstedt, J. Richer, B. Campbell, "OAuth 2.0 Rich Authorization Requests", RFC 9396, DOI 10.17487/RFC9396, May 2023, <https://www.rfc-editor.org/rfc/rfc9396>.  

[OAUTH-TXN-TOKENS](https://datatracker.ietf.org/doc/draft-tulshibagwale-oauth-transaction-tokens)
     Atul Tulshibagwale, George Fletcher, Pieter Kasselman, "OAuth Transaction Tokens", <https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html>


--- back

# Acknowledgments
name: Dr. Chunchi (Peter) Liu 
email: Liuchunchi(Peter) <liuchunchi=40huawei.com@dmarc.ietf.org>

# Contributors
name: Atul Tulshibagwale
org: SGNL
email: atul@sgnl.ai
