---
title: "Transaction Tokens For Agents"
category: info

docname: draft-oauth-transaction-tokens-for-agents-06
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 5
# area: AREA
# workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Working Group
#  mail: oauth@ietf.org
#  arch: https://mailarchive.ietf.org/arch/browse/oauth
  github: "ashayraut/oauth-transactiontokens-for-agents"
  latest: "https://ashayraut.github.io/oauth-transactiontokens-for-agents/draft-oauth-transaction-tokens-for-agents.html"

author:
 -
    fullname: ASHAY RAUT
    organization: Amazon
    email: asharaut@amazon.com

normative:

informative:

...

--- abstract

This document specifies an extension to the [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) to support agent context propagation within Transaction Tokens for agent-based workloads. The extension defines the use of the 'act' field to identify the agent performing the action, and leverages the existing 'sub' field (as defined in the base Transaction Tokens specification) to represent the principal. The 'sub' field is populated according to the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), based on the 'subject_token' provided in the token request. For autonomous agents operating independently, the 'sub' field represents the agent itself. These mechanisms enable services within the call graph to make more granular access control decisions, thereby enhancing security. 

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

   1. The act claim, which identifies the AI agent performing the action, aligning with OAuth 2.0 Token Exchange [RFC8693](https://tools.ietf.org/html/rfc8693) terminology for actor tokens 

   2. The sub claim, as defined in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), which represents the principal on whose behalf the transaction is being performed. The population of this field follows the rules specified in the base Transaction Tokens specification, based on the 'subject_token' provided in the token request.

   3. An optional agentic_ctx claim. The value of this claim, if present, MUST be a JSON object. The agentic_ctx claim conveys attributes about the agent and its operational constraints that are relevant to authorization, auditing, and policy evaluation.
   
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

   6. The Txn-Token Service populates the Txn-Token's 'sub' claim following the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html). The 'sub' claim is determined based on the subject_token provided in the request, according to the conditions and rules defined in the base Transaction Tokens specification. This ensures that the principal is properly represented in the Txn-Token. 

   7. The Txn-Token Service copies the access token's 'clientId' claim to the Txn-Token's 'act' field. Any nested structure within the 'clientId' claim is preserved. If the access token contains an 'act' claim, that value MAY be used instead of 'clientId'. 

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

   7. The Txn-Token Service populates the Txn-Token's 'sub' claim following the rules specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html). The 'sub' claim is determined based on the subject_token provided in the request. For autonomous agents, this typically represents the agent's own identity. 

   8. The Txn-Token Service copies the access token's 'sub' or 'clientId' claim to the Txn-Token's 'act' field. Any nested structure is preserved. The 'act' field identifies the agent performing the autonomous action. 


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
   |            | sub, clientId claims     |             |
   |            |<-------------------------|             |
   |            |           |              |             |
   |            | Call with AT1            |             |
   |            |---------->|              |             |
   |            |           |              |             |
   |            |           | Request Txn-Token          |
   |            |           | with AT1 as param          |
   |            |           |--------------------------->|
   |            |           |              |             |
   |            |           |              |    Validate AT1
   |            |           |              |    Extract claims
   |            |           |              |    Set sub from aud
   |            |           |              |    Set actor from
   |            |           |              |    clientId
   |            |           |              |    Set principal
   |            |           |              |    from sub
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
    |           | with AT1 as param          |
    |           |--------------------------->|
    |           |              |             |
    |           |              |    Validate AT1
    |           |              |    Extract claims
    |           |              |    Set sub from aud
    |           |              |    Set actor from
    |           |              |    sub in actor
    |           |              |    claim
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
* External Endpoint uses its own credentials for Txn-Token Service
* AT1 is included as parameter in Txn-Token request
* Self-triggered events can be scheduled tasks or external triggers
* Token validation includes signature and claims verification
~~~

## Replacement tokens

