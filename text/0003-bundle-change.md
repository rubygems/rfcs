- Feature Name: bundle\_change
- Start Date: 2018-07-22
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

This RFC proposes a new bundle change command. This command makes Gemfile editable via command line.

# Guide-level explanation

At an abstract level the change command is a composite of remove and add command. The requested gem would be first removed and then added with the supplied options (which it supports) and then runs `bundle install`.

This feature should be viewed as a quick way of editing Gemfile. It makes changing basic properties of a gem easier.

## Basic Usage

```bash
$ cat gems.rb | grep "rack"
gem "rack", :group => :dev

$ bundle change rack --group=prod

$ cat gems.rb | grep "rack"
gem "rack", :group => :prod

```

## Edge cases

Edge cases include:

- gem not in gemfile,
`gem could not be found in the Gemfile.`
- supplying an invalid option
- no options supplied
`Please supply at least one option to change.`
- handling an unsupported gem property
`option is not yet supported.`

# Reference-level explanation

This command fully relies on `remove` and `add` command for functioning. There are individual limitations of both of these commands so proper and reliable working is highly dependent on them.

Some corner cases that have been mentioned in above section are explained below with example.

- gem not in gemfile,

```ruby
# Gemfile
source "https://rubygems.org/"

gem "rack", "~> 1.2", :groups => [:test, :dev]
```

```bash
dbundle change rails --group=test
# `rails` could not be found in the Gemfile.
```

- no options supplied.

```bash
dbundle change rails
# Please supply at least one option to change.
```

- handling an unsupported gem property

```ruby
# Gemfile
source "https://rubygems.org/"

gem "rack", "~> 1.2", :platforms => [:jruby]
```

```bash
dbundle change rack --group=test
# `platforms` is not yet supported.
```

A successfull run is illustrated below

```ruby
# Original Gemfile
source "https://rubygems.org/"

gem "rack", "~> 1.2"
```

```bash
dbundle change rack --group=test
```

```ruby
# New Gemfile
source "https://rubygems.org/"

gem "rack", "~> 1.2", :groups => [:test]
```

# Drawbacks

There aren't any drawbacks, but special care is needed to prevent any unintened loss of data from Gemfile.

# Unresolved questions

- How to handle options that add does not support? Should we add support for those options in add itself?
- Should we add `--skip-install` option as in `add` command to prevent it to run `bundle install`?
