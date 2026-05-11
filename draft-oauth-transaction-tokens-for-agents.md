---
title: "Transaction Tokens For Agents"
category: info

docname: draft-araut-oauth-transaction-tokens-for-agents-01
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 1
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
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

informative:

...

--- abstract

This document specifies an extension to the [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) to support agent context propagation within Transaction Tokens for agent-based workloads. The extension defines the use of the `act` field to identify the agent performing the action, and leverages the existing `sub` field (as defined in the base Transaction Tokens specification) to represent the principal. The `sub` field is populated according to the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), based on the 'subject_token' provided in the token request. For autonomous agents operating independently, the `sub` field represents the agent itself. These mechanisms enable services within the call graph to make more granular access control decisions, thereby enhancing security. 

--- middle

# Introduction

   Traditional zero trust authorization systems face new challenges when
   applied to AI agent workloads. Unlike conventional web services,
   AI agents possess capabilities for autonomous operation, behavioral
   adaptation, and dynamic integration with various data sources. These
   characteristics may lead to decisions that extend beyond their
   initial operational boundaries.

   Existing zero trust models, which effectively manage permissions and
   access scopes for traditional web services, require enhancement to
   address the unique properties of AI agents. Authorization systems
   must evaluate each AI agent interaction independently, considering
   both the immediate context and intended action. This necessitates
   more sophisticated approaches to policy enforcement, behavioral
   monitoring, and audit tracking to maintain security governance.

   Transaction Tokens (Txn-Tokens) are short-lived, signed JSON Web
   Tokens [RFC7519](https://tools.ietf.org/html/rfc7519) that convey identity and authorization context.
   However, the current Txn-Token format lacks sufficient context for
   services within the call chain to implement fine-grained access
   control policies for agent-based workflows. Specifically, it does
   not provide adequate information about the AI agent's identity or
   its initiating entity, limiting transaction traceability. With this
   extension, Transaction Tokens will carry agent identity information
   which will help in better traceability for AI Agent's actions
   deep down the web service graph connecting multiple web services
   involved in completing a transaction in distributed systems.

   This document defines three new contexts within the Transaction Token
   to address these limitations:

   1. The `act` claim, which identifies the AI agent performing the action, aligning with OAuth 2.0 Token Exchange [RFC8693](https://tools.ietf.org/html/rfc8693) terminology for actor tokens 

   2. The `sub` claim, as defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), which represents the principal on whose behalf the transaction is being performed. The population of this field follows the rules specified in the base Transaction Tokens specification, based on the 'subject_token' provided in the token request.

   3. An optional `agentic_ctx` claim. The value of this claim, if present, MUST be a JSON object. The `agentic_ctx` claim conveys attributes about the agent and its operational constraints that are relevant to authorization, auditing, and policy evaluation.
   
   This extension leverages the existing Txn-Token infrastructure to
   enable secure propagation of AI agent context throughout the
   service graph. 

## Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119) [RFC8174](https://datatracker.ietf.org/doc/html/rfc8174) when, and only when, they appear in all capitals, as shown here.

# Terminology

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


# Protocol overview

## Transaction Flow

   This section describes the process by which an agent application
   obtains a Transaction Token, either acting autonomously or on behalf
   of a principal. The external endpoint requests a Txn-Token following
   the procedures defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), augmented with additional
   context for agent identity and, when applicable, principal identity.

## Agent Application Transaction Flows

   The Transaction Token creation process varies depending on the
   presence of a principal.

