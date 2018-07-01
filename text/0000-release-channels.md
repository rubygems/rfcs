# Bundler release channels

- Feature Name: release\_channels
- Start Date: 2018-06-30
- RFC PR:
- Bundler Issue:

# Summary

Bundler release channels allow users to opt in to the level of release stability that they want. The “canary” channel has a release from every commit that builds green. The default channel has regular releases when features and bugfixes are ready. The “stable” channel only has releases that have fixes for any issues found by the default channel. When possible, bugfixes are backported from the default channel to the stable channel.

# Motivation

As the developers of Bundler, we want to be able to continue to develop and revise Bundler into a better version of itself. As users of Bundler, we want the safest, most stable, least problematic version of Bundler at any given time.

Release channels are a way for the Bundler team to officially bless and support versions that have proven themselves to be stable. It also adds an easy and straightforward way to try the latest upcoming changes to Bundler at any time, increasing the chances that bugs can be found and fixed.

With the stable channel, this RFC aims to actively support the usecases of users deploying Ruby code to important production servers. With the canary channel, this RFC aims to actively support power users, Bundler contributors, early adopters, and anyone who wants to participate in the future of Bundler.

Hopefully, release channels will allow Bundler to change and move forward more easily, while at the same time providing even more stability as an option for users who care most about that.

# Guide-level explanation

To use the default release channel, install the Bundler gem and use it as usual. For example, you install Bundler by running `gem install bundler`, and you install gems by running `bundle install`. The default release of Bundler will let you know anytime there is a newer default release available, but will not install it for you.

To use the stable release channel, instead install with `gem install bundler-stable`. Install gems as usually, by running `bundle install`. The stable version of Bundler will let you know anytime there is a newer stable version available, but will not install it for you. You can identify stable releases of Bundler by running `bundle install --version` and looking for `(stable)` in the output.

To use the canary release channel, instead install with `gem install bundler-canary`. Install gems as usually, by running `bundle install`. The canary version of Bundler will continuously keep itself up to date, installing the latest canary version (if you don’t already have it) during any `bundle install`. You can identify canary versions of Bundler by running `bundle --version`, and checking the output for `(canary)`.

# Reference-level explanation

Default releases are created using today’s release process, running `rake release`. Maybe default releases could be the same as canary releases, but with different feature flags?

Stable releases are created by running a new rake task, `rake release:stable`. Rather than creating a new version from the latest commit, `rake release:stable` takes an existing version of Bundler as an argument, and releases that version to the stable channel. The rake task needs to check out the tag, edit the gemspec to change the name of the gem, and then build and push to RubyGems.org. Minor versions probably should not be promoted to stable until they have had at least one bugfix release or two weeks in the default channel with no significant issues.

Canary releases are created automatically from Travis CI anytime the build passes with the latest released Ruby and RubyGems versions. The canary release task edits the gemspec to change the gem name and append the first 6 characters of the commit sha to the end of the version.

We’ll likely need to update `bundle env` to include channel information.

# Drawbacks

We might not want to do this if we think it would be more development effort than is worth the benefits of having multiple release channels.

This change increases the possible number of Bundler versions in the wild, so it will likely increase the workload for maintainers by at least a small amount.

Adding a stable channel means supporting one older version of Bundler for longer than our current support plans, which is likely small but still some more additional work.

# Rationale

The Bundler team deals with the tension between stability and change on a daily basis. With the tens or hundreds of thousands of users that Bundler has, it feels impossible to satisfy everyone with every release.

Release channels tries to make it possible to satisfy everyone, by dividing users into groups based on risk tolerance, and deliberately aiming for that level of risk. Adding stable releases might make it possible to support default releases for less time overall, to try to balance out the overall support load.

The strategy proposed in this RFC (or something very similar) is currently in active use by the Rust language, the NPM package manager, and the Ember.js framework, among many others. It seems to strike the best balance, allowing ongoing development work while protecting production systems from changes and accidents as much as possible.

If we don’t do this, we will continue to struggle with the existing problem of strongly conflicting expectations and tensions within the user base for each version of Bundler that is released.

# Alternatives

Instead of releasing to canary with every commit, we could release to canary on an automated cadence, like every 24 hours.

Other possible solutions to this class of problem include: the Unix and early Ruby strategy of alternating stable and unstable releases, declaring all releases unstable except one blessed release per year or quarter, or any other option to classify some versions as stable and some as unstable.

# Unresolved questions

The details of how to release, when to release, how feature flags interact with release channels, and how to distinguish channel releases all likely need to be worked out before implementation.

It would be a wonderful next step (albeit out of scope for this RFC) to ship and support one Heroku buildpack for each Bundler release channel, so that Ruby developers on Heroku can easily test each channel, give feedback, and provide bug reports _before_ Bundler releases are rolled out to all Heroku users.

Questions about breaking changes, how long to support versions, support for versions of Ruby or RubyGems, and integration with RubyGems all seem out of scope for this specific RFC.
