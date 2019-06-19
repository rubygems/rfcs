- Feature Name: stdlib-gems
- Start Date: 2019-06-18
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

## Summary
As more of Ruby’s stdlib is turned into gems, Bundler’s “no gem dependency” policy is getting harder and harder to maintain. This proposal aims to record the official plan for dealing with stdlib gems. That plan is: 1) try to avoid dependencies, 2) vendor dependencies when possible, 3) if we can’t avoid a dependency and we can’t vendor it (like openssl), document the version we will lock to and print a warning if the application version does not match, 4) eventually separate `bundler-setup` so that it can be run standalone without any dependencies. Then the `bundler` gem can have dependencies for running CLI commands other than `exec`.

## Motivation
We need to have a plan as more gems we depend on (for example, fileutils, tsort, net/http, openssl, and others) become external gems that can have varying versions on a single version of Ruby.

By making this plan, we should be able to handle future gemification without breaking Bundler.

## Guide-level explanation
Here are some examples of each proposed step:

### 1) Try to avoid dependencies
At the most surface level, this means not depending on gems or stdlib if we can avoid it. Don’t add anything that depends on activesupport (😂). If you want a method from a stdlib gem, see if you can implement it yourself or copy the implementation into Bundler before you try to vendor the entire gem.

### 2) Vendor dependencies when possible
If you really need to depend on a gem (like Thor, or net-http-persistent), use Automatiek to vendor a copy of the gem into Bundler’s codebase with module namespacing. That means other applications can load their own versions of Thor, etc.

### 3) Document deps and warn on mismatches
If there’s no way to avoid depending on a gem (like openssl, soon), make sure that Bundler explicitly spells out what version(s) of the gem it is compatible with, and prints a warning if either 1) the version Bundler required is not on the list Bundler knows it is compatible with, or 2) the application is locked to a different version than the one Bundler required.

### 4) Separate out `setup`, let the CLI have dependencies
If it’s possible to `bundle exec` or `require 'bundler/setup'` without requiring any dependencies at all, that would mean that the CLI could have normal dependencies on net-http-persistent and Thor, without having to do the awkward vendoring dance. It would also mean that Bundler could declare a dependency on specific versions of the OpenSSL gem, and actually know that those versions would be available anytime it was installed. 

## Drawbacks
There are drawbacks to all of the different options involved, here: avoiding deps makes developing new features on Bundler harder, vendoring deps makes updates hard and awkward, requiring gems that aren’t declared dependencies means they might not even be installed let alone the correct version, and carefully separating the CLI from `bundler/setup` is going to be even more work than we already have to do to handle dependencies without breaking anything.

Basically the drawbacks are “this is a ton of work”.

## Rationale and Alternatives
If we don’t do anything about this, Bundler users will start to either see exceptions (when RubyGems tries to activate a specific gem version but Bundler has already activated a different version), or they will see mysterious errors resulting from the newest installed version of a stdlib gem being loaded instead of whatever version of that gem is in their lockfile.

## Unresolved questions
- Is this enough to completely resolve the issue?
- Are there any other options that would get us the same result?
- How hard will it be to separate out the CLI and let it have dependencies?