### Principal-Initiated Flow

   When a principal initiates the workflow, the following steps occur:

   1. The principal invokes the agent application to perform a task.

   2. The agent application calls an external endpoint. External endpoint throws back OAuth challenges.

   3. The agent application authenticates using an OAuth 2.0 Auth code flow [RFC6749](https://tools.ietf.org/html/rfc6749)
      access token. The access token contains subject and clientId claims as per [RFC9068](https://datatracker.ietf.org/doc/rfc9068).

   4. The external endpoint submits the received access token along with its Subject token to the
      Txn-Token Service. Subject token requirements are specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).
      

   5. The Txn-Token Service validates the access token.

   6. The Txn-Token Service populates the Txn-Token's `sub` claim following the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html). The `sub` claim is determined based on the subject_token provided in the request, according to the conditions and rules defined in the base Transaction Tokens specification. This ensures that the principal is properly represented in the Txn-Token. 

   7. The Txn-Token Service copies the access token's `clientId` claim to the Txn-Token's `act` field. Any nested structure within the `clientId` claim is preserved. If the access token contains an `act` claim, that value MAY be used instead of `clientId`. 

   8. The Txn-Token Service issues the Txn-Token to the requesting workload. 

### Autonomous Flow

   When the agent application operates autonomously, the following
   steps occur:

   1. The agent application initiates a task based on an event or
      scheduled assignment.

   2. The agent application calls an external endpoint. OAuth challenge flow starts.

   3. The agent application authenticates using an OAuth 2.0 [RFC6749](https://tools.ietf.org/html/rfc6749). When an autonomous agent
      (no human resource owner) needs to call another resource server using OAuth,
      it follows the Client Credentials Grant defined explicitly in [RFC6749](https://tools.ietf.org/html/rfc6749).

   4. The agent application uses the access token to call the external endpoint.

   5. The external endpoint submits the received access token along with its Subject token to the
      Txn-Token Service. Subject token requirements are specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).

   6. The Txn-Token Service validates the access token.

   7. The Txn-Token Service populates the Txn-Token's `sub` claim following the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html). The `sub` claim is determined based on the subject_token provided in the request. For autonomous agents, this typically represents the agent's own identity. 

   8. The Txn-Token Service copies the access token's `sub` or `clientId` claim to the Txn-Token's `act` field. Any nested structure is preserved. The `act` field identifies the agent performing the autonomous action. 


## Flow Diagrams

### Principal-Initiated Flow

Based on the updated flow, here's a more detailed RFC-style flow diagram:

~~~ ascii-art
Principal    Agent App    External    Authorization   Txn-Token
                         Endpoint        Server        Service
   |            |           |              |             |
   | Invoke     |           |              |             |
   | agent task |           |              |             |
   |----------->|           |              |             |
   |            |           |              |             |
   |            | Call external API        |             |
   |            |---------->|              |             |
   |            |           |              |             |
   |            |   OAuth Challenge        |             |
   |            |<----------|              |             |
   |            |           |              |             |
   |            | Initiate Auth Code Flow  |             |
   |            |------------------------->|             |
   |            |           |              |             |
   |            | Auth Code                |             |
   |            |<-------------------------|             |
   |            |           |              |             |
   |            | Exchange code for token  |             |
   |            |------------------------->|             |
   |            |           |              |             |
   |            | Access Token (AT1)       |             |
   |            | w/ sub, clientId claims  |             |
   |            |<-------------------------|             |
   |            |           |              |             |
   |            | Call with AT1            |             |
   |            |---------->|              |             |
   |            |           |              |             |
   |            |           | Request Txn-Token          |
   |            |           | with AT1, Subject token    |
   |            |           | as param     |             |
   |            |           |--------------------------->|
   |            |           |              |             |
   |            |           |              |    Validate AT1
   |            |           |              |    Extract claims
   |            |           |              |    Set sub from Subject token
   |            |           |              |    Set act from AT1.clientId
   |            |           |              |             |
   |            |           |              |             |
   |            |           |              |             |
   |            |           |              |             |
   |            |           | Txn-Token    |             |
   |            |           |<---------------------------|
   |            |           |              |             |

Legend:
----> : Request flow
<---- : Response flow
  |   : Component boundary
~~~

Notes:
1. AT1 refers to the access token obtained by Agent App
2. The External Endpoint uses its own access token to call Txn-Token Service
3. AT1 is passed as a parameter in the Txn-Token request
4. The flow shows detailed OAuth 2.0 Authorization Code flow steps
5. Token validation and claim extraction steps are shown in the Txn-Token Service


### Autonomous Flow


~~~ ascii-art
Agent App    External    Authorization   Txn-Token
            Endpoint        Server        Service
    |           |              |             |
    | Self-     |              |             |
    | triggered |              |             |
    | event     |              |             |
    |--+        |              |             |
    |  |        |              |             |
    |<-+        |              |             |
    |           |              |             |
    | Call external API        |             |
    |---------->|              |             |
    |           |              |             |
    |   OAuth Challenge        |             |
    |<----------|              |             |
    |           |              |             |
    | Client Credentials Grant |             |
    |------------------------->|             |
    |           |              |             |
    | Access Token (AT1)       |             |
    |  sub, aud claims         |             |
    |<-------------------------|             |
    |           |              |             |
    | Call with AT1            |             |
    |---------->|              |             |
    |           |              |             |
    |           | Request Txn-Token          |
    |           | with AT1, Subject token    |
    |           | as param     |             |
    |           |--------------------------->|
    |           |              |             |
    |           |              |    Validate AT1
    |           |              |    Extract claims
    |           |              |    Set sub from aud
    |           |              |    Set act.sub from clientId or sub
    |           |              |             |
    |           |              |             |
    |           |              |             |
    |           | Txn-Token    |             |
    |           |<---------------------------|
    |           |              |             |

Legend:
----> : Request flow
<---- : Response flow
  |   : Component boundary
  +   : Internal process
--+   : Self-triggered event

Notes:
* AT1: Access token obtained via Client Credentials Grant
* External Endpoint uses subject token for authenticating itself to Txn-Token Service
* AT1 is included as parameter in Txn-Token request
* Self-triggered events can be scheduled tasks or external triggers
* Token validation includes signature and claims verification
~~~

## Replacement tokens

Txn-Token Service provides capability to get a replacement Txn-Token as defined in the [OAUTH-TXN-TOKENS.replacement flow](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html#name-creating-replacement-txn-to). If the original Txn-Token used to get replacement token contains 'actor' and 'principal' claims then in the replaced Txn-Token, the values of the 'actor' and 'principal' MUST remain unchanged similar to 'txn', `sub` and 'aud' claims.

## Txn-Token Format

### JWT Header

No changes to the JWT header from the base specification: `typ` MUST be `txntoken+jwt`, with a signing key identifier such as `kid`.

### JWT Body Claims

The Txn-Token body augments the base claim set with the `act` field for agent context. Existing claims like txn, sub, aud, iss, iat, exp, scope, tctx, and req_wl retain identical semantics, population rules, and immutability guarantees as defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html).

In this example, the agent is 3rd party and not part of trust domain. It hits API Gateway in trust domain and API Gateway requests Txn-Token from Txn-token Service using
access token received from 3P agent and its own subject token (to authenticate with Txn-Token Service). Requesting workload is API Gateway. Agent is agent-identity-1 (clientId in the access token issued to 3P agent to act on behalf of user:alice)

~~~ json
{
  "txn": "c2dc3992-2d65-483a-93b5-2dd9f02c276e",
  "sub": "user:alice@example.com", // if its human initiated
  "aud": "https://trading.trust-domain.example/stocks",
  "iss": "https://txn-svc.trust-domain.example",
  "iat": 1697059200,
  "exp": 1697059500,
  "purp": "trade.stocks",
  "tctx": {
    "action": "BUY",
    "ticker": "MSFT",
    "quantity": "100"
  },
  "req_wl": "apigateway.trust-domain.example", // API gateway requests Txn-token
  "act": {
     "sub":"agent-identity-1" // 3P agent hitting API gateway owned by trust domain
  }
}
~~~

## Agentic Context

The Txn-Token MAY contain an `agentic_ctx` claim. Txn-Tokens are increasingly used in environments where transactions are executed by or with the assistance of autonomous or semi-autonomous agents (for example, Large Language Model (LLM)–based agents, workflow orchestrators, and policy-driven automation components). In such deployments, relying exclusively on subject identity and generic transaction parameters is insufficient to make robust authorization decisions. Additional information about the agent that is interpreting and acting on the transaction is often required.

### Field definitions and population
All fields are OPTIONAL as some of these fields may not be avaiable for 3P (third party agents) connecting to your external service (edge) within trust domain. However, for internal agents within trust domain, you MAY get following information. For 3P agents outside your trust domain, as called out above, you will be able to get `act` claim information using access token and rest of the below fields in `agentic_context` may or may not be available.

* prov (Provenance): Defines the "Behavioral DNA" of the agent. The `manifest_hash` is a cryptographic digest of the agent’s system instructions and core logic, ensuring the agent’s "guardrails" have not been modified. The `manifest_hash` is an opaque key. Resource Servers are expected to resolve this hash against a local or remote policy store to determine the specific behavioral guardrails applied to the agent at that version.
* posture (Environmental Integrity): Details the security tier of the runtime. This includes hardware-backed proof (e.g., TEE) that the agent is isolated from the host OS or cloud provider.
* identity (Workload Origin): Captures the specific machine-actor instance. The `workload_id` distinguishes the instrument (the agent software) from the subject (the end-user).

The `agentic_ctx` claim is populated during the token exchange process by the Transaction Token Service (TTS), which serves as the authoritative source for the agent’s operational identity. This context MAY be derived from various but not limited to sources such as : (1) Static Manifests, which include cryptographic hashes of the agent’s system prompt, model configuration, and registered source code identifiers; (2) Environmental Posture, consisting of telemetry injected by the execution environment or API gateway, such as the hardware security level (e.g., TEE measurements) or network origin; and (3) Workload Mapping, where attributes are mirrored from the agent’s primary workload identity (e.g., SPIFFE or OIDC claims). By centralizing this population at the TTS, the `agentic_ctx` provides a consistent and tamper-resistant representation of the agent’s "persona." This allows downstream resource servers to evaluate the agent’s integrity and origin against local safety policies independently of the specific actions or permissions requested in the transaction.

To ensure the integrity of the `agentic_ctx`, the Transaction Token Service (TTS) MUST not rely on self-reported data from the agent. Instead, it populates these fields through a Verified Exchange model. Hardware-backed fields like `posture` and `tee` are derived from cryptographic Attestation Documents generated by the agent's execution environment (e.g., a Trusted Execution Environment) and verified by the TTS against cloud provider roots of trust. Software-related fields, such as the `manifest_hash`, are retrieved via a Registry-First approach: the TTS performs an out-of-band lookup in a secure Agent Registry using the agent’s authenticated client_id, ensuring that the "behavioral fingerprint" in the token matches the developer’s registered configuration rather than a potentially tampered runtime state. Finally, the identity claims are mirrored from the transport layer (e.g., mTLS certificates or SPIFFE SVIDs), binding the token to the specific verified workload instance.

### Example of `agentic_ctx` with additional context

~~~ json
{
  "agentic_ctx": {
    "prov": {
      "manifest_hash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "model_id": "llama-3.1-70b-v1",
      "version": "2.4.1"
    },
    "posture": {
      "tee": "aws-nitro-enclave",
      "assurance": "high",
      "boot_gold": true
    },
    "identity": {
      "workload_id": "spiffe://prod.acme.com/billing-agent",
      "origin_node": "node-77-east-1"
    }
  }
}
~~~

### Implementation Note: Integrity and Resolution
The fields within `agentic_ctx` represent a "Statement of Posture" rather than a set of permissions. To avoid authorization failure, implementations should ensure that:
* Registry Synchronization: The `manifest_hash` is treated as a lookup key. Resource Servers should maintain or have access to a known-good database of hashes to map cryptographic signatures to behavioral guardrails.
* Hardware Roots of Trust: When `posture` claims are present, the Transaction Token Service MUST verify the underlying attestation document against the hardware manufacturer's public keys.
* Non-Collusion: The `agentic_ctx` is distinct from the `sub` claim. While the `sub` identifies the authorizing principal, the `agentic_ctx` identifies the machine-actor. Authorization logic SHOULD evaluate the intersection of both identities.

# Multi-agent flows
There can be AI agents within the trust domain instead of traditional workloads. In complex agentic workflows within the trust domain , a primary agent (the "Delegator") may delegate sub-tasks to one or more secondary agents ("Delegatees"). This document defines a mechanism to preserve the delegation lineage across these transitions. Note that preserving lineage is optional.

## Agent-to-Agent Delegation 
When an agent (the "Delegator") invokes another agent (the "Delegatee") to perform a sub-task, it SHOULD NOT pass its own Transaction Token to the Delegatee. Instead, the Delegator MUST obtain a narrowed Transaction Token for the Delegatee using the replacement flow defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html). The Delegator SHOULD request a restricted scope or purp (Purpose) that is specific to the sub-task assigned to the Delegatee. 

## The 'actchain' (Actor Chain) Claim  

The `actchain` claim is an OPTIONAL top-level JSON Web Token (JWT) claim that provides a cryptographic trace of the delegation path. It is represented as an ordered array of JSON objects, where each object represents a previous agent in the call chain.

### Claim structure
Each object within the `actchain` array MUST contain the following members:
* **sub (Subject)**: REQUIRED. The identity of the delegating agent.
* **iat (Issued At)**: OPTIONAL. A timestamp indicating when the delegating agent initiated its portion of the transaction.
* **iss (Issuer)**: OPTIONAL. The issuer of the token that identified the delegating agent. This might be required in case of multiple Txn Token Services being present across domains and token is passed across domains.

### Delegation via Replacement Flow
When an agent requests a narrowed Transaction Token for a sub-agent, the Transaction Token Service (TTS) MUST follow the replacement flow procedures defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) with the following modifications:

