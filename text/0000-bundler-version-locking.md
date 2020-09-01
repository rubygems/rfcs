- Feature Name: bundler_version_locking
- Start Date: 2020-09-01
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

If a user specifies a required Bundler version in the Gemfile/gemspec, it should be installed and used during the normal `bundle install`/`bundle exec` workflow.

# Motivation

There are many times where locking your Bundler version is useful. The existence of `BundlerVersionFinder` shows that, but that approach clearly has not worked. **This RFC assumes that `BundlerVersionFinder` is removed first. The discussion on whether or not to do that should be had elsewhere.**

# Guide-level explanation

## Example 1

For this example, assume Bundler 2.1.4 is installed but Bundler 2.0.2 is not.

Gemfile:

```
source "https://rubygems.org"

gem "bundler", "= 2.0.2"
```

Output:

```
$ bundle install
Bundler 2.1.4 is being run, but "Gemfile" requires version "= 2.0.2".
Installing Bundler 2.0.2...
(... output of installing Bundler 2.0.2 ...)
Switching to Bundler 2.0.2...
(... rest of output, as normal ...)
$ bundle install
Bundler 2.1.4 is being run, but "Gemfile" requires version "= 2.0.2".
Switching to Bundler 2.0.2...
(... normal output ...)
```

## Example 2

For this example, assume both Bundler 2.0.2 and Bundler 2.1.4 are installed.

Gemfile:

```
source "https://rubygems.org"

gem "bundler", "~> 2.0"
```

Output:

```
$ bundle install
Bundler 2.1.4 is being run, but "Gemfile" requires version "~> 2.0".
Switching to Bundler 2.0.2...
(... normal output ...)
```

## Example 3

For this example, assume both Bundler 2.0.2 and Bundler 2.1.4 are installed.

Gemfile:

```
source "https://rubygems.org"

gemspec
```

blah.gemspec:

```
<...>
   spec.add_development_dependency "bundler", "~> 2.0"
<...>
```

Output:

```
$ bundle install
Bundler 2.1.4 is being run, but "blah.gemspec" requires version "~> 2.0".
Switching to Bundler 2.0.2...
(... normal output ...)
```

## Example 4

For this example, assume both Bundler 2.0.2 and Bundler 2.1.4 are installed.
Note that since the default behavior is to run the newest version installed, and that matches the requirement, it never needs to switch versions.

Gemfile:

```
source "https://rubygems.org"

gem "bundler", "~> 2.1"
```

Output:

```
$ bundle install
(... normal output ...)
```

# Reference-level explanation

First, the `BundlerVersionFinder` would be removed, as mentioned in "Motivation."

Then, Bundler would do the following when `bundle` is executed:

1. If the first argument isn't `_<bundler version>_` _and_ the Gemfile/gemspec specify a required version of Bundler _and_ the requirement isn't met by the currently-running version:
   a. Install the required version of Bundler, if needed.
   b. Replace the current process with `bundle _<required version>_ <args...>` (e.g. something along the lines of `Kernel.exec("bundle", "_#{required_version}_", *args)`)
2. Run as normal.

# Drawbacks

TBD. (I'm sure there are some.)

# Rationale and Alternatives

The main alternative is a refinement of the BundlerVersionFinder, but I think the approach in this RFC better follows the [principle of least surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment) by relying on existing knowledge and assumptions.

The approach in this RFC tries to ensure:

1. It is inherently opt-in: it won't do anything if you don't explicitly list Bundler as a dependency.
2. The user has more control:
    - It's opt-in, so it won't get in the way if it's not actively wanted.
    - By implementing it in terms of the `_<some version>_` feature, we provide a way for users to override the behavior if needed.
3. It builds on existing conventions:
    - Locking the Bundler version is done in the same place and way as any other dependency.
    - Changing the locked Bundler version is done the same way as any other dependency.

# Unresolved questions

There are many quality-of-life things that could be added, like telling users if they're relying on an outdated Bundler version, but I feel this can be added after the fact.

Things I want to figure out:

1. If the version it needs is found, should it still print a message explaining what version was ran and which was switched to? Or should it avoid printing anything unless there's a problem?
   - In theory, passing `--verbose` includes this, because the first line from `bundle install --verbose` is `Running `bundle install --verbose` with bundler 2.0.2`
