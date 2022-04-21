# RubyGems Scoped Gems and Organizations

* Feature Name: Scoped Gems
* Start Date: 4/1/2022
* RFC PR:
* Bundler Issue:

# Summary

 Scoped gems is a feature that allows for grouping related gems together under an organization account. Scoped gems are identified by their name following a specific pattern of `gem@scope`. They can be built, installed, and required the same way as existing gems today. An Organization is a group of users that can own 0 or many gem scopes and 0 or many gems. Only the users that are part of an organization are permitted to publish a new gem under a scope that is registered to that organization (first-time publish).

# Motivation

There are three main motivations for supporting scoped gems and organizations:

1. Reduce customer confusion for tools and services spanning multiple gems. It is not uncommon for an organization to distribute many packages, such as Rails with the ActiveX packages, AWS with SDK clients, Ruby standard library gems, Azure SDKs, etc. Scoped gems are a good way to signal official packages from organizations.
2. Help reduce [typo-squatting](https://blog.reversinglabs.com/blog/mining-for-malicious-ruby-gems) with malicious code. Scoped gems are also susceptible to typos, however, gems published under a scope are verified to be owned by that organization.
3. An organization allows for a better gem owner permissions model. Gems owned by the same company or organization currently need to manage users for each gem. Users and gem owners can now be part of a single entity, and that entity can be the owner of the gem.

# Guide-level explanation

## Scoped gems

A scoped gem is any gem that is named following the pattern `gem@scope`, where “gem” is the name of the gem, and “scope” is an organization's reserved gem scope. Gem scopes are globally unique across all organizations.

As an example, consider a scoped gem defined as `hola@mullermp.gemspec`. Note that both the gemspec’s `name` and the required file `lib/hola@mullermp.rb` follows the `gem@scope` pattern:

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

When installed, developers can expect that the gem `hola` comes from a known organization that owns the `mullermp` gem scope, because the [RubyGems.org](http://rubygems.org/) website validated the first-time publish of `hola@mullermp` to be from a user in that organization.

Finally, in Ruby code, the gem can be required:

```
$ irb
irb(main):001:0> require 'hola@mullermp'
=> true
irb(main):002:0> Hola.hi
Hello world from my Scoped gem!
=> nil
```

The user can *reasonably* assume that the `hola@mullermp` gem has a namespace of `Hola`, following the [recommended naming guidelines](https://guides.rubygems.org/name-your-gem/).

## Organizations

An organization is a group of users that can own 0 or many gems and 0 or many gem scopes. A user can visit the `mullermp` organization by visiting the following link: `https://rubygems.org/organizations/mullermp` where `mullermp` can be any organization name. On this page, a user would see a "profile" that lists the organization's members (a user named `mullermp`), the gems owned by the organization (one gem, `hola@mullermp`), and any reserved gem scopes (`mullermp` scope). If the user is part of the organization, they can register additional gem scopes, with limitations.

The organization's members are effectively the gem owners and maintainers. On a gem's page, such as `https://rubygems.org/gems/hola@mullermp`, a new section labeled as "Organization" is displayed, with a link to the `mullermp` organization page. The organization's members, are "splat" onto the gem's page like normal gem owners.

# Reference-level explanation

## [RubyGems.org](http://rubygems.org/) Organization MVC

A standard Rails scaffold is needed for an organization entity.

1) Controller - An organizations controller is needed for administration, such as: listing/adding/removing owners, listing gems owned by the organization, and reserving gem scopes.

2) Model - The Organization model will need the following fields/references: a list of owners that can perform administration (user references), a list of reserved scopes that the organization owns (string list or references), and a list of gems owned by the organization (rubygem references).

3) View - The Organization view will be routed under `/organizations/<org name>`. The page will have forms for adding/removing of owners and reserving gem scopes, as well as displaying any textual information such as the organization's mission statement, a profile picture, etc.

## `gem` commands