* **Subject Immutability**: The txn and `sub` (principal) claims MUST be copied from the subject_token to the new Transaction Token without modification.
* **Chain Progression**: The TTS MUST extract the `act` claim from the incoming subject_token. This extracted object MUST be appended to the end of the `actchain` array in the new token.
If no `actchain` existed in the subject_token, a new array is created containing only the extracted `act` object.
* **New Actor Assignment**: The top-level act claim of the new token MUST be set to the identity of the Delegatee (the sub-agent).

### Multi-agent example JWT body claims
This example represents a delegated state: a human principal initiated a task via a Researcher Agent, which then delegated a specific action to a Search Agent.

~~~ json
{
  "txn": "c2dc3992-2d65-483a-93b5-2dd9f02c276e",
  "sub": "user-77",
  "iss": "https://txn-svc.trust-domain.example",
  "iat": 1712850000,
  "exp": 1712850300,
  "act": {
    "sub": "search-agent-v2",
    "deployment": "prod-us-west-1"
  },
  "actchain": [
    {
      "sub": "researcher-agent-v1",
      "iat": 1712849950
    }
  ],
  "purp": "web.search.execute",
  "agentic_ctx": {
    "prov": {
      "manifest_hash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "model_id": "llama-3.1-70b-v1",
      "version": "2.4.1"
    },
    "posture": {
      "tee": "aws-nitro-enclave",
      "assurance": "high",
      "boot_gold": true
    },
    "identity": {
      "workload_id": "spiffe://prod.acme.com/billing-agent",
      "origin_node": "node-77-east-1"
    }
  }
}
~~~

