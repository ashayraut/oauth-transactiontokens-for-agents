---
title: "Transaction Tokens For Agents"
category: info

docname: draft-oauth-transaction-tokens-for-agents-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: oauth@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/oauth
  github: "ashayraut/oauth-transactiontokens-for-agents"
  latest: "https://drafts.oauth.net/oauth-transactiontokens-for-agents/draft-oauth-transaction-tokens-for-agents.html"

author:
 -
    fullname: ASHAY RAUT
    organization: Amazon
    email: asharaut@amazon.com

normative:

informative:

...

--- abstract

This document specifies an extension to the OAuth Transaction Tokens
framework (https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html)
to support agent context propagation within Transaction
Tokens for agent-based workloads. The extension defines two new
context fields: 'actor' and 'principal'. The 'actor' field identifies
the agent performing the action, while the 'principal' field identifies
the human or system entity that initiated the agent's action. For
autonomous agents operating independently, the 'principal' field MAY
be omitted. These additional context fields enable services within
the call graph to make more granular access control decisions,
thereby enhancing security.

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
   its initiating entity, limiting transaction traceability.

   This document defines two new contexts within the Transaction Token
   to address these limitations:

   1. The actor context, which identifies the AI agent performing
      the action

   2. The principal context, which identifies the human or system
      entity on whose behalf the actor operates

   This extension leverages the existing Txn-Token infrastructure to
   enable secure propagation of AI agent context throughout the
   service graph.

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

   2. The agent application calls an external endpoint.

   3. The agent application authenticates using an OAuth 2.0 [RFC6749](https://tools.ietf.org/html/rfc6749)
      access token.

   4. The agent application performs OAuth Token Exchange [RFC8693](https://tools.ietf.org/html/rfc8693),
      submitting both actor token and subject token to the
      Authorization Server.

   5. The external endpoint submits the received access token to the
      Txn-Token Service.

   6. The Txn-Token Service validates the access token and extracts
      the principal, actor, and subject identities.

   7. As specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), the Txn-Token Service uses
      the access token's 'aud' claim to populate the Txn-Token's
      'sub' claim.

   8. The Txn-Token Service copies the access token's 'actor' claim
      to the Txn-Token's 'actor' context. Any nested structure within
      the 'actor' claim is preserved.

   9. The Txn-Token Service uses the access token's 'sub' claim to
      populate the Txn-Token's 'principal' context.

### Autonomous Flow

   When the agent application operates autonomously, the following
   steps occur:

   1. The agent application initiates a task based on an event or
      scheduled assignment.

   2. The agent application calls an external endpoint.

   3. The agent application authenticates using an OAuth 2.0 [RFC6749](https://tools.ietf.org/html/rfc6749)
      access token.

   4. The external endpoint submits the received access token to the
      Txn-Token Service.

   5. The Txn-Token Service validates the access token and extracts
      the actor and subject identities.

   6. As specified in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html), the Txn-Token Service uses
      the access token's 'aud' claim to populate the Txn-Token's
      'sub' claim.

   7. The Txn-Token Service copies the 'sub' field from within the
      access token's 'actor' claim to the Txn-Token's 'actor' context.
      Any nested structure is preserved.


## Flow Diagrams

### Principal-Initiated Flow

     Principal    Agent App    External    Txn-Token    Authorization
                              Endpoint     Service         Server
        |            |           |            |             |
        | Invoke     |           |            |             |
        | agent task |           |            |             |
        |----------->|           |            |             |
        |            |           |            |             |
        |            | Call external API      |             |
        |            |---------->|            |             |
        |            |           |            |             |
        |            | Request access token   |             |
        |            |---------------------------------->   |
        |            |           |            |             |
        |            | Token exchange (actor + subject)     |
        |            |<----------------------------------|  |
        |            |           |            |             |
        |            |           | Request    |             |
        |            |           | Txn-Token  |             |
        |            |           |----------->|             |
        |            |           |            |             |
        |            |           | Validate token           |
        |            |           | Extract identities       |
        |            |           | Set sub, actor,          |
        |            |           | principal claims         |
        |            |           |<-----------|             |
        |            |           |            |             |

### Autonomous Flow

     Agent App    External    Txn-Token    Authorization
                 Endpoint     Service         Server
        |           |            |             |
        | Self-     |            |             |
        | triggered |            |             |
        | event     |            |             |
        |--+        |            |             |
        |  |        |            |             |
        |<-+        |            |             |
        |           |            |             |
        | Call external API      |             |
        |---------->|            |             |
        |           |            |             |
        | Request access token   |             |
        |---------------------------------->   |
        |           |            |             |
        | Return access token    |             |
        |<----------------------------------|  |
        |           |            |             |
        |           | Request    |             |
        |           | Txn-Token  |             |
        |           |----------->|             |
        |           |            |             |
        |           | Validate token          |
        |           | Extract actor/subject   |
        |           | Set sub and actor      |
        |           |<-----------|             |
        |           |            |             |

Legend:
```
----> : Request flow
<---- : Response flow
   +  : Internal process
--+   : Self-triggered event
```

## Replacement tokens
Txn-Token Service provides capability to get a replacement Txn-Token as defined in the [OAUTH-TXN-TOKENS.replacement flow](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html#name-creating-replacement-txn-to). If the original Txn-Token used to get replacement token contains 'actor' and 'principal' claims then in the replaced Txn-Token, the values of the 'actor' and 'principal' MUST remain unchanged similar to 'txn', 'sub' and 'aud' claims.


## Security Considerations

All the security considerations mentioned in [OAUTH-TXN-TOKENS](https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html) apply.

## References

### Normative References
[RFC6749](https://tools.ietf.org/html/rfc6749)
    Hardt, D., Ed., "The OAuth 2.0 Authorization Framework", RFC 6749, DOI 10.17487/RFC6749, October 2012, <https://www.rfc-editor.org/rfc/rfc6749>. 
    
[RFC7519](https://tools.ietf.org/html/rfc7519)
    Jones, M., Bradley, J., and N. Sakimura, "JSON Web Token (JWT)", RFC 7519, DOI 10.17487/RFC7519, May 2015, <https://www.rfc-editor.org/rfc/rfc7519>. 
    
[RFC7515]((https://tools.ietf.org/html/rfc7515)
    Jones, M., Bradley, J., and N. Sakimura, "JSON Web Signature (JWS)", RFC 7515, DOI 10.17487/RFC7515, May 2015, <https://www.rfc-editor.org/rfc/rfc7515>. 
    
[RFC8693]((https://tools.ietf.org/html/rfc8693)
    Jones, M., Nadalin, A., Campbell, B., Ed., Bradley, J., and C. Mortimore, "OAuth 2.0 Token Exchange", RFC 8693, DOI 10.17487/RFC8693, January 2020, <https://www.rfc-editor.org/rfc/rfc8693>. 
    
[RFC9068]((https://tools.ietf.org/html/rfc9068)
    Bertocci, V., "JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens", RFC 9068, DOI 10.17487/RFC9068, October 2021, <https://www.rfc-editor.org/rfc/rfc9068>. 
    
[OAUTH-TXN-TOKENS](https://datatracker.ietf.org/doc/draft-tulshibagwale-oauth-transaction-tokens)
     Atul Tulshibagwale, George Fletcher, Pieter Kasselman, "OAuth Transaction Tokens", <https://drafts.oauth.net/oauth-transaction-tokens/draft-ietf-oauth-transaction-tokens.html>
    

--- back

# Acknowledgments
{:numbered="false"}
The authors would like to thank the contributors and the OAuth working group members who gave valuable input to this draft.

# Contributors
name: Atul Tulshibagwale
org: SGNL
email: atul@sgnl.ai
