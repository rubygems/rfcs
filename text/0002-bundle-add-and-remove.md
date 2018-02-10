- Feature Name: bundle\_add\_and\_remove\_improvements
- Start Date: 2018-02-10
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

This feature adds improvements for `bundle add` command and adds
the opposite command for removing gems called `bundle remove`.

The purpose of this RFC is to get feedback on use cases, CLI format/syntax
and feature options.

# Motivation

The main use case is improving user productivity. 

This is done by adding gem(s) to the Gemfile with the latest version in the range specified from the command line. User saves time since they don't have to search Rubygems for the latest version. 

The opposite command is also available, which provides convenience for removing gem(s) from the Gemfile from the command line.

# Guide-level explanation

The usage of the `add` and `remove` command can be grouped in 7 sections.

## 1. adding gem to group block

### 1.1. when Gemfile doesn't yet have the group specified

#### Proposed command

```ruby
# Gemfile before
source "https://rubygems.org"
```

```ruby
bundle add minitest --group test
```

```ruby
# Gemfile after
source "https://rubygems.org"

group :test do
  gem "minitest", "~> 5.11"
end
```

### 1.2. when Gemfile has the group specified

#### Proposed command

```ruby
# Gemfile before
source "https://rubygems.org"

group :test do
  gem "minitest", "~> 5.11"
end
```

```shell
bundle add minitest-reporters --group test
```

```ruby
# Gemfile after
source "https://rubygems.org"

group :test do
  gem "minitest", "~> 5.11"
  gem "minitest-reporters" "~> 1.1"
end
```

### Description

Instead of adding gems to the Gemfile in the format

```ruby
gem "minitest", "~> 5.11", group: [:test]
```

the `add` command works with group blocks


```ruby
group :test do
  gem "minitest", "~> 5.11"
end
```

which are more natural for users.

It appends gems to the end of the group block if the group already exists. It also supports adding to multiple groups.

## 2. adding multiple gems with options

#### Proposed command

```shell
bundle add sinatra:2.0 minitest:~>5.11
```

### Description

The `add` command supports specifying versions for multiple gems.

## 3. adding gems with optimistic or strict version ranges

### 3.1 adding gems with strict version

#### Proposed commands

```shell
bundle add sinatra minitest --strict
```

```ruby
# Gemfile
gem "sinatra", "2.0.0"
gem "minitest", "5.11.3"
```

### 3.2 adding gems with optimistic version range

#### Proposed commands

```shell
bundle add sinatra minitest --optimistic
```

```ruby
# Gemfile
gem "sinatra", ">= 2.0.0"
gem "minitest", ">= 5.11.3"
```

### 3.3 adding gems without the version range

```shell
bundle add sinatra minitest --no-version
```

```ruby
# Gemfile
gem "sinatra"
gem "minitest"
```

### Description

The `add` command supports specifying different version ranges.

## 4. skipping install step

#### Proposed commands

```shell
bundle add sinatra minitest --skip-install
```

### Description

The `--skip-install` flag skips the install step and just adds the gem to Gemfile.

## 5. removing gems

#### Proposed commands

```ruby
# Gemfile before
source "https://rubygems.org"

group :test do
  gem "minitest"
  gem "minitest-reporters"
end
```

```shell
bundle remove minitest minitest-reporters
bundle rm minitest minitest-reporters
```

```ruby
# Gemfile after
source "https://rubygems.org"
```

### Description

The `remove` command removes gems from the Gemfile, along with their corresponding group block if it remains empty after the deletion. By default it runs `bundle install` afterwards, which can also be skipped with the `--skip-install` flag.


## 6. print flag

#### Proposed commands

```shell
bundle add sinatra --print
```

Prints

```
sinatra, "~> 2.0"
```

#### Description

The `--print` flag causes the gem name with the version to be printed out to STDOUT. It doesn't add the gem to the Gemfile and skips the install step.

This is useful when editing a Gemfile with the editor since it can be used to insert the latest version of a gem to a cursor position.

## 7. error messages

### 7.1. adding gem when it already exists in the Gemfile

```ruby
# Gemfile
source "https://rubygems.org"

gem "minitest"
```

```
bundle add minitest sinatra
```

Prints out and exits with error status code:

```shell
Gem "minitest" already present in the Gemfile. Aborting.
```

### 7.2. removing gem when it's not present

```ruby
# Gemfile
source "https://rubygems.org"
```

```
bundle rm minitest sinatra
```

Prints out and exits with error status code:

```
Gem "minitest" couldn't be found in the Gemfile. Aborting.
```


# Drawbacks

Backwards compatibility with `bundle add` command that didn't support multiple gems.

# Rationale and Alternatives

- *What is the impact of not doing this?*

The `add` command would not support multiple gems and flags for configuring what version ranges are added to the Gemfile. The install step wouldn't be skippable. The `remove` command wouldn't be present.


# Unresolved questions

- *What parts of the design do you expect to resolve through the RFC process before this gets merged?*

The API design for CLI for adding/removing gems to/from Gemfile.

- *What happens if a user wants to add gem(s) to multiple groups? How do we find the matching block to add the gem to?*

We probably have to permutate and find all the possible matches for given groups.