# Security Considerations

1. All the security considerations mentioned in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) apply.

2. Token Replay Protection Implementations MUST enforce strict token lifetime validation. The short-lived nature of Transaction Tokens helps mitigate replay attacks, but implementations SHOULD also consider:
   * Implementing token tracking mechanisms within trust domains
   * Validating token usage context

3. Actor Identity Security
   * Implementations MUST validate `act` claims in tokens
   * The Txn-Token Service MUST verify the authenticity of actor context before token issuance
   * During replacement flow, Txn-Token Service MUST NOT modify the `act` field in the incoming Txn-Token

4. Principal Context Protection
   * Systems MUST prevent unauthorized modifications to the `sub` claim during token propagation. Txn-Tokens are cryptographically signed to ensure integrity. 
   * During replacement flow, Txn-Token Service MUST NOT modify the `sub` claim in the incoming Txn-Token
   * The Txn-Token Service MUST follow the subject population rules defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) to ensure proper principal representation 

5. Transaction Chain Integrity
   * Implementations MUST maintain cryptographic integrity of the token chain
   * Services MUST validate tokens at trust domain boundaries
   * Systems MUST implement protection against token tampering during service-to-service communication

6. AI Agent Specific Controls
   * Implementations MUST enforce scope boundaries for AI agent operations
   * Systems SHOULD implement behavioral monitoring for AI agent activities by logging `act` and `sub` claims in audit logs 
   * Systems MUST maintain audit trails of AI agent activities

