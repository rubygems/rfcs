- Feature Name: add\_funding\_uri
- Start Date: 2020-01-20
- RFC PR:
- Bundler Issue:

# Summary

Add a new `bundle fund` feature (Refer: [rfcs#22](https://github.com/rubygems/rfcs/issues/22)). The `bundle fund` command will list out all the URLs for gems whose maintainers are actively looking for funding.

The purpose of this RFC is to get feedback on use cases, CLI format/syntax, and feature options.

# Motivation

The use case is for open source maintainers to hopefully garner monetary support for their work.

Open source is free as in beer, but there are very real mental, labor, and time costs that every open source maintainer pays for. For every hour spent on a gem, that's one hour less time with their family, their friends, their own free time, and yes--an hour of revenue lost somewhere else.

[GitHub Sponsors](https://github.com/sponsors/), [Tidlelift](https://tidelift.com/), and [Open Collective](https://opencollective.com/) are just some of the ways open source developers can receive funding. But while people can sponsor projects on these sites, discoverability of which projects need help remains a problem.

A `bundle fund` command would be an _optional_ and _unobtrusive_ way for users to identify which projects need funding. It is designed to be a message available right when a user install a gem; similarly, it could be parsed and presented on RubyGems.org for additional visibility.

For more analysis on the benefits of such a change, see [the Node.js community's recent adoption of the `npm fund` command](https://blog.opencollective.com/beyond-post-install/).

# Guide-level explanation

The `bundle fund` command is powered by a `funding` argument in your Gemspec:

```ruby
Gem::Specification.new do |gem|
  gem.name = "#{GEM_NAME}"
  gem.homepage = "#{GEM_HOMEPAGE}"
  gem.funding = "#{GEM_FUNDING_PAGE}"
end
```

`funding` must be a valid URI, much like `homepage`.

After a user runs `bundle update` or `bundle install` on the command line, a message pops up with some additional information:

```
Updating files in vendor/cache
  * byebug-11.1.0.gem
  * ffi-1.12.1.gem
  * mercenary-0.4.0.gem
  * unicode-display_width-1.6.1.gem
Removing outdated .gem files from vendor/cache
  * unicode-display_width-1.6.0.gem
  * byebug-11.0.1.gem
  * mercenary-0.3.6.gem
  * ffi-1.11.3.gem
Bundle updated!
```

This RFC proposes printing some more information:

```
Updating files in vendor/cache
  * byebug-11.1.0.gem
  * ffi-1.12.1.gem
  * mercenary-0.4.0.gem
  * unicode-display_width-1.6.1.gem
Removing outdated .gem files from vendor/cache
  * unicode-display_width-1.6.0.gem
  * byebug-11.0.1.gem
  * mercenary-0.3.6.gem
  * ffi-1.11.3.gem
Bundle updated!

4 gems are looking for funding
  Run `bundle fund` for details
```

This information is parsed out of each gem's `funding` link. The user now has a choice: move on with their day, or run `bundle fund`. If they choose the latter option, the information is printed out:

```
$ bundle fund
  * $GEM_NAME ($GEM_VERSION)
	Funding: <$FUNDING_URL>
```

As well, `bundle info GEM` should show this information as part of its printout:

```
$ bundle info GEM

  * GEM (GEM_VERSION)
	Homepage: GEM_HOMEPAGE
	Funding:  GEM_FUNDING_PAGE
```

If there is no `funding` attribute, no information will be displayed. If none of the gems have a `funding` like, the command should print a message out indicated that:

```
$ bundle fund
No gems are looking for funding!
```

This is a backwards-compatible, additive change to bundler and the gem specification.

# Reference-level explanation

Right now, there is a very similar "print a message" gem feature by way of `post_install_message`. A crucial difference between `post_install_message` and `funding` is that `post_install_message` ought to be preserved for essential information. Sometimes, open source maintainers abuse this by printing their own non-essential messages, to the annoyance of users. To this end, bundler added an `ignore_messages` configuration option, which would silence `post_install_message`s on an individual gem or global level. I do not believe that displaying a single message indicating gems which need `funding` reaches this level of annoyance. Therefore, I do not expect `ignore_messages` to have an impact on `bundle fund`, or the `X gems are looking for funding` message printout, nor should an opt-out flag or configuration should be considered at this time.

`bundle init` should not add a placeholder for `funding`.

The inclusion of a group (eg. `bundle update --group test`) should only show funding information for that group.

Pretty much all of the foundation of representing the "looking for funding" message is already available. I don't expect there to be any clash with any other bundler feature or gem spec behavior.

# Drawbacks

A possibility is that many gems will decide to include a `funding` link in their specification. Keeping track of this data may result in a performance issue or an issue storing all of these URLs in an array. Most likely, this runs the risk of rendering the feature useless. Imagine a scenario where `bundle install` prints this information out:

```
75 gems are looking for funding
  Run `bundle fund` for details
```

Will someone sit down and look at 75 different URLs? Probably not. This is partly the reason why I am not currently keen to include "looking for funding" information for a gem's dependencies, as that number may bloat the results to an unreasonable figure.

# Rationale and Alternatives

The argument for doing this is based primarily on the way our friends [at npm](https://github.com/npm/rfcs/pull/54) came about implementing their RFC. This idea is not perfect, but any step towards increasing the visiblity of open source maintainers is good progress.

Whether we do this or not, the responsibility for funding open source projects continues to rest on the large corporations which make use of these projects. All RubyGems and Bundler can do is increase visibility for that need as best as they are able.

An alternative to including `bundle fund` is to simply have a `funding` gem specification, but only visualize it on RubyGems.org. While I agree this is a good first step, gem users are unlikely to return to a gem's page, once they've finished finding the gem they need. At least a "funding is needed" plea might remind them more frequently to investigate on their own what can be done to help.

# Unresolved questions

Should `bundle fund` show information from dependencies? That is, imagine if gem A relies on gem B, a project's gemspec only defines gem A, yet `funding` information is only provided on gem B. Does the `bundle install` message show that gem B needs funding, even though the user is unaware of what gem B even is? I believe for this first iteration, no. Let's not let perfect be the enemy of good. As adoption increases, we can explore whether this information should be:

* included by default
* included with a flag/config
* included by default and excludable via a flag/config (the most likely and "fair" option)
