# Merge Bundler and RubyGems

- Feature Name: bundler\_rubygems\_merger
- Start Date: 2019-04-23

## Summary

Let's combine Bundler and RubyGems into one thing, with one team, repo, issues, version number, release cycle, and all the other things. When everything is done,  `bundle install` will be a feature provided by RubyGems.

## Motivation

When Bundler was first created, in the late 2000s, it had very different goals than RubyGems had at the time. For several years after Bundler launched, the Bundler and RubyGems teams had different goals, and different ideas about how to reach those goals.

However, over the last ten years Bundler and RubyGems have become more and more overlapping. Today, the Bundler and RubyGems teams are almost all the same people, and many changes to one of Bundler or RubyGems also require changes in the other.

As Bundler and RubyGems continue to overlap more and more, especially now that they both ship inside Ruby itself, it keeps getting more and more annoying to keep them separate.

In order to make future development and releases easier, let's combine Bundler with RubyGems, both organizationally (team membership, GitHub org and issues, git repositories) and technically (combined repo, codebase, version number, releases, and install process).

Combining some things will be easy, although combining everything will take some experimentation, development work, testing, and future releases. It seems worth it.

## Guide-level explanation

** Before the code-level merge:**

Bundler 3 is a stepping stone on the way to combining Bundler and RubyGems. You can install it by running `gem install bundler`. You will notice that there are some changes to how the `bundle` command works.

We've been planning and working towards those changes for a long time, and we think you'll find Bundler easier and more straightforward to use once you've had a chance to get used to the changes.

We've already started working on the next version of Bundler, which will be fully part of RubyGems. It will not need to be installed separately, and will always be available anywhere you have Ruby and RubyGems available.

To work towards a combined Bundler and RubyGems, we are combining the Bundler and RubyGems maintainer teams, and we plan to combine GitHub organizations, security teams, Slacks, and eventually repositories and websites.

We're excited about the possibilities opened up by combining the Bundler and RubyGems code, and looking forward to a codebase that is easier to debug, release, and maintain in the future.

**After the code-level merge:**

As of RubyGems 4, Bundler and RubyGems are now merged! That means that they have the same version number, and upgrading RubyGems is how you update to the latest version of Bundler.

As part of this transition, Bundler 4 (as part of RubyGems 4) will be able to be able to install any existing Bundler project: full backwards compatibility.

To use the new, merged Bundler and RubyGems, install the latest RubyGems by running `gem update --system`. You're done! You now have the latest Bundler and the latest RubyGems.

From time to time, Bundler may introduce opt-in changes that require a certain minimum version of Bundler. In that case, it is possible to choose to upgrade an application to require the latest version of Bundler in order to use those features.

However, Bundler will never force an application (or other users of an application) to upgrade to the latest Bundler version. The minimum required version of Bundler will always be opt-in, per application, and never be systemwide across an entire Ruby.

## Reference-level explanation

Implementing the merger will likely require multiple phases. The organizational merger is easy to start right away, while the code-level merger will require some research and experimentation beforehand.

### Phase 1, organization merger (April 2019–May 2019)
- Move GitHub repos from the `bundler/` org to the `rubygems/` org
  - For example, `bundler/bundler` will become `rubygems/bundler`, etc
  - Open Bundler PRs can be migrated with help from [this script](https://gist.github.com/segiddins/7a846516bd15f6655120db5f91e11e42)
- Add all members of the Bundler maintainers team to the existing RubyGems maintainers team to create a new combined maintainers team
  - Bundler committers and RubyGems committers will not be combined at this time
- Combine maintainer Slack workspaces and Slack channels
- Combine security@, team@, and other emails
- Start to update documentation and website links to issues, pull requests, troubleshooting, developer docs, etc

### Phase 2, research for codebase merger (June 2019–??? 2019)
- Research and test a merge plan that replaces the Bundler submodule in RubyGems with the entire git history and all commits from the Bundler git repo
- Create a CI task that runs run all of the RubyGems and Bundler tests together, covering the current RubyGems and Bundler build matrix (we can delete SO MANY MATRIX ENTRIES 🤩)
- Update the Bundler release scripts and automation to release out of the RubyGems repo, and test it

### Phase 3, codebase merger (est. 2019)
- Use the plan from Phase 2 to merge the Bundler repo into the RubyGems repo in place of the submodule
  - We plan to use https://github.com/jeremysears/scripts/blob/master/bin/git-submodule-rewrite
  - It's ok if this needs a force push, but we want to avoid more than one force push
- Ensure that CI continues to pass with Bundler development moved into the RubyGems repo
- Use GitHub's "Move Issue" beta feature to move active issues from the Bundler repo over to the RubyGems repo
- Combine READMEs, issue reporting guides, and other in-repo docs
- Update the `rubygems/bundler` readme and "new issue" message to point to `rubygems/rubygems` for everything except bug fixes for old Bundler versions

### Phase 4, release merger (est. 2020)
- Combine version numbers, git tags, and changelogs
- Support releasing RubyGems and Bundler as a single unit that can be installed via `gem install --system`
- Update RubyGems' automated releases to ship this single unit
- Remove the separate Bundler release automation
- Finally start easily sharing code between RubyGems and Bundler 😅

### Phase 4, archive Bundler repo (Dec 2020)
- Wait until Bundler 2.x is no longer supported (I think Dec 2020?)
- Disable CI for `rubygems/bundler`
- Update the readme to note that all Bundler development has moved into RubyGems
- Archive the repo on GitHub

## Drawbacks

I think the major downside to doing this is that it will take a bunch of time and effort without producing any clear direct benefits to end users. We could spend this time on other fixes or features, and it would just... be the mildly frustrating status quo, in terms of separate repos and releases.

Doing this also reduces Bundler's ability to release quickly, to release outside of the RubyGems release cycle, and (eventually) will remove Bundler's ability to have more than one version installed at the same time on one Ruby version. We think we have a solution for the "only one Bundler per Ruby" thing, but that's the subject of the next RFC.

## Rationale and Alternatives

The argument for doing this is based primarily on the way that Bundler and RubyGems have effectively become a single project over time, and it would be much more pleasant for everyone involved if the structure of the project matched the way it works and is used.

If we don't do this, Bundler and RubyGems will continue to get more entangled (since they now each depend on parts of the other), but maintenance and releases will be quite annoying because they each have to stay compatible will all possible other versions of each other.

## Unresolved questions

Is anything missing from this plan? Are there any objections to this plan?

We'll need to test as we move through each of these phases, and we'll probably break some things that need to be fixed. Bundler's been a separate thing for a long time!

Out of scope here, but related to this RFC, is the planned implementation of full backwards compatibility for Bundler, allowing it to work with any existing lockfile without forcing other users to upgrade their Bundler version. I'll post that one as soon as I finish writing it.