7. Token Transformation Security
   * The Txn-Token Service MUST validate all claims during access token to Txn-Token conversion
   * Implementations MUST verify signatures and formats of all tokens
   * Systems MUST prevent unauthorized manipulation during token transformation
   * The Txn-Token Service MUST ensure that the `act` field accurately represents the agent identity from the access token

8. Replacement Token Considerations
   * Systems MUST verify the authenticity and validity of original tokens before replacement
   * Systems MUST implement controls to prevent unauthorized replacement requests
   * The immutability of `act` and `sub` claims during replacement ensures consistent identity context throughout the transaction lifecycle 

9. Infrastructure Security
   * All component communications MUST use secure channels
   * Implementations MUST enforce strong authentication of the Authorization Server
   * Systems MUST implement regular rotation of cryptographic keys
   * Trust domain boundaries MUST be clearly defined and enforced
   
10. Multi-Agent Considerations:
    * Chain Depth and Bloat: Deeply nested agent calls can lead to significant JWT size increases, potentially impacting HTTP header limits. The TTS MAY impose a maximum depth for the `actchain`. If the maximum depth is exceeded, the TTS MUST either reject the request or truncate the oldest entries in the chain, provided that a "truncated" flag is added to the claim to alert downstream services of the loss of provenance.
    * Privilege Escalation: A Delegator MUST NOT be able to request a replacement token with broader permissions or a higher-tier principal than what is asserted in its own subject_token. The TTS MUST validate that the requested scope and purp are a logical subset of the original token.
    * The TTS MUST verify that the workload requesting a replacement token is the entity identified in the  `act` claim of the subject_token. This prevents an unauthorized workload from "injecting" itself into a transaction chain or extending a chain it is not part of.
    * During the replacement flow, the TTS MUST NOT allow the modification of the `sub` (principal) or txn claims. These fields provide the "anchor" for the entire transaction; any modification would effectively initiate a new transaction, requiring a fresh authentication event rather than a replacement flow.

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
