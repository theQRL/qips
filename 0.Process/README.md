# Introduction

Following mainnet-launch, any significant design changes must pass through a QIP process, prior to being implemented in release code. We are a friendly bunch and likely to approve useful and innovative suggestions. 

# Process

## 1. Ideas

QIPs start as ideas: it’s a good idea to share these ideas with the development and wider QRL community.  These ideas live in the “1. Ideas” folder within this repository, and comments on the parent parent pull request should allow refinement of the idea before it becomes a formal proposal.

## 2. Proposals

### 2.1 Open proposals

The first stage of a formal proposal is submission of a new open proposal.  Public comment is invited on open proposals in a similar manner to ideas.  Once the proposal is sufficiently mature it will be moved by the QRL Foundation maintainers to the next stage.

### 2.2 Under discussion

Proposals under discussion will be considered at the next QRL Developer meeting.  Particularly complex or in depth QIPs may require a separate meeting or face-to-face discussions by the QRL development team prior to decisions being made as to the outcome.  As this point they may be returned to 2.1 for further refinement.

### 3a. Accepted

Accepted proposals require development until such a point when they are released or they are ready for release at the next hard fork of the network.  As such, an accepted QIP may be in one of four states:

1. Available (for a developer/team to take on engineering the QIP; avoiding duplication of effort)

2. In development

3. Awaiting hard fork

4. Completed

### 3b. Deferred

Deferred proposals have merit to the project but for operational reasons are held back from entering development.  Deferred proposals may enter development at a later date or may move into a different category as the network matures.

### 3c. Rejected

Rejected proposals will not enter development without commencing the QIP process from the beginning and re-working the proposal.

# How to submit

## Document format

In general, documents should be formatted using the [Markdown](https://daringfireball.net/projects/markdown/syntax) syntax.  There a [plenty](https://www.ossblog.org/markdown-editors/) of very good Markdown editors available.

## Submission

To submit, clone this Github repository, add a sequentially named folder to either the _1. Ideas_ or _2. Proposals / 1. Open_ folder and place a `README.md` Markdown document within the newly created folder.  Include any linked images in the folder then submit a pull request.

Comments on your QIP should be initiated on the pull request.

## Template

A template QIP document is in this folder titled [TEMPLATE_qip-000.md](./TEMPLATE_qip-000.md)