# StreamNative RFCs
[StreamNative RFCs]: #sn-rfcs

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the community.

The "RFC" (request for comments) process is intended to provide a consistent
and controlled path for new features to enter the core Apache Pulsar and its
ecosystem projects, so that all stakeholders can be confident about the direction
the Pulsar ecosystem is evolving in.


## Table of Contents
[Table of Contents]: #table-of-contents

  - [Opening](#sn-rfcs)
  - [Table of Contents]
  - [RFC List](./rfcs/README.md)
  - [When you need to follow this process]
  - [Sub-team specific guidelines]
  - [Before creating an RFC]
  - [What the process is]
  - [The RFC life-cycle]
  - [Reviewing RFCs]
  - [Implementing an RFC]
  - [RFC Postponement]
  - [Help this is all too informal!]
  - [Contributions]


## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to
Pulsar and its ecosystem projects, or the RFC process itself. What constitutes a
"substantial" change is evolving based on community norms and varies depending
on what part of the ecosystem you are proposing to change, but may include the
following.

- Any major new feature, subsystem, or piece of functionality
- Any change that impacts the public interfaces of the project

All of the following are public interfaces that people build around:

- Pub/Sub API, including classes related to that
- Functions API, including classes related to that
- I/O API, including classes related to that
- Public API to construct any Pulsar I/O connectors
- Classes marked with the @Public annotation
- On-disk metadata formats
- On-disk binary formats
- Network wire protocols
- User-facing scripts/command-line tools
- Configuration settings
- Exposed monitoring information

Not all compatibility commitments are the same. We need to spend significantly
more time on public APIs as these can break code for users. They cause people
to rebuild code and lead to compatibility issues in large multi-dependency
projects (which end up requiring multiple incompatible versions). Configuration,
monitoring, and command line tools can be faster and looser — changes here will
break monitoring dashboards and require a bit of care during upgrades but aren't
a huge burden.

For the most part monitoring, command line tool changes, and configs are added
with new features so these can be done with a single RFC.

If you submit a pull request to implement a new feature without going through
the RFC process, it may be closed with a polite request to submit an RFC first.

## Before creating an RFC
[Before creating an RFC]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the RFC can
make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is
generally a good idea to pursue feedback from other project developers
beforehand, to ascertain that the RFC may be desirable; having a consistent
impact on the project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include talking
the idea over on the Apache Pulsar slack channel, and filing issues on this repo
for discussion.

As a rule of thumb, receiving encouraging feedback from long-standing project
developers, and particularly members of the relevant projects is a good
indication that the RFC is worth pursuing.


## What the process is
[What the process is]: #what-the-process-is

In short, to get a major feature added to the projects maintained by StreamNative,
one must first get the RFC merged into the RFC repository as a markdown file.
At that point the RFC is "active" and may be implemented with the goal of eventual
inclusion into Pulsar ecosystem.

  - Fork the RFC repo [RFC repository]
  - Copy [`rfc/0000-template.md`](./0000-template.md) to
    `rfc/rfcs/<RFC-number>-my-feature.md` (where "my-feature" is
    descriptive). Assign the RFC number using the `Next RFC number` in 
    [RFC Index](./rfcs/README.md).
  - Fill in the RFC. Put care into the details: RFCs that do not present
    convincing motivation, demonstrate lack of understanding of the design's
    impact, or are disingenuous about the drawbacks or alternatives tend to
    be poorly-received.
  - Add the RFC to [RFC Index](./rfcs/README.md) and bump the `Next RFC number`.
  - Submit a pull request. As a pull request the RFC will receive design
    feedback from the larger community, and the author should be prepared to
    revise it in response.
  - Build consensus and integrate feedback. RFCs that have broad support are
    much more likely to make progress than those that don't receive any
    comments. Feel free to reach out to the RFC assignee in particular to get
    help identifying stakeholders and obstacles.
  - The community will discuss the RFC pull request, as much as possible in the
    comment thread of the pull request itself. Offline discussion will be
    summarized on the pull request comment thread.
  - RFCs rarely go through this process unchanged, especially as alternatives
    and drawbacks are shown. You can make edits, big and small, to the RFC to
    clarify or change the design, but make changes as new commits to the pull
    request, and leave a comment on the pull request explaining your changes.
    Specifically, do not squash or rebase commits after they are visible on the
    pull request.
  - The RFC is approved and merged if at least two people with write access
    approve the change.

## The RFC life-cycle
[The RFC life-cycle]: #the-rfc-life-cycle

Once an RFC becomes "active" then authors may implement it and submit the
feature as a pull request to the corresponding project repo. Being "active"
is not a rubber stamp, and in particular still does not mean the feature
will ultimately be merged; it does mean that in principle all the major
stakeholders have agreed to the feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a Pulsar developer has been assigned the task of
implementing the feature. While it is not *necessary* that the author of the
RFC also write the implementation, it is by far the most effective way to see
an RFC through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We
strive to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect every
merged RFC to actually reflect what the end result will be at the time of the
next major release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new RFCs, with a note added to the original RFC.


## Reviewing RFCs
[Reviewing RFCs]: #reviewing-rfcs

While the RFC pull request is up, the community may schedule meetings with the
author and/or relevant stakeholders to discuss the issues in greater detail,
and in some cases the topic may be discussed at a SIG meeting. In either
case a summary from the meeting will be posted back to the RFC pull request.

A SIG makes final decisions about RFCs after the benefits and drawbacks
are well understood. These decisions can be made at any time, but the SIG 
will regularly issue decisions. When a decision is made, the RFC pull request
will either be merged or closed. In either case, if the reasoning is not clear
from the discussion in thread, the SIG will add a comment describing the
rationale for the decision.


## Implementing an RFC
[Implementing an RFC]: #implementing-an-rfc

Some accepted RFCs represent vital features that need to be implemented right
away. Other accepted RFCs can represent features that can wait until some
arbitrary developer feels like doing the work. Every accepted RFC has an
associated issue tracking its implementation in the corresponding project
repository; thus that associated issue can be assigned a priority via the
triage process that the team uses for all issues in its repository.

The author of an RFC is not obligated to implement it. Of course, the RFC
author (like any other developer) is welcome to post an implementation for
review after the RFC has been accepted.

If you are interested in working on the implementation for an "active" RFC,
but cannot determine if someone else is already working on it, feel free to
ask (e.g. by leaving a comment on the associated issue).


## RFC Postponement
[RFC Postponement]: #rfc-postponement

Some RFC pull requests are tagged with the "postponed" label when they are
closed (as part of the rejection process). An RFC closed with "postponed" is
marked as such because we want neither to think about evaluating the proposal
nor about implementing the described feature until some time in the future, and
we believe that we can afford to wait until then to do so. Historically,
"postponed" was used to postpone features until after 1.0. Postponed pull
requests may be re-opened when the time is right. We don't have any formal
process for that, you should ask members of the relevant sub-team.

Usually an RFC pull request marked as "postponed" has already passed an
informal first round of evaluation, namely the round of "do we think we would
ever possibly consider making this change, as outlined in the RFC pull request,
or some semi-obvious variation of it." (When the answer to the latter question
is "no", then the appropriate response is to close the RFC, not postpone it.)


### Help this is all too informal!
[Help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the present
circumstances. As usual, we are trying to let the process be driven by
consensus and community norms, not impose more structure than necessary.


### Contributions
[Contributions]: #contributions

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be licensed without any additional terms or conditions.


[RFC repository]: https://github.com/streamnative/community

## Acknowledgements

Thank you to the [Rust RFC](https://github.com/rust-lang/rfcs) pages for providing us with inspirations.
