- Feature Name: groups\_command
- Start Date: 2017-08-05
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

This is a proposal to deprecate the `with` and `without` settings, replacing them with a `bundle groups` command to list groups and manage whether each group is enabled or disabled for future runs of `bundle install`.

# Motivation

Enabling and disabling groups has been a consistent thorn in the side of Bundler’s UX ever since we first shipped 1.0. We only supported `--without` for more than five years. After more than five years of relentless user requests, we finally caved and agreed to add `group: :foo, optional: true` and the `--with` option.

In Bundler 2.0, the `--with` and `--without` options will (at a minimum) no longer be remembered, and they may be removed entirely. Instead, we currently tell users to instead run `bundle config with` and `bundle config without`, respectively.

In one stroke, adding a `bundle groups` command would replace both flags, both config commands, and open up the possibility of a _not_ remembered `install --only` to satisfy [the requests for `--only`]. It also provides users with an instant readout of what they will actually get when they run `bundle install` without having to do mental gymnastics combining the output of `config`. (Or even worse, give up and try it out, and then get surprised by the results.)

# Guide-level explanation

### Migration

For users coming from Bundler 1, the `groups` command replaces the old options `--with` and `--without`, which allowed you to enable or disable groups on the fly while running `install`. In Bundler 1.99, the `--with` and `--without` options will print a deprecation warning when used, directing users to switch to using the `groups` command instead.

### New `groups` command

The `bundle groups` command allows you to list groups of gems declared in the Gemfile, and find out whether they will be installed or not. It also lets you enable or disable groups for future installs.

Run the command `bundle groups` to list the groups defined in your Gemfile and find out whether they are enabled or disabled. Enabled groups will be installed when you run `bundle install`, and disabled groups will not be installed. Groups will be enabled by default, but they can be disabled by default by setting `optional: true` inside the Gemfile.

Let’s say we have this example Gemfile:

'' $ cat Gemfile
'' gem "rails"
'' group :production, optional: true do
''   gem "sprockets-redirect"
'' end
'' group :test do
''   gem "webmock"
'' end

In this Gemfile, the `rails` gem is in the default group, the gem `sprockets-redirect` is in the production group (which is disabled by default), and the gem `webmock` is in the test group. If you were to run `bundle groups`, this is what you would see:

'' $ bundle groups
''   - production   disabled  (default disabled)
''   - test         enabled   (default enabled)

Turning groups on is done via `bundle groups enable` and `bundle groups disable`. Here’s what it would look like to turn off the test group:

'' $ bundle groups enable test
'' Group `test` is now enabled

After disabling the test group, running `bundle install` will install the gem `rails`, but will not install the gems `sprockets-redirect` or `webmock`.

The `bundle groups` command exists to make the process of creating and installing groups understandable and predictable—you’ll always know what groups are going to be installed, and why those groups are enabled or disabled.

### New `--only` option

There are two ways to choose which groups will be installed. The first option simply uses the `groups enable` and `groups disable` commands to set your groups to be enabled and disabled as you want them to be.

Alternatively, you can use the `--only` option (provided on both the `install` and `deploy` commands) to install only the groups whose names you give at that time. For example, if you were to run `bundle install --only production`, the `test` group would not be installed even though it is enabled by default, and the `production` group would be installed even though it is disabled by default.

# Reference-level explanation

The `groups` command can be backed by the same Bundler settings as always, with keys of WITH and WITHOUT. It could alternately be backed by separate settings keys for any group set to a different status than the default status declared in the Gemfile. Either way, I think it amounts to an implementation detail that users won’t care about.

Switching to the `groups` command completely removes ambiguity about whether a group should be enabled or disabled… a group should be enabled if it was passed as an argument to `groups enable`, and disabled when it’s passed as an argument to `groups disable`.

I also propose that this command would allow us to deprecate and remove the `install_if` feature. We could extend the `groups` command guide to explain how to use optional groups and a build script that runs a check and executes `bundle groups enable foo` before running `bundle install`. That removes one (maybe the only?) hard blocker for non-executable Gemfiles. /cc @segiddins

# Drawbacks

This is definitely a backwards-breaking change. We could keep the existing plan of having users set enabled and disabled groups via `bundle config with foo` and `bundle config without foo`.

I think the downsides to this approach in general are that it will take a decent amount of work in the form of implementation, docs, and training users.

# Rationale and Alternatives

I think this approach is the clearest, least surprising, most easily explainable, and most flexible way of handling this user need. The `--with` option is basically a hack directly on top of the `--without` system that didn’t fully meet user needs, and the proposed `--only` option is an additional hack on top of those layered hacks trying to meet further user needs.

We’ve considered other options throughout the history of Bundler, but those options have almost entirely boiled down to “keep the options we have that aren’t flexible enough” or “accept additional options that dramatically increase complexity”.

The impact of not doing this is that we would be maintaining the status quo, which is… not great, for `with` and `without` groups.

# Unresolved questions

How should we store which groups are enabled and disabled? Should we keep the existing settings keys or use new keys?

Should we include the `--only` option in the final implementation of this RFC, or skip it?

