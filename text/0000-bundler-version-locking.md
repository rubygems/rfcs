- Feature Name: bundler_version_locking
- Start Date: 2020-09-01
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Bundler should respect the exact version specified for a project.

If there is a lockfile (typically `Gemfile.lock`) with a `BUNDLED WITH` statement,
it should install and use that Bundler version.

Otherwise, if there is a dependency on `bundler` (in `Gemfile` or equivalent),
it should install and use that Bundler version.

This should happen transparently during the normal `bundle install`/`bundle exec` workflow.

# Motivation

## Usability

Bundler is used in a wide variety of scenarios. Experience has proven that
seemingly-minor changes in Bundler can cause things to break in certain
situations, even if the intention was to preserve compatibility.

By explicitly listing Bundler as a dependency, like it allows you to do with
other dependencies, you can ensure people are using the expected versions.

To keep the exact version, either use a tight constraint (e.g. `= 2.2.24`)
or use a lockfile. To use a version expected to be compatible, you can
use a `~>` constraint (e.g. `~> 2.2.0`).

## Source Analysis

Bundler is currently the only Ruby dependency where you can not reliably
determine from the source code for a project which version will be run.
Every other gem a project depends on is included in the `Gemfile.lock`,
and the Ruby version is in the `Gemfile` and/or `.ruby-version`.

Even bundler-audit [does not currently check the Bundler version](https://github.com/rubysec/bundler-audit/pull/299)
due to the complexity of doing so.

Any sufficiently complex software has security vulnerabilities, and Bundler
is no exception. By including the version of Bundler used in Gemfile.lock,
and respecting the version in the Gemfile when Gemfile.lock does not exist,
we bring Bundler in line with the rest of the dependencies.

## Forcing Bundler Updates

If Bundler has a bug that causes problems for you, or a vulnerability that
affects you, the best you could currently do is have a script install the
appropriate version. However, this can only work if people remember to run
the script.

By allowing the Gemfile and Gemfile.lock files to dictate the version of
Bundler that is used, it becomes possible to force Bundler updates to avoid
bugs and vulnerabilities. Running `bundle install` becomes enough to update
_every_ dependency, including Bundler itself.

# Guide-level explanation

Pin the Bundler version for a project by either:

1. Commiting `Gemfile.lock`.
2. Adding `gem "bundler", some_version_constraint` to Gemfile.

If you take approach 1, you can upgrade the locked Bundler version by
running `bundle update --bundler`.

If you take approach 2, you can upgrade the locked Bundler version by
changing the version constraint and running `bundle install`.


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

When executing a Bundler command, it should do the following:

1. If the first argument is `_<bundler version>_`:
    1. If the specified version is running, skip to step 5.
    2. Install the specified version, if needed.
    3. Re-execute Bundler using the specified version.
2. If there is a lockfile with a `BUNDLED WITH` statement:
    1. If the specified version is running, skip to step 5.
    2. Install the version specified, if needed.
    3. Re-execute Bundler using the specified version.
3. Resolve dependencies.
4. If the resolved dependencies include `bundler`:
    1. If the specified version is running, skip to step 5.
    2. Install the Bundler version specified, if needed.
    3. Re-execute Bundler using the specified version.
5. Run as normal.

# Drawbacks

This does add localized complexity to part of the codebase, either in the binstub or the `bundle install` process (depending on the implementation).

# Rationale and Alternatives

The approach in this RFC tries to ensure:

1. It is transparent about what is occurring.
2. The user stays in control:
    - By respecting the `_<some version>_` feature, we provide a way for users to override the behavior if needed.
3. It builds on existing conventions:
    - Locking the Bundler version is done in the same place and way as any other dependency.
    - Changing the locked Bundler version is done the same way as any other dependency.

I am not aware of any alternatives that accomplish all of these.

# Unresolved questions

1. There are many quality-of-life things that could be added, like telling
   users if they're relying on an outdated Bundler version, but these can be
   added after the fact.
2. The exact implentation is still unclear &mdash; it could be part of
   `bundle install`, or installing the right Bundler version could be handled
   by the `bundle` binstub.
3. How do we [preserve the system environment](https://github.com/rubygems/rfcs/pull/29#issuecomment-735416819)?
   ```
   $ MY_ENV=a ruby -e 'ENV["MY_ENV"]="b";Kernel.exec("ruby", "-e", "print ENV[\"MY_ENV\"]")'
   b
   ```
