- Feature Name: override_gem_dependency_version
- Start Date: 2018-07-10
- RFC PR:
- Bundler Issue:

# Summary

Enable the version constraints of gem dependencies to be explicitly overridden for gems listed in a Gemfile.

The purpose of this RFC is to get feedback on use cases, format/syntax/DSL, and feature options.

# Motivation

This feature has been requested many times since 2011.

Project authors want the ability to specify newer versions of gems in their Gemfile even if the version is being constrained by another gem.

There several causes for this issue:
- The version dependency specified in the gemspec is unnecessarily strict: `s.add_dependency 'json', '= 1.7.7'`
- The gem is slow to update or is no-longer maintained to the point that version constraints do not keep up with the version updates of their dependencies: `s.add_dependency 'json', '>= 1.7.7', '<= 2'`

The common approach to this problem is to fork the problematic gem, relax the dependencies specified in the gemspec, and reference the fork in the Gemfile. This is a very heavyweight and time intensive solution to a problem that could be easily solved by a well-designed solution to override dependencies from within the Gemfile itself.

Obviously, overriding version constraints of gems would be *AT YOUR OWN RISK*, however, a properly defined DSL and output messages should mitigate and address these issues.

Most of the time, this kind of override should *NOT* be a permanent solution, and as such, *SHOULD* be accompanied with some type of message that is additionally displayed to indicate why the override has occurred.  

# Guide-level explanation

Say you want to use the latest version of the `json` gem (currently `2.1.0`), however, another gem you are using constrains the version to `s.add_dependency 'json', '>= 1.7.7', '<= 2'`.  To override this version constraint so that version `2.1.0` is installed:

```ruby
gem 'json'
gem 'other_gem' do |s|
  s.override_runtime_dependency('json', '>= 1.7.7', '<3')
end
```

This will allow bundler to install the `2.1.0` version of the `json` gem.

The output when running `bundle install` will look like the following:

```
Installing json 2.1.0 (forced version, conflicts with other_gem)
Installing other_gem 1.0.0
```

Most of the time, this should only be a temporary fix as we expect the gem author to release a new version with updated dependency constraints.

To provide additional information a note can be included:

```ruby
gem 'json'
gem 'other_gem' do |s|
  s.override_runtime_dependency('json', '>= 1.7.7', '<3', 'waiting on pull request #5')
end
```

The output when running `bundle install` will look like the following:

```
Installing json 2.1.0 (forced version, conflicts with other_gem: waiting on pull request #5)
Installing other_gem 1.0.0
```

# Reference-level explanation (TODO)

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks

These are the common drawbacks and counter-arguments mentioned by the community:
* This issue can already be solved by forking gem repositories and updating the dependencies yourself
* Allowing this means there is no longer a canonical source of truth about what a gem's dependencies are
* When these kinds of changes cause resolver errors (and they will), it will be extremely hard to diagnose
* The potential that people will complain that Bundler is broken when it’s actually something they did themselves
* The potential that people will complain to the gem author that the gem is broken when it’s actually something they did themselves

These are all valid points of view, however, there are also counter-arguments to each of these points which are also equally valid:
* The most obvious place to manage version dependencies (even overridden version constraints) is in the Gemfile and not in a forked repository
* Forking an entire project (and subsequently managing upstream changes, etc.) is an unnecessarily tall order for something that can certainly be fixed with a simple, at-your-own-risk dependency override
* This feature becomes more and more important with the number of gems growing. .NET has it. Java has it. Nothing bad happened to them because of it
* Maintaining overridden version constraints within the Gemfile saves a significant amount of development cost when compared to maintaining a forked repository
* Explicit syntax within the Gemfile with custom notes and obvious output messages when running `bundle install` can help to mitigate developers not knowing that a version conflict exists

# Rationale and Alternatives

This is the best design because it is very explicit and clearly documents what is happening.
It requires project maintainers to have a clear understanding of the conflicts they are encountering.

Other designs are as follows:

```ruby
gem "puma", "5.1" # depends on rack <= 2.0
gem "rack", "3.0", force_version: true
```

This version forces the `rack` gem to explicitly force the install of a version matching `3.0`.
This syntax will work fine, but it is not documenting which is the problematic gem preventing the requested version of `rack`.

The equivalent using the proposed solution would look like this:

```ruby
gem "puma", "5.1" do |s|
  s.override_runtime_dependency "rack", "~> 3.0"
end
gem "rack", "3.0"
```

The proposed solution clearly documents that the problematic gem is `puma` and that we are explicitly overriding it's version constraints for the `rack` gem.

The impact of this is verbose and possibly unsightly code.  The benefit, however, is that version overrides are explicitly documented which will help prevent excessive issues being lodged to bundler and to gem authors.

# Unresolved questions

* What is the best mechanism to show override messages to the user?
* What is the best syntax to include a custom override message?
