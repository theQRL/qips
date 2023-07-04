# QIP Workflow <!-- omit in toc -->

- [1. Considerations and Governance](#1-considerations-and-governance)
  - [1.1. Scope Considerations](#11-scope-considerations)
  - [1.2. Governance](#12-governance)
- [2. Shepherding your QIP](#2-shepherding-your-qip)
  - [2.1. Drafting your idea](#21-drafting-your-idea)
  - [2.2. Facilitate Discussions](#22-facilitate-discussions)
  - [2.3. Request for proposal](#23-request-for-proposal)
  - [2.4. Proposal/open](#24-proposalopen)
  - [2.5. Proposal review and summary](#25-proposal-review-and-summary)
- [3. Process Specification](#3-process-specification)
  - [3.1. Fork and setup your QIP Repository](#31-fork-and-setup-your-qip-repository)
    - [3.1.1. Clone your local repository](#311-clone-your-local-repository)
    - [3.1.2. Set theQRL organization's repository as upstream](#312-set-theqrl-organizations-repository-as-upstream)
    - [3.1.3. Setup your local environment](#313-setup-your-local-environment)
  - [3.2. Starting a draft](#32-starting-a-draft)
    - [3.2.1. Copy the reference QIP](#321-copy-the-reference-qip)
    - [3.2.2. Add anything needed from the specification](#322-add-anything-needed-from-the-specification)
  - [3.3. Submit your draft](#33-submit-your-draft)
    - [3.3.1. Commit and push](#331-commit-and-push)
    - [3.3.2. Submit your pull request](#332-submit-your-pull-request)
  - [3.4. Making adjustments](#34-making-adjustments)
    - [3.4.1. Fetch and merge upstream changes](#341-fetch-and-merge-upstream-changes)
    - [3.4.2. Make changes, commit, and push](#342-make-changes-commit-and-push)
    - [3.4.3. Submit a new pull request](#343-submit-a-new-pull-request)
- [4. Getting help](#4-getting-help)

# 1. Considerations and Governance

## 1.1. Scope Considerations

Understand that QRL Improvement Proposals are design documents which govern the core structure of the QRL ecosystem. This includes the following layers:

- **core**: Improvements requiring a consensus fork.
  - **networking**: Improvements around network components
  - **security**: Improvements and upgrades to to the security.
- **Interface**: Improvements around APIs
- **meta**: Self referential to the QIP Process, governance, and structure.

Anything outside of these layers will result in a rejected QIP.

## 1.2. Governance

Anyone can and is encouraged to submit a QRL Improvement Proposal (QIP) if they have an idea to improve the QRL core protocol.

Votes are loosely coupled and takes place on-chain by taking a snapshot of each addresses funds and distributing a proportional amount of tokens. Known organizational entities, such as exchanges, and funds from the foundation, are excluded from this proportional distribution.

When a proposal is opened, the snapshot and voting blockheight are declared along with the minimum voting threshold and turnout. Destination addresses are created, where the distributed tokens can be sent to 'register' the voters decision.

# 2. Shepherding your QIP

Parties involved in the process are you, the champion(s) and QIP author(s), the QIP editors, the QRL community, and the QRL contributors.

QIPs start as ideas: it’s a good idea to share these ideas with the parties involved before starting. Ask the QRL community first if an idea is original to avoid wasting time on something that will be rejected based on prior research.

## 2.1. Drafting your idea

Once it's determined that the idea is original and with some merit, begin by drafting your proposal from the reference QIP and following the process specification in this document below.

## 2.2. Facilitate Discussions

Facilitate discussions and adjust your QIP in accordance with the discussions that have taken place.

While you're championing for your own QIP, skill will be required to keep discussions positive, forward-thinking, and in good faith in order to consider all viewpoints.

Thoroughly understanding all viewpoints will allow you to refine your draft into a proposal that's more likely to gain community consensus. 

## 2.3. Request for proposal

After a minimum of a two week period, and when you're ready for the draft to reach a proposal stage, put in a proposal request to the authors as part of the reference QIP. The QIP editors will then review the draft

Assuming no adjustments are requested, the editors will evaluate and propose an on-chain governance specification.

At this point, the QIP will move from `draft/incomplete` to `proposal/open`.

Special note: While the QIP editors endeavor to achieve on-chain governance for every QIP, it's possible a proposal may not be suitable due to unforeseen factors. 

## 2.4. Proposal/open

Discussion will continue on the open proposal through to the final consensus blockheight. This is a process that can last months, depending on the scope of the proposal.

## 2.5. Proposal review and summary

Whether or not consensus is reached, a summary of the proposal is made and the discussion locked.

If consensus is reached at it passes review, the QIP will be set as a recommendation and slated for development depending on resources available.

# 3. Process Specification

The process involves an understanding of git, which many find to be cumbersome and may feel more at ease in a GUI, of which [there are a few](https://git-scm.com/downloads/guis).

## 3.1. Fork and setup your QIP Repository

Login to GitHub and fork the [QIP repository](https://github.com/theQRL/qips) by clicking on the fork in the top right hand corner of your browser.

### 3.1.1. Clone your local repository

```
git clone https://github.com/[YOUR-USERNAME]/qips
```

### 3.1.2. Set theQRL organization's repository as upstream

```
git remote add upstream https://github.com/theQRL/qips
```

### 3.1.3. Setup your local environment

For smoother editing, a local jekyll site can be built.

1. Open terminal
2. Install [rbenv](https://github.com/rbenv/rbenv), if you haven't done so already.
3. Install local Ruby version

```bash
rbenv install
```

4. Install the bundler

```bash
gem install bundler
```

5. Install dependencies

```bash
gem install bundler jekyll
bundle install
```

6. Start the server!

```bash
bundle exec jekyll serve
```

## 3.2. Starting a draft

### 3.2.1. Copy the reference QIP

Copy [qips/reference](qips/reference.md) and rename to "draft" followed a self-descriptive slug. 

Examples:

- `draft-privacy.md`
- `draft-encoding.md`

### 3.2.2. Add anything needed from the specification

Each QIP is preceded and specified by front-matter denoted between two sets of dashes "`---`", which allows for it to be easily parsed by other programs. For quick reference, a the [`qip-template.md`](./qip-template) can be used to fill out the frontmatter along with the reference copied in step 3.2.1.

A more detailed specification is outlined in the file [`qip-specification.md`](./qip-specification.md).

Formatting below the frontmatter follows a WYSIWYM (What You See Is What You Mean) syntax called [Markdown](https://daringfireball.net/projects/markdown/syntax). There a [plenty](https://www.ossblog.org/markdown-editors/) of very good Markdown editors available.

## 3.3. Submit your draft

Once you're satisfied, you'll want to submit a draft PR which consists of committing your files, pushing to your GitHub repository and then creating a pull request from theQRL's GitHub repository.

### 3.3.1. Commit and push

Commit your files:

```
git add [filename]
``` 
or (with caution)
```
git add --all
```

It will open an editor asking you to briefly describe the changes made. 

Once that's saved, you can then push to your repository on GitHub

```
git push
```

### 3.3.2. Submit your pull request

Head over to the [pull requests section of theQRL/qips](https://github.com/theQRL/qips/pulls) repo and click on [New pull request].

## 3.4. Making adjustments

After your draft is merged and discussions take place, you'll want to make some final adjustments before the draft is turned into a proposal.

### 3.4.1. Fetch and merge upstream changes

From the local repo you had before, fetch and merge upstream changes.

```
git fetch upstream
git merge upstream/master
```

### 3.4.2. Make changes, commit, and push

Same thing as 3.3.1. Make any changes to your files and commit.

```
git add [filename]
``` 
or (with caution)
```
git add --all
```

It will open an editor asking you to briefly describe the changes made. 

Once that's saved, you can then push to your repository on GitHub

```
git push
```

### 3.4.3. Submit a new pull request

Head over to the [pull requests section of theQRL/qips](https://github.com/theQRL/qips/pulls) repo and click on [New pull request].

# 4. Getting help

If you're lost at anypoint during this process, feel free to reach out to the community on [Reddit](https://reddit.com/r/QRL), [Discord](https://www.theqrl.org/discord) or email QRL support at [support@theqrl.org](mailto:support@theqrl.org).