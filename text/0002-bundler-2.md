# Bundler 2 RFC

This document outlines the changes proposed for the Bundler v2 release. The main goal of this release will be to remove
support for Ruby < 2.0, remove and deprecate existing functionality and introduce any changes to existing behaviour.

## Removed Functionality
* The `bundle viz` and `bundle show` commands. The `viz` command will be moved to a Gem/Bundler Plugin and `show` will be removed in favour of `bundle info` and `bundle list`
* The `bundle_ruby` executable [#3489]
* Capistrano integration in `bundler/capistrano.rb` [#3551]
* remove `--no-cache`, `--binstubs`, `--path`, `--with`, `--without`, `--system` options from `bundle install` [#3555, #3774, #3827, #3937, #3939, #3994]
* The `bundle cache` command implementation is removed [#4008]
* No longer checking for older version of Bundler in the $LOAD_PATH in `exe/bundler` [#3662]
* The `disable_shared_gems` option [#3999]
* The `bundle console` command [#4034]
* RubyGems Aggregate & support transitive source pinning [#4714]

## New Functionality
* Add `--cache` option to `bundle install` [#3555]
* Summarize the number of using gems [#3872]
* cache downloaded gems across applications [#3983, #5782]
* Infer the version of ruby from `.ruby-version` [#4036]
* Add `--shebang` option to `bundle binstubs` [#4070]
* Specify ruby version when using the gemspec method [#4321]

## Breaking Changes
* remove support for version of Ruby < 2.0 [#3842]
* The `install` command no longer remembers the given options, ie: `bundle install --path vendor` [#3955]
* Updating all gems in the bundle through the `bundle update` command will now require the `--all` option. [#2646]
* `bundle update` now requires a gem name unless the `--all` option is provided.
* `bundle install` will install gems into `.bundle/gems` by default [#3784]
* Don't create gem home before setting it [#2884]
* The `--path` option for `bundle install` will raise an error notifying the user to use `bundle config path <path/to/bundle>` instead.
* `bundle` without any subcommand will print the help page instead installing gems [#3831]
* The `Gemfile` and `Gemfile.lock` will move to `gems.rb` and `gems.locked` [#3834]
* Allow mis-matches in bundler requirements [#3871]
* The `bundle package` command is now `bundle cache` [#4008]
* Use 'bundle gem --exe' as primary option name, instead of '--bin' [#4261]
* Print only version number in `bundler --version` [#4708]
* Dont update git repos when --local is passed [#4535]
* Skip eval'ing the Gemfile multiple times when a lockfile is present [#4952]

## Deprecated
This is functionality that in Bundler that will be deprecated in Bundler 2 and removed in v3
* Gemfile and Gemfile.lock
