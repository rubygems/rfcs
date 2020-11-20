- Feature Name: bundler_version_locking
- Start Date: 2020-09-01
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

If a user specifies a required Bundler version in the Gemfile/gemspec, it should be installed and used during the normal `bundle install`/`bundle exec` workflow.

# Motivation

There are many times where locking your Bundler version is useful. The existence of `BundlerVersionFinder` shows that, but that approach has been confusing for end-users.

# Guide-level explanation

If you need to pin the Bundler version, simply specify it in your `Gemfile` or `gemspec and run `bundle install` as usual.

If the running version doesn't meet the requirements, Bundler will install the specified version of itself, and then re-run itself using that version.

## Example 1

For this example, assume Bundler 2.1.4 is installed but Bundler 2.0.2 is not.

Gemfile:

```
source "https://rubygems.org"

gem "bundler", "= 2.0.2"
```

Output, first run:

```
$ bundle install
Bundler 2.1.4 is being run, but "Gemfile" requires version "= 2.0.2".
Fetching bundler-2.0.2.gem
(... rest of output from installing Bundler 2.0.2 ...)
Using bundler 2.0.2
(... rest of output, as normal ...)
```

Output, second or later run:
```
$ bundle install
Using bundler 2.0.2
(... rest of output, as normal ...)
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
Using bundler 2.0.2
(... rest of output, as normal ...)
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

Output, first run:

```
$ bundle install
Bundler 2.1.4 is being run, but "blah.gemspec" requires version "~> 2.0".
Using bundler 2.0.2
(... normal output ...)
```

Output, second or later run:

```
$ bundle install
Using bundler 2.0.2
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

Then, Bundler would do the following when `bundle` is executed:

1. If the first argument isn't `_<bundler version>_` _and_ the Gemfile/gemspec specify a required version of Bundler _and_ the requirement isn't met by the currently-running version:
   a. Install the required version of Bundler, if needed.
   b. Replace the current process with `bundle _<required version>_ <args...>` (e.g. something along the lines of `Kernel.exec($0, "_#{required_version}_", *ARGV)`)
2. Run as normal.

# Drawbacks

TBD. (I'm sure there are some.)

# Rationale and Alternatives

The approach in this RFC tries to ensure:

1. It is inherently opt-in: it won't do anything if you don't explicitly list Bundler as a dependency.
2. The user has more control:
    - It's opt-in, so it won't get in the way if it's not actively wanted.
    - By implementing it in terms of the `_<some version>_` feature, we provide a way for users to override the behavior if needed.
3. It builds on existing conventions:
    - Locking the Bundler version is done in the same place and way as any other dependency.
    - Changing the locked Bundler version is done the same way as any other dependency.

I am not aware of any alternatives that accomplish all of these.

# Unresolved questions

There are many quality-of-life things that could be added, like telling users if they're relying on an outdated Bundler version, but I feel this can be added after the fact.
