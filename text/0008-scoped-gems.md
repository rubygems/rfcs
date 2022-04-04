# RubyGems Scope

* Feature Name: Scoped Gems
* Start Date: 4/1/2022
* RFC PR:
* Bundler Issue:

# Summary

Scoped gems is a feature that allows for grouping related gems together under a specific user or organization account. Scoped gems are identified via their name following a specific pattern, and can be built, installed, and required the same way as existing gems today. Each user account has their own scope, and only that user account is permitted to register a new gem under that scope (first-time publish); Once one version is published, any owner of the gem may publish new versions.

# Motivation

There are three main motivations for supporting scoped gems:

1. Widen the availability of gem names. Many users may want to publish a Configuration gem, but the name configuration has already been reserved. With scoped gems, each user can have their own configuration gem in their own scope.
2. Reduce customer confusion for tools and services spanning multiple gems. It is not uncommon for an organization to distribute many packages, such as Rails with the ActiveX packages, AWS with SDK clients, etc. Scoped gems are a good way to signal official packages for organizations.
3. Help reduce [typo-squatting](https://blog.reversinglabs.com/blog/mining-for-malicious-ruby-gems) with malicious code. Scoped gems are also susceptible to typos, however, gems published under a scope are verified to be owned by that user.

# Guide-level explanation

A scoped gem is any gem that is named following the pattern `gem@scope`, where “gem” is the name of the gem, and “scope” is the user’s scope (username or organization account).

As an example, consider a scoped gem called `hola@mullermp.gemspec`. Note that both the gemspec’s `name` and the required file `lib/hola@mullermp.rb` follows the `gem@scope` pattern:

```
Gem::Specification.new do |s|
  s.name = 'hola@mullermp'
  s.version = '0.0.0'
  s.summary = "Hola!"
  s.authors = ["Matt Muller"]
  s.files = ["lib/hola@mullermp.rb"]
end
```

The scoped gem can be built with `gem build`:

```
$ gem build hola@mullermp.gemspec 
  Successfully built RubyGem
  Name: hola@mullermp
  Version: 0.0.0
  File: hola@mullermp-0.0.0.gem

```

And the scoped gem can be installed via command line:

```
$ gem install hola@mullermp-0.0.0.gem
Successfully installed hola@mullermp-0.0.0
Parsing documentation for hola@mullermp-0.0.0
Done installing documentation for hola@mullermp after 0 seconds
1 gem installed
```

or installed via Bundler:

```
$ cat Gemfile
gem 'hola@mullermp'

$ bundle install
Resolving dependencies...
Using bundler 2.2.22
Using hola@mullermp 0.0.0
Bundle complete! 1 Gemfile dependency, 2 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
```

When installed, developers can expect that the gem `hola` comes from a known source user or organization, `mullermp`, because the [RubyGems.org](http://rubygems.org/) website validated the first-time publish of `hola@mullermp` to be from `mullermp`.

Finally, in Ruby code, the gem can be required:

```
$ irb
irb(main):001:0> require 'hola@mullermp'
=> true
irb(main):002:0> Hola.hi
Hello world from my Scoped gem!
=> nil
```

# Reference-level explanation

There are two types of changes required, one change to the `gem` command tooling and another on the [RubyGems.org](http://rubygems.org/) API.

## `gem` commands

To support the pattern in the `gem` commands, the `VALID_NAME_PATTERN` regex should be updated to include the `@` character.

```
VALID_NAME_PATTERN = /\A[a-zA-Z0-9\@\.\-\_]+\z/.freeze # :nodoc:
```

This change allows for `hola@mullermp.gemspec` to be built, and for `hola@mullermp-0.0.0.gem` to be installed.

## [RubyGems.org](http://rubygems.org/) API

Similar to the `gem` commands, the `NAME_PATTERN` regex in [patterns.rb](https://github.com/rubygems/rubygems.org/blob/84a49869c19303bf3a02c9f7edab67de093c65c1/lib/patterns.rb) needs to be updated:

```
# Adds @ for scope support
ALLOWED_CHARACTERS = "[A-Za-z0-9**@**#{Regexp.escape(SPECIAL_CHARACTERS)}]+".freeze
```

In the [Rubygem model](https://github.com/rubygems/rubygems.org/blob/master/app/models/rubygem.rb), we can make use of the `needs_name_validation?` predicate:

```
validate :ensure_gem_scope, if: :needs_name_validation?
```

and `ensure_gem_scope` might look something like:

```
# Only invoked when needs_name_validation? is true
# Which is on new_record? or name_changed?
def ensure_gem_scope
  gem, scope = name.split("@")
  return unless scope
  if scope != user
    # Raise an error and/or prevent gem push
  end
end
```

# Drawbacks

One main drawback is confusion between normal gems and scoped gems. It is important that we update the [recommendations](https://guides.rubygems.org/name-your-gem/) to explain the scoped gems feature and what Ruby namespaces to expect.

An obvious drawback is, with widened gem name availability, there may be more namespace conflicts. Consider a case where user1 creates `configuration@user1` and user2 creates `configuration@user2`, both creating a `Configuration` class. If user 3 installs and requires both, then there may be side effects. This behavior exists today albeit gem names may be different.

# Rationale and Alternatives

The main benefits of this approach are:

1. It is a frugal change to the RubyGems gem specification that is mostly a naming standard. It removes validation around gem naming on `gem build` and `gem install`, and adds validation for first-time gem publishing via `gem push`.
2. Avoids changes to the Ruby library (and its versions) and how gems are required.
3. Similar to reserved prefixes for verified publishers (like [NuGet](https://github.com/NuGet/Home/wiki/NuGet-Package-Identity-Verification)) but it can be considered as a *reserved suffix*.
4. Possible removal of “blacklisted” gem names as important gem names can be migrated to scopes, i.e. `socket@ruby`.

## Alternative 1 - Reserved Prefixes

An alternative to this approach would be to reserve prefixes, like `aws-sdk-` for AWS SDKs. The major upside is that there’s no change to the gem specification and can be implemented even more cheaply. However, the downsides to this approach are:

1. Prefix registration may involve an application and/or manual review of some kind.
2. Gem name availability is not widened (i.e. configuration gem example), unless you reserve a prefix for your own username.
3. Some organizations do not follow a prefixing pattern (i.e. Rails with Active/ActionX, some AWS products, etc).
4. Enforcing new validation on an existing set of gems may result in conflicts and exceptions that need to be codified. An “open” prefixing reservation system (i.e. each user has their own prefix reserved by default) is certainly a backwards incompatible change.

## Alternative 2 - Scoping with dots (.)

In the current specification (although not documented), dots (`.`) are allowed. There are [a few “hidden” gems](https://rubygems.org/search?query=.) that use dots. If adding the `@` character is not desired, the dot character can be re-used to handle scoping, perhaps with the pattern `scope.gem_name` such as `aws-sdk.s3`.

# Unresolved questions

**How do we migrate existing gems to scoped naming?**
Take for example `aws-sdk-s3`. A new scoped gem name might be `s3@aws-sdk`. `aws-sdk-s3` may need to have an “alias” pointing to `s3@aws-sdk` and be resolved when performing `gem install`, or `s3@aws-sdk` will need to be a new gem (duplicate publish) with an eventual deprecation of `aws-sdk-s3`.

**What about old versions of the `gem` command tooling?**
Some users will be using an older rubygems version and attempt to push or install a scoped gem. What user experience do we need or want?

# Related Links

Namespace gem discussion - https://github.com/rubygems/rfcs/issues/31
Verified users proposal - https://github.com/rubygems/rubygems.org/pull/2698

