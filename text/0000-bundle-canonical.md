- Feature Name: bundle_canonical
- Start Date: 2018-08-03
- RFC PR:
- Bundler Issue:

# Summary

Add a new `bundle canonical` feature. The `bundle canonical` command will modify the gemfile by giving it consistent formatting and beautify it by adding summary of the gems if required.

The purpose of this RFC is to get feedback on use cases, cli format/syntax, and feature options.

# Motivation

The main use case is to make a large (maybe not easily maintainable) Gemfile consistent.

# Detailed design

The usage:

```ruby
# Gemfile

source "https://rubygems.org/"

gem "rack"
```

```bash
$ bundle canonical
```

```ruby
# Gemfile after

source "https://rubygems.org/"

# a modular Ruby webserver interface
gem "rack"
```

Adding `--dry-run` option does not change the Gemfile rather shows what the Gemfile would look like if run without `--dry-run`.

# How We Teach This

An addition to the documentation. To bring awareness among users, I can think of normal blog articles, and/or bundler press/social media channels.

# Drawbacks

I don't think there are any.
