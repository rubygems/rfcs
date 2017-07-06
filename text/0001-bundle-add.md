- Start Date: 2016-12-04
- RFC PR: https://github.com/bundler/rfcs/pull/4
- Bundler Issue: https://github.com/bundler/bundler/pull/5486

# Summary

Add a new `bundle add GEM` feature. The `bundle add GEM` command will add the gem to the gemfile (if valid) and run bundle install in one step. (Refer: [#4901](https://github.com/bundler/bundler/issues/4901)).

The purpose of this RFC is to get feedback on use cases, cli format/syntax, and feature options.

# Motivation

The main use case is to deploy scripts and/or improve user productivity by adding a gem to Gemfile and bundle install in one step.

The question is how far to extend this feature, is there a need to support all gem options like version, source, and group. And whether or not it should support multiple gems with their respective options.

This command overlaps some functionally with the current `bundle inject GEM` command.

# Detailed design

The usage of the feature can be grouped into 4 categories:

## 1. add a single gem
#### Proposed command
```ruby
bundle add "GEM"

#examples
bundle add "rails"
```
#### Description
This is the base use case. The following steps will occur when the command is executed:
1. It will load the current Gemfile and resolve the new GEM with it.
2. If it fails to resolve correctly or is an invalid GEM, it will error out.
3. If it resolves successfully, it will install the latest version of the GEM.
4. It will add the GEM to the Gemfile and place a conservative version update requirement* e.g.
```ruby
  # Gemfile
  gem "GEM", "~> 1.0.0"
```

## 2. add a single gem with options
#### Proposed command
```ruby
# First priority
bundle add "GEM" [--version VERSION]
bundle add "GEM" [-v VERSION]

# Possible full implementation
bundle add "GEM" [--version VERSION] [--source SOURCE] [--group GROUP]
bundle add "GEM" [-v VERSION] [-s SOURCE] [-g GROUP]

#examples
bundle add "rails" --version "~> 5.0.0"
bundle add "rails" --version ">= 5.0, < 5.1"
bundle add "rails" --version "~> 5.0.0" --group "development, test"
bundle add "rails" --version "~> 5.0.0" --source "https://gems.example.com" --group "development"
bundle add "rails" -v "~> 5.0.0"
bundle add "rails" -v ">= 5.0, < 5.1"
bundle add "rails" -v "~> 5.0.0" -g "development, test"
bundle add "rails" -v "~> 5.0.0" -s "https://gems.example.com" -g "development"
```
#### Description
This extends the base case allowing a user to specify optional version, source, and group.

The following steps will occur when the command is executed:
1. It will load the current Gemfile and resolve the new GEM with it.
2. If it fails to resolve correctly or is an invalid GEM/options, it will error out.
3. If it resolves successfully, it will install the specified or latest version of the GEM.
4. It will add the GEM to the Gemfile and the specified options.

```ruby
  # Gemfile
  gem "GEM", "~> 5.0.0"
  gem "GEM", ">= 5.0", "< 5.1"
  gem "GEM", "~> 5.0.0", group: ["development", "test"]
  gem "GEM", "~> 5.0.0", source: "https://gems.example.com", group: "development"
```
## 3. add multiple gems
#### Proposed command
```ruby
bundle add "GEM1" "GEM2" "GEM3"

#examples
bundle add "rails" "devise" "simple_form"
```
#### Description
This extends the base case allowing multiple gems to be added.
The following steps will occur when the command is executed:
1. It will load the current Gemfile and resolve the new GEM's with it.
2. If it fails to resolve correctly or there are invalid GEM's, it will error out.
3. If it resolves successfully, it will install the latest version of each GEM's.
4. It will add the GEM's to the Gemfile and place a conservative version update requirement* e.g.
```ruby
  # Gemfile
  gem "GEM1", "~> 1.0.0"
  gem "GEM2", "~> 1.0.0"
  gem "GEM3", "~> 1.0.0"
```

## 4. add multiple gems with options
#### Proposed command
```ruby

# TBD
bundle add "MULTIPLE_GEMS + OPTIONS ???"
```
#### Description
This case presents complexities in maintaining bundlers cli consistency while keeping a level of simplicity in command use with flexibility.

I personally don't see a strong case for this and think if a user has to add many gems with so many options, it will be easier to either edit the Gemfile or doing it as single gem additions (slower since bundle install runs each time).

*Per @indirect, adding specific versions or version constraints should be allowed here using the same syntax as gem install: for example, bundle add GEM1:2.0 GEM2:~>1.2 GEM3:>=4.0.*

## *Conservative version addition in Gemfile

the version that is chosen depends on the version of the gem that is available. for example, if the version chosen by the resolver is < 1.0, the constraint should have three places e.g. version 0.12.3 becomes ~> 0.12.3. If the resolved version is > 1.0, the constraint should only have two places e.g. version 1.2.3 becomes ~> 1.2.

# How We Teach This

This will require an addition to the documentation. To bring awareness and train users, I can think of normal blog articles, and current bundler press/social media channels.

# Drawbacks

This will overlap functionality with the `bundle inject` command but it's more descriptive as it fits a users mental model better (add something vs inject something).

# Alternatives

*(What other designs have been considered? What is the impact of not doing this?)*

# Unresolved questions

1. Of the 4 categories above, which ones will be most commonly used to prioritize development? The current plan is first get category 1 (single gem) and category 2 (single gem with version option only) working.

2. For category 2 (add a single gem with options), there are other cli format's to consider. However, I think these break consistency with bundler's cli and can present some confusion to users. Some alternatives being considered are:
```ruby
bundle add "GEM" [VERSION] [--source SOURCE] [--group GROUP]
bundle add "GEM" [-v VERSION] [-s SOURCE] [-g GROUP]

#examples
bundle add "rails" "~> 5.0.0" --group "production"
bundle add "rails" --version "~> 5.0.0" --group "production"
bundle add "rails" -v "~> 5.0.0" -s "https://gems.example.com" -g "development"
```
Update: It's been decided to implement this short form.

3. Will this eventually replace `bundle inject`? If so, what will be the deprecation plan?
*Per @indirect, no, it will not replace inject — inject is designed to be run by a completely automated tool with no human interaction or input, and it seems fine to leave that around for now.*
