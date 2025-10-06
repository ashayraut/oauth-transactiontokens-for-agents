---
title: "Transaction Tokens For Agents"
abbrev: "TODO - Abbreviation"
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
  arch: [https://example.com/WG](https://mailarchive.ietf.org/arch/browse/oauth/)
  github: [USER/REPO](https://github.com/ashayraut/oauth-transactiontokens-for-agents)
  latest: https://example.com/LATEST

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
   Tokens [RFC7519] that convey identity and authorization context.
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



# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
