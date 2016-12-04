# RubyGems + Bundler RFCs

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

However, some changes are more substantial, and we ask that these be put
through a bit of a design process and produce a consensus in the community and
among the [maintainers].

The "RFC" (request for comments) process is intended to provide a consistent
and predictable path for new features and projects.

[Active RFC List](https://github.com/bundler/rfcs/pulls)

## When you need to follow this process

You need to follow this process if you intend to make "substantial" changes to
RubyGems, Bundler, or their documentation. What constitutes a "substantial"
change is evolving based on community norms, but may include the following.

   - A new feature that creates new API, command, or option surface area.
   - The removal of features that have already shipped in a minor release.
   - The introduction of new idiomatic usage or conventions, even if they
     do not include code changes.

Some changes do not require an RFC:

  - Rephrasing, reorganizing, refactoring, or otherwise "changing shape does
    not change meaning".
  - Additions that strictly improve objective, numerical quality criteria
    (warning removal, speedup, better platform coverage, more parallelism, trap
    more errors, etc.)
  - Addition or removal of warnings.
  - Additions only likely to be noticed by other implementors, invisible to end
    users.

If you submit a pull request to implement a new feature without going through
the RFC process, it may be closed with a polite request to submit an RFC first.

## Before creating an RFC

It's often helpful to get feedback on your concept before diving into the level
of API design detail required for an RFC. **You may open an issue on this repo
to start a high-level discussion**, with the goal of eventually formulating an
RFC pull request with the specific implementation design.

## What the process is

In short, any major feature needs an RFC that has been merged into this RFC
repo as a markdown file. At that point the RFC is 'active' and may be
implemented with the goal of eventual inclusion into Bundler and/or RubyGems.

* Fork [the RFC repo](http://github.com/bundler/rfcs) (that's this one!)
* Copy `0000-template.md` to `text/0000-my-feature.md` (Where 'my-feature' is
  descriptive—don't assign an RFC number yet.)
* Fill in the RFC. Put care into the details: RFCs should include convincing
  motivation, demonstrate understanding of the impact of the design, and be
  clear about drawbacks or alternatives.
* Submit a pull request. As a pull request the RFC will receive design feedback
  from the larger community, and the author should be prepared to revise it in
  response.
* Build consensus and integrate feedback. RFCs that have broad support are much
  more likely to make progress than those that don't receive any comments.
* RFCs rarely go through this process unchanged, especially as alternatives and 
  drawbacks are shown. You can make edits, big and small, to the RFC to clarify 
  or change the design, but make changes as new commits to the PR, and leave a 
  comment on the PR explaining your changes. Specifically, do not squash or 
  rebase commits after they are visible on the PR.
* RFCs that are candidates for inclusion will enter a "final comment period"
  lasting 7 days. The beginning of this period will be signaled with a comment
  and tag on the RFC's pull request. At that point, the [Bundler Twitter
  account](https://twitter.com/bundlerio) will post a tweet about the RFC to
  attract the community's attention.
* An RFC can be modified based upon feedback from the [maintainers] and
  community. Significant modifications may trigger a new final comment period.
* An RFC may be rejected by the [maintainers] after public discussion has
  settled and comments have been made summarizing the rationale for rejection.
  A [maintainer] should then close the RFC's associated pull request.
* An RFC may be accepted at the close of its final comment period. A
  [maintainer] will merge the RFC's associated pull request, at which point the
  RFC will become 'active'.

## The RFC life-cycle

Once an RFC becomes active then authors may implement it and submit the feature
as a pull request to the Rust repo. Being 'active' is not a rubber stamp, and
in particular still does not mean the feature will ultimately be merged; it
does mean that in principle all the major stakeholders have agreed to the
feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is 'active'
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a Rust developer has been assigned the task of
implementing the feature. While it is not necessary that the author of the RFC
also write the implementation, it is by far the most effective way to see an
RFC through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to active RFC's can be done in follow-up PR's. We strive to write
each RFC in a manner that it will reflect the final design of the feature; but
the nature of the process means that we cannot expect every merged RFC to
actually reflect what the end result will be at the time of the next major
release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new RFCs, with a note added to the original RFC.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the RFC
author (like any other developer) is welcome to post an implementation for
review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active' RFC, but
cannot determine if someone else is already working on it, feel free to ask
(e.g. by leaving a comment on the associated issue).

## Reviewing RFC's

The [maintainers] will review open RFC pull requests, giving feedback and
eventually accepting or rejecting each one. Every accepted feature should have
a maintainer champion, who will represent the feature and its progress.

#### Inspiration

The RubyGems + Bundler RFC process is inspired by the [Rust], [Ember.js], and [Swift] RFC processes.

[Rust]: https://github.com/rust-lang/rfcs
[Swift]: https://github.com/apple/swift-evolution
[Ember.js]: https://github.com/emberjs/rfcs
[maintainer]: http://bundler.io/contributors.html
[maintainers]: http://bundler.io/contributors.html
