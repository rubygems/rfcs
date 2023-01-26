- Feature Name: Add `gem exec` command to install & run an executable from a gem
- Start Date: 2023-01-24
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Add a `gem exec` subcommand to RubyGems that will handle installing (if necessary) and running an executable from a gem (on rubygems.org or the user's configured gem server), regardless of whether that gem is currently installed.

# Motivation

Inspired by [`npx` / `npm exec`](https://docs.npmjs.com/cli/v9/commands/npm-exec). This already exists as a separate gem called [gemx](https://github.com/rubygems/gemx), but is far less discoverable and dependable as a standalone gem than as a built-in RubyGems command.

# Guide-level explanation

The `gem exec` command can be thought of as a shortcut to running `gem install` and then the executable from the installed gem.

For example, `gem exec rails new .` will run `rails new .` in the current directory, without having to manually run `gem install rails` only to run `rails new` once, and no other `rails` commands (because `rails new` generates a Gemfile).

For a more complete example:

```sh
$ gem exec --gem cocoapods -r '> 1' -r '< 1.3' -v -- pod install --no-color --help
```

will run `pod install --no-color --help`, using the `pod` executable from the `cocoapods` gem, specificall a version between 1 and 1.3.

The full command help is as follows:

```
Usage: gem exec [options --] COMMAND [args] [options]

  Options:
        --platform PLATFORM          Specify the platform of gem to exec
    -v, --version VERSION            Specify version of gem to exec
        --[no-]prerelease            Allow prerelease versions of a gem
                                     to be installed
    -g, --gem GEM                    run the executable from the given gem


  Install/Update Options:
        --conservative               Prefer the most recent installed version,
                                     rather than the latest version overall


  Common Options:
    -h, --help                       Get help on this command
    -V, --[no-]verbose               Set the verbose level of output
    -q, --quiet                      Silence command progress meter
        --silent                     Silence RubyGems output
        --config-file FILE           Use this config file instead of default
        --backtrace                  Show stack backtrace on errors
        --debug                      Turn on Ruby debugging
        --norc                       Avoid loading any .gemrc file


  Arguments:
    COMMAND  the executable command to run

  Summary:
    Run a command from a gem

  Description:

  Defaults:
    --version '>= 0'
```

# Reference-level explanation

A new `gem` subcommand called `exec` will be added to RubyGems.

`gem exec` will use its own gem path to install gems (when needed), and will attempt to load the requested binary from the existing set of gems.
If the binary is not found, the command will install the gem (by default of the same name as the binary, though it can be set by a command line option),
and then activate and execute that binary.

# Drawbacks

This proposal should have no impact on existing users, since it is purely additive.

One potential area of confusion is with Bundler / users using Gemfiles.
To start, we plan on ignoring the Gemfile entirely when running `gem exec`, which may be confusing if the user is attempting to use the Gemfile to customize gem sources.
Additionally, `gem exec` is confusable with the `bundle exec` command, and they will work differently. A potential way to ameliorate that confusion is for `gem exec` to check if the requested executable exists in the current bundle, and advise the user to use `bundle exec` in that case.

# Rationale and Alternatives

We could continue to point users to the existing [`gemx`](https://github.com/rubygems/gemx) gem to solve this problem, or encourage using `gem install $gem && $gemBinary`, but those solutions (which have existed for years) are not sufficiently discoverable.

Implementing this proposal will make it easy for users to run one-off commands that _happen_ to be shipped as gems, and instructions to `gem exec $gem` can become widespread.

# Unresolved questions

Should there be an option to use an isolated gem path for `gem exec` installs? Should it be required or the default? Should we look into using existing gems but not installing into the default path?

Out of scope: interactions with Bundler / Gemfiles.
