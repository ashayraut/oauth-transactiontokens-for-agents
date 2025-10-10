---
title: "Transaction Tokens For Agents"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-oauth-transaction-tokens-for-agents-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 1
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