To support the pattern in the `gem` commands, the [`VALID_NAME_PATTERN` regex](https://github.com/rubygems/rubygems/blob/504272e35d35a5c7dcdccfece574f5d8a060b148/lib/rubygems/specification.rb#L109) should be updated to include the `@` character (in both the specification and the policy).

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

In the [Rubygem model](https://github.com/rubygems/rubygems.org/blob/1cb71cb167076d08fb3a81209a72af04d37da16f/app/models/rubygem.rb), we can make use of the `needs_name_validation?` predicate:

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
  # find the owning organization
  org = Organization.find_by_scope(scope)

  unless org.owners.include?(current_user)
    # Raise an error and/or prevent gem push
  end
end
```

## Migrating existing gems to an organization (scoped gems)

Migration from an existing gem to a scoped gem cannot be done by RubyGems.org alone. The preferred migration strategy will be to convert un-scoped gems into a shim that forwards to a scoped gem using a gemspec dependency and a require statement.

As an example, consider the un-scoped gem `hola`. This gem needs to be transferred to the `mullermp` namespace. The steps are as follows:

1. Build and publish any version of `hola@mullermp` on RubyGems.org just like any other gem.

2. Modify `hola.gemspec` to have one single dependency on `hola@mullermp`:
  ```
  s.add_runtime_dependency 'hola@mullermp' # open ended, but can use major version constraint
  ```

3. Modify `lib/hola.rb` to have a single require statement: `require 'hola@mullermp'`. Optionally modify the gemspec's `post_install_message` to warn the user about forwarding.

4. Build and publish `hola`. No new versions to `hola` should be published.

Legacy applications that install `hola` will also download and install `hola@mullermp`. In code, `hola` can still be required and used as normal. Attempting to require `hola@mullermp` will return false as it is already loaded.

```
irb(main):001:0> require 'hola'
=> true
irb(main):002:0> Hola.hi
Hello world!
=> nil
irb(main):003:0> require 'hola@mullermp'
=> false
```

# Drawbacks

One main drawback is confusion between normal gems and scoped gems. It is important that we update the [recommendations](https://guides.rubygems.org/name-your-gem/) to explain the scoped gems feature and what Ruby namespaces to expect. RubyGems.org does not enforce Ruby namespace with gem naming conventions, nor should it. The `hola@mullermp` gem is expected to have a `Hola` namespace, just as `net-http-digest_auth@ruby` (a potential Ruby official gem) would have `Net::HTTP::DigestAuth`.

An obvious drawback is, with widened gem name availability, there may be more namespace conflicts. Consider a case where org1 creates `configuration@org1` and org2 creates `configuration@org2`, both creating a `Configuration` class. If a user installs and requires both, then there may be side effects. This behavior exists today albeit gem names may be different. Developers will need to continue using discretion.

# Rationale and Alternatives

The main benefits of this approach are:

1. It is a frugal change to the RubyGems gem specification that is mostly a naming standard. It removes validation around gem naming on `gem build` and `gem install`, and adds validation for first-time gem publishing via `gem push`.
2. Avoids changes to the Ruby library (and its versions) and how gems are required.
3. Similar to reserved prefixes for verified publishers (like [NuGet](https://github.com/NuGet/Home/wiki/NuGet-Package-Identity-Verification)). This approach can be considered as a *reserved suffix*.
4. Eventual removal of the [reserved gem list](https://github.com/rubygems/rubygems.org/blob/fc7cc3a9879d69adb48e2d37282aea34d7f311b0/lib/patterns.rb) as important standard Ruby gem names are migrated to scopes (i.e. `socket@ruby`, no need to reserve `socket`).

## Alternative 1 - Reserved Prefixes

An alternative to this approach would be to reserve prefixes, like `aws-sdk-` for AWS SDKs. The major upside is that there’s no change to the gem specification and can be implemented even more cheaply. However, the downsides to this approach are:

1. Prefix registration may involve an application and/or manual review of some kind.
2. Gem name availability is not widened (i.e. configuration gem example), unless you reserve a prefix for your own username.
3. Some organizations do not follow a prefixing pattern (i.e. Rails with Active/ActionX, some AWS products, etc).
4. Enforcing new validation on an existing set of gems may result in conflicts and exceptions that need to be codified. An “open” prefixing reservation system (i.e. each user has their own prefix reserved by default) is certainly a backwards incompatible change.

## Alternative 2 - Scoping with dots (.)

In the current specification (although not documented), dots (`.`) are allowed. There are [a few “hidden” gems](https://rubygems.org/search?query=.) that use dots. If adding the `@` character is not desired, the dot character can be re-used to handle scoping, perhaps with the pattern `scope.gem_name` such as `aws-sdk.s3`.

## Alternative 3 - `@scope/gem_name` pattern.

While it might be preferable to use this pattern, it has implementation and legacy complications, specifically with forward slash `/`. Gem install and build will assume `@scope` as a directory and will fail. Gems installed in a location is currently flat and would have impact on existing tooling if installed within folders. Unless the `/` is escaped, the RubyGems.org API will also have issues resolving and displaying scoped gems because of Rails resource routing. This approach can be used with a more thought out design and a willingness for making large and downstream changes.

# Unresolved questions

**Adding `scope` to the gemspec**
As an alternative to overloading the gemspec's name, we can consider having a `scope` attribute, i.e.
```
  s.name = 'hola'
  s.scope = 'mullermp'
  s.version = '0.0.0'
```
This gem would still build as `hola@mullermp-0.0.0.gem` and have the same usage experience. However, would the gemspec file be named `hola@mullermp.gemspec` or `hola.gemspec`, and what implication would that have? If using `hola.gemspec`, is there a possibility someone may work with 2 `hola` gems from different organizations (x-organizational work). If the spec's name is `hola@mullermp`, we may as well overload the spec name too.

**Requirements for having an organization or for reserving gem scopes**
To prevent a rush on reservations, organizations and/or gem scopes should be limited by some initial process. Eventually this limitation would be removed. How should this be handled?

**Updated Regex to validate one single `@` character in the middle**
Optionally, the `VALID_NAME_PATTERN` regex can be updated to `\A[a-zA-Z][a-zA-Z0-9\.\-\_]+(@[a-zA-Z0-9\@\.\-\_]+)?\z`. This regex matches `hola` and `hola@mullermp` but does not match `@hola`, `hola@`, and `hola@hola@ruby`.

# Related Links

Namespace gem discussion - https://github.com/rubygems/rfcs/issues/31
Verified users proposal - https://github.com/rubygems/rubygems.org/pull/2698