Txn-Token Service provides capability to get a replacement Txn-Token as defined in the [OAUTH-TXN-TOKENS.replacement flow](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html#name-creating-replacement-txn-to). If the original Txn-Token used to get replacement token contains 'actor' and 'principal' claims then in the replaced Txn-Token, the values of the 'actor' and 'principal' MUST remain unchanged similar to 'txn', 'sub' and 'aud' claims.

## Txn-Token Format

### JWT Header

No changes to the JWT header from the base specification: `typ` MUST be `txntoken+jwt`, with a signing key identifier such as `kid`.

### JWT Body Claims

The Txn-Token body augments the base claim set with two new top-level claims for agent context: `actor` and `principal`. Existing claims like `txn`, `sub`, `aud`, `iss`, `iat`, `exp`, `purp`, `tctx`, and `req_wl` retain identical semantics, population rules, and immutability guarantees.

~~~ json
{
  "txn": "c2dc3992-2d65-483a-93b5-2dd9f02c276e",
  "sub": "api-gw.trust-domain.example",
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
  "req_wl": "apigateway.trust-domain.example",
  "actor": {
    "agent_id": "agent-1234",
    "version": "v2.1.0",
    "deployment": "prod-us-east-1"
  },
  "principal": "user:alice@example.com"
}
~~~

## Agentic Context

The Txn-Token MAY contain an agentic_ctx claim. Txn-Tokens are increasingly used in environments where transactions are executed by or with the assistance of autonomous or semi-autonomous agents (for example, Large Language Model (LLM)–based agents, workflow orchestrators, and policy-driven automation components). In such deployments, relying exclusively on subject identity and generic transaction parameters is insufficient to make robust authorization decisions. Additional information about the agent that is interpreting and acting on the transaction is often required.

~~~ json
"agentic_ctx": {
  "agent_type": "planner+tool-orchestrator", // A string describing the functional role of the agent (for example, “planner”, “tool-orchestrator”, “data-assistant”, “code-execution-agent”). The semantics and allowed values are deployment-specific.
  "agent_version": "3.4.2", // A string indicating a version or configuration identifier for the agent. This value can be used to associate the transaction with a particular, reviewed agent policy or release
  "intent": "enumerate and validate production search services before Q4 traffic spike", // A string describing the high-level purpose of the transaction from the agent’s perspective (for example, “trade.stocks”, “enumerate.search.services”, “generate.billing.report”). This value is intended to support coarse-grained, intent-aware authorization policies.
  "allowed_actions": ["read"],
  "environment_constraints": { "environment": "prod", "region": "us" },
}
~~~

### Integration with OAuth Rich Authorization Requests

When the Authorization Server supports Rich Authorization Requests (RAR) as defined in [RFC9396](https://datatracker.ietf.org/doc/html/rfc9396), the authorization details captured during the authorization flow can provide valuable context for downstream authorization decisions. The RAR mechanism enables clients to specify fine-grained authorization requirements that may be captured, reviewed, and potentially consented to by the resource owner during the authorization process.

Authorization Servers implementing RAR MAY include relevant authorization details within the access token. When the Txn-Token Service processes such access tokens to issue Transaction Tokens, it MAY extract these authorization details and include them within the agentic_ctx claim. This approach enables services deeper in the call chain to leverage authorization details for fine-grained access control decisions, even when the Authorization Server itself does not enforce policies based on all captured details.

This pattern offers several advantages:
Deferred Policy Enforcement: Authorization details can be captured and validated at the Authorization Server without requiring immediate policy decisions on all details. Fine-grained authorization policies can be enforced closer to the resources being accessed.

1. Context Enrichment: The authorization details provide additional context about the scope and nature of the authorized operation, enabling more informed authorization decisions throughout the service graph.

2. Consent Propagation: When user consent is obtained for specific authorization details, this consent context can be propagated through the Transaction Token to services that need to honor those consent decisions.

3. Reduced Complexity: This approach allows businesses to avoid the complexity of implementing all fine-grained authorization checks at the Authorization Server, instead distributing authorization decisions to services with domain-specific knowledge.

For example, an Authorization Server might capture detailed authorization requirements using RAR, obtain necessary user consent, and include these details in the access token. The Txn-Token Service can then extract relevant authorization details and include them in the agentic_ctx claim. Services receiving Transaction Tokens with authorization details in the agentic_ctx can use this information to make context-aware authorization decisions that respect the original authorization scope, user consent, and intended purpose of the operation. Implementations SHOULD carefully consider which authorization details are relevant for downstream services and avoid including sensitive information that is not necessary for authorization decisions in the call chain.

# Security Considerations

1. All the security considerations mentioned in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) apply.

2. Token Replay Protection Implementations MUST enforce strict token lifetime validation. The short-lived nature of Transaction Tokens helps mitigate replay attacks, but implementations SHOULD also consider:
   2.1 Implementing token tracking mechanisms within trust domains
   2.2 Validating token usage context

4. Actor Identity Security
   3.1. Implementations MUST validate actor claims in tokens
   3.2. The Txn-Token Service MUST verify the authenticity of actor context before token issuance
   3.3. During replacement flow, Txn-Token Service MUST avoid replacing actor context in the incoming Txn-Token.

4. Principal Context Protection
   4.1. Systems MUST prevent unauthorized modifications to principal context during token propagation. Txn-Token is cryptographically signed.
   4.3. During replacement flow, Txn-Token Service MUST avoid replacing principal context in the incoming Txn-Token.

5. Transaction Chain Integrity
   5.1. Implementations MUST maintain cryptographic integrity of the token chain
   5.2. Services MUST validate tokens at trust domain boundaries
   5.3. Systems MUST implement protection against token tampering during service-to-service communication

6. AI Agent Specific Controls
   6.1. Implementations MUST enforce scope boundaries for AI agent operations
   6.2. Systems SHOULD implement behavioral monitoring for AI agent activities by logging actor, principal in logs.
   6.3. Systems MUST maintain audit trails of AI agent activities

7. Token Transformation Security
   7.1. The Txn-Token Service MUST validate all claims during access token to Txn-Token conversion
   7.2. Implementations MUST verify signatures and formats of all tokens
   7.3. Systems MUST prevent unauthorized manipulation during token transformation

8. Replacement Token Considerations
   8.1. Systems MUST verify the authenticity and validity of original tokens before replacement
   8.2. Systems MUST implement controls to prevent unauthorized replacement requests

9. Infrastructure Security
   9.1. All component communications MUST use secure channels
   9.2. Implementations MUST enforce strong authentication of the Authorization Server
   9.3. Systems MUST implement regular rotation of cryptographic keys
   9.4. Trust domain boundaries MUST be clearly defined and enforced


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
The authors would like to thank the contributors and the OAuth working group members who gave valuable input to this draft.

# Contributors
name: Atul Tulshibagwale
org: SGNL
email: atul@sgnl.ai
