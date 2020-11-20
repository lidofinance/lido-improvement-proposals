---
lip: 0
title: LIP Purpose and Guidelines
status: WIP
author: Lidos Community
discussions-to: https://forum.lido.com/t/lip-0-what-is-an-lip/13
created: 2020-07-22
updated: 2020-09-03
---

## What is an LIP?

LIP stands for Lidos Improvement Proposal, it has been adapted from the YIP (Yearn Improvement Proposal). The purpose of this process is to ensure changes to Lido are transparent and well governed. An LIP is a design document providing information to the Lido community about a proposed change to the system. The author is responsible for building consensus within the community and documenting dissenting opinions.

## LIP Rationale
We intend LIPs to be the primary mechanisms for proposing new features, collecting community input on an issue, and for documenting the design decisions for changes to Lido. Because they are maintained as text files in a versioned repository, their revision history is the historical record of the feature proposal.

It is highly recommended that a single LIP contain a single key proposal or new idea. The more focused the LIP, the more successful it is likely to be.

An LIP must meet certain minimum criteria. It must be a clear and complete description of the proposed enhancement. The enhancement must represent a net improvement.

## LIP Work Flow

:warning: Before you begin, vet your idea, this will save you time. Ask the Lido community first if an idea is original to avoid wasting time on something that will be rejected based on prior research (searching the Internet does not always do the trick). It also helps to make sure the idea is applicable to the entire community and not just the author. Just because an idea sounds good to the author does not mean it will have the intend effect.

Your role as the champion is to write the LIP using the style and format described below, shepherd the discussions in the appropriate forums, and build community consensus around the idea. Following is the process that a successful LIP will move along:

```
Proposed -> Approved -> Implemented
  ^                     |
  +----> Rejected       +----> Moribund
  |
  +----> Withdrawn
  v
Deferred
```

Each status change is requested by the LIP author and reviewed by the LIP editors. Use a pull request to update the status. Please include a link to where people should continue discussing your LIP. The LIP editors will process these requests as per the conditions below.

* **Work in progress (WIP)** -- Once the champion has asked the Lido community whether an idea has any chance of support, they will write a draft LIP as a [pull request]. Consider including an implementation if this will aid people in studying the LIP.
* **Proposed** If agreeable, LIP editor will assign the LIP a number (generally the issue or PR number related to the LIP) and merge your pull request. The LIP editor will not unreasonably deny an LIP. Proposed LIPs will be discussed on governance calls and in Discord. If there is a reasonable level of consensus around the change on the governance call the change will be moved to approved. If the change is contentious a vote of token holders may be held to resolve the issue or approval may be delayed until consensus is reached.
* **Approved** -- This LIP has passed community governance and is now being prioritised for development.
* **Implemented** -- This LIP has been implemented and deployed to mainnet.
* **Rejected** -- This LIP has failed to reach community consensus.
* **Withdrawn** -- This LIP has has been withdrawn by the author(s).
* **Deferred** -- This LIP is pending another LIP/some other change that should be bundled with it together.
* **Moribund** -- This LIP has been implemented and is now obsolete and requires no explicit replacement.

## What belongs in a successful LIP?

Each LIP should have the following parts:

- Preamble - RFC 822 style headers containing metadata about the LIP, including the LIP number, a short descriptive title (limited to a maximum of 44 characters), and the author details.
- Simple Summary - “If you can’t explain it simply, you don’t understand it well enough.” Provide a simplified and layman-accessible explanation of the LIP.
- Abstract - a short (~200 word) description of the technical issue being addressed.
- Motivation (*optional) - The motivation is critical for LIPs that want to change Lido. It should clearly explain why the existing specification is inadequate to address the problem that the LIP solves. LIP submissions without sufficient motivation may be rejected outright.
- Specification - The technical specification should describe the syntax and semantics of any new feature.
- Rationale - The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.
- Test Cases - Test cases may be added during the implementation phase but are required before implementation.
- Copyright Waiver - All LIPs must be in the public domain. See the bottom of this LIP for an example copyright waiver.

## LIP Formats and Templates

LIPs should be written in [markdown] format.
Image files should be included in a subdirectory of the `assets` folder for that LIP as follows: `assets/Lip-X` (for Lip **X**). When linking to an image in the LIP, use relative links such as `../assets/Lip-X/image.png`.

## LIP Header Preamble

Each LIP must begin with an [RFC 822](https://www.ietf.org/rfc/rfc822.txt) style header preamble, preceded and followed by three hyphens (`---`). This header is also termed ["front matter" by Jekyll](https://jekyllrb.com/docs/front-matter/). The headers must appear in the following order. Headers marked with "*" are optional and are described below. All other headers are required.

` Lip:` <LIP number> (this is determined by the LIP editor)

` title:` <LIP title>

` author:` <a list of the author's or authors' name(s) and/or username(s), or name(s) and email(s). Details are below.>

` * discussions-to:` \<a url pointing to the official discussion thread at forum.lido.fi \>

` status:` < WIP | PROPOSED | APPROVED | IMPLEMENTED >

` created:` <date created on>

` * updated:` <comma separated list of dates>

` * requires:` <LIP number(s)>

` * resolution:` \<a url pointing to the resolution of this LIP\>

Headers that permit lists must separate elements with commas.

Headers requiring dates will always do so in the format of ISO 8601 (yyyy-mm-dd).


#### `discussions-to` header

While an LIP is in WIP or Proposed status, a `discussions-to` header will indicate the URL at [forum.lido.fi/](https://forum.lido.fi/) where the LIP is being discussed.

#### `created` header

The `created` header records the date that the LIP was assigned a number. Both headers should be in yyyy-mm-dd format, e.g. 2001-08-14.

#### `updated` header

The `updated` header records the date(s) when the LIP was updated with "substantial" changes. This header is only valid for LIPs of Draft and Active status.

#### `requires` header

LIPs may have a `requires` header, indicating the LIP numbers that this LIP depends on.

## Auxiliary Files

LIPs may include auxiliary files such as diagrams. Such files must be named LIP-XXXX-Y.ext, where “XXXX” is the LIP number, “Y” is a serial number (starting at 1), and “ext” is replaced by the actual file extension (e.g. “png”).

## LIP Editor Responsibilities

For each new LIP that comes in, an editor does the following:

- Read the LIP to check if it is ready: sound and complete. The ideas must make technical sense, even if they don't seem likely to get to final status.
- The title should accurately describe the content.
- Check the LIP for language (spelling, grammar, sentence structure, etc.), markup (Github flavored Markdown), code style

If the LIP isn't ready, the editor will send it back to the author for revision, with specific instructions.

Once the LIP is ready for the repository, the LIP editor will:

- Assign an LIP number (generally the PR number or, if preferred by the author, the Issue # if there was discussion in the Issues section of this repository about this LIP)

- Merge the corresponding pull request

- Send a message back to the LIP author with the next step.

The LIP editors monitor LIP changes, and correct any structure, grammar, spelling, or markup mistakes we see.

The editors don't pass judgment on LIPs. We merely do the administrative & editorial part.

## History

The LIP document was derived heavily from the YIP Yearn Improvement Proposal document in many places text was simply copied and modified. Any comments about the LIP document should be directed to the LIP editors.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
