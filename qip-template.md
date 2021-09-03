---
qip: <to be assigned at pull request stage, non-padded positive integer>
title: <QIP title> (required)
author: <Author (@githubusername), Author <email> Author (platorm/username), Author> (required, can be plural, format defined in order of preference)
layer: <core, interface, application, meta> (required)
status: <draft[/incomplete,/abandoned,/withdrawn], proposals[/open,/under_discussion,/accepted[/available,/in_development,/awaiting_hardfork,/completed]/deferred,/rejected]> (required)
comments_uri: <to be assigned at pull request stage>
comments_summary_uri: <to be assigned at proposals/(accepted, or rejected) stage> 
created: <ISO 8601 creation date (yyyy-mm-dd)>
requires: <QIP number(s),> (optional)
replaces: <QIP number(s),> (optional)
superseded_by: <QIP number(s), URI> (optional)
---

This is the suggested template for new QIPs which is modeled closely after the Bitcoin Improvement Proposals (BIPs) and Ethereum Improvement Proposals (EIPs).

- To get started, see the [workflow document](qip-workflow.md).
- For the full specification, see [specification](qip-specification.md) page.

See the repos [README.md](README.md) for more information on the structure.

## Abstract

A short (~200 word) description of the technical issue being addressed. This should be a very terse and human-readable version of the specification section. Someone should be able to read only the abstract to get the gist of what this specification does.

## Motivation

The motivation section should describe the "why" of this QIP. What problem does it solve? Why should someone want to implement this standard? What benefit does it provide to the QRL ecosystem? What use cases does this QIP address?

## Specification

The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for any of the current QRL platforms to integrate.

### Rationale

The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

## Backward compatibility

All QIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Reference Implementation

An optional section that contains a reference/example implementation that people can use to assist in understanding or implementing this specification. If the implementation is too large to reasonably be included inline, then consider adding it as one or more files in

## Security Considerations

All QIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. QIP submissions missing the "Security Considerations" section will be rejected. An QIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.