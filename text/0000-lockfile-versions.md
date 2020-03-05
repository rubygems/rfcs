# Version lockfiles, not Bundler

- Feature Name: lockfile-versions
- Start Date: 2020-01-30
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)
## Summary 

Many years ago, the Bundler and RubyGems teams made an assumption that Bundler 1 and the future Bundler 2 would not both work with the same Gemfile.lock, to protect against possible future breaking changes. This assumption turned out to be wrong—Bundler 2 uses the same Gemfile and lockfile as Bundler 1. When Bundler 2 finally came out, some unfinished features and some bugs combined to break many applications and CIs that still depended on Bundler 1. For the technical details, read this [blog post about Bundler 2](https://eregon.me/blog/2020/01/13/a-migration-path-to-bundler2.html) by Benoit Daloze.

To ensure that future versions of Bundler will not break existing applications, this RFC proposes adding explicit versions to the Bundler lockfile format. In this RFC, we will call the current lockfile format “lockfile v1” and the upcoming (unreleased) lockfile format “lockfile v2”. By versioning the lockfile, we hope that any developer will be able to upgrade to any new Bundler version at any time, and keep working on the same projects, *without* forcing other developers on those projects to upgrade.


## Motivation

The Bundler 2 release was confusing for a complicated set of reasons, including a partially-functional automatic version switcher, some bugs in Bundler, some bugs in RubyGems, and some confusing error messages.

In the future, we want to be able to release Bundler major version bumps without disrupting users, or breaking anything that already works. The goal is to allow **users** to update Bundler as soon (or as late) as they want, while also allowing **applications** to require a higher version of Bundler at their own separate pace.

## Guide-level explanation

Here's an example of how Bundler versions and lockfile versions could interact.
Alice is a developer on two projects, PonyHub and Sparklebox. She uses Bundler 15, while PonyHub has a v2 lockfile (Bundler 3+), and Sparklebox has a v3 lockfile (Bundler 8+). Since Bundler 15 supports both lockfile v2 and v3, Alice is able to work on both projects.

When Bundler 16 comes out, Alice can upgrade as soon as she wants. Bundler 16 continues to work with v2 and v3 lockfiles, and other developers on her projects can upgrade whenever they want to upgrade.

If PonyHub decides to use a feature that requires a v3 lockfile, they can upgrade their lockfile by running `bundle lock --update`. After that, PonyHub developers running `bundle install` with Bundler <=7 will see an error message like this one:


> You are running Bundler 7.2.3, but this lockfile requires a newer version of Bundler. Upgrade by running `gem install bundler`, then try `bundle install` again.

Merely using newer Bundler versions will no longer update the `BUNDLED_WITH` value in the lockfile. If you have a lock file that says `BUNDLED_WITH 15.4.2`, and you run `bundle install` or `bundle update rack-bors` with Bundler 16, the `BUNDLED_WITH` value will stay the same.

That said, if Sparklebox discovers they need a bugfix from Bundler 16 for their app to work, they can bump their `BUNDLED_WITH` by running `bundle update --bundler`. (Or by running `bundle update` with no arguments, and updating everything.) After that, Sparklebox developers running `bundle install` with Bundler <=15 will see a warning like this one:


> WARNING: This application uses Bundler >= 16.0.3, but you have Bundler 15.4.2. You can disable this message by running `bundle config` `--``local disable_version_warning true`, but we recommend you upgrade by running  `gem install bundler`.

Bundler 3 and all higher versions should be able to install v1 and v2 lockfiles. In this example, Bundler 8 and higher should be able to install v1, v2, and v3 lockfiles.

If we put this policy in place before releasing Bundler 3 and lockfile v2, then Bundler 3 will continue to be able to install lockfiles created with Bundler 1, 2, and 3. The reason lockfile v2 exists is to solve a security issue that is only present in some lockfiles. Bundler 3 could require a lockfile update only for those applications, and allow all other applications to keep using lockfile v1.

By staying compatible with old lockfile versions, we hope Bundler can have the flexibility to make major version breaking changes (like dropping ruby versions), but keep its impressive stability and compatibility for the future.

## Reference-level explanation


1. Only drop RubyGems version support when also dropping the Ruby that ships with that RubyGems.
2. Remove Bundler version autoswitching from RubyGems.
3. Allow Bundler 2+ to install Bundler 1+ gemfiles without rewriting the lockfile or bumping the major version of `BUNDLED_WITH`.
4. Only increase the `BUNDLED_WITH` major version when a user runs `bundle update` `--``bundler`.
5. Add a `LOCKFILE_VERSION` field to the lockfile. Only increase `LOCKFILE_VERSION` when a user runs `bundle lock` `--``update`. Only prompt a user to update their lockfile if they need the new version. For Bundler 3 and lockfile v2, that means users with multiple sources that contain gems with the same name, but no one else.

The primary mechanism to accomplish separate versioning is keeping lockfile parser/generators for each lockfile version. If needed, an entire copy of previous lockfile code can be kept, along with adapter code for the latest data structure expected by other Bundler code. If future lockfile versions are very similar, it maybe possible to parse and generate multiple lockfile versions based on conditionals, without creating separate copies of the parser or generator.

## Drawbacks

The biggest reason to not do this is complexity: future versions of Bundler will need to keep and maintain code to read and generate old versions of the lockfile. It could also be confusing to know which version of Bundler you need, since the lockfile version will not map one-to-one to a Bundler version.

## Rationale and Alternatives

Despite the drawbacks above, this option appears to be the best possible option that can potentially meet Bundler’s current needs:


1. Developers need to be able to upgrade Bundler without forcing every project they work on to upgrade at the same time.
2. Projects need to be able to upgrade to the latest Bundler only when they are ready.
3. Each Ruby version can only have only one version of RubyGems installed at a time.

Bundler and RubyGems have so much overlap that their core teams have already combined into one team. We want to eventually make Bundler a command provided by RubyGems itself, but any copy of Ruby can only have one version of RubyGems installed at a time. Combining items 1, 2, and 3 together means that we need to support developer Alice upgrading to Bundler 15, while still allowing her to work on PonyHub, without forcing PonyHub to upgrade to Bundler 15.

The only other option that seems even slightly viable is having RubyGems or Bundler automatically download and install and exec to older versions, but that solution is both *very* confusing to end users, can go very wrong if it doesn’t work, and still doesn’t solve the problem of merging Bundler into RubyGems and only being able to install one version of Bundler per Ruby.

## Unresolved questions

We thought the Bundler Version Switcher feature in RubyGems would meet our needs, and that turned out to be… not true. Will this proposed solution actually meet everyone’s needs in real life?

What implementation option do we want to choose to support installing v1 and v2 lockfiles, as well as allow one-way upgrades from v1 to v2? Separate lockfile parsers per version? Version flags and lots of conditionals?

How should we communicate with users about this? Bundler.io and RubyGems.org blog posts? Conference talks?

How soon should we fully remove the Bundler Version Finder from RubyGems? Immediately? In RubyGems 3.1? In RubyGems 4? Will this change break anything in ruby-core?

What upgrade paths will we support to migrate from the current RubyGems version that still have the Bundler Version Finder to a future RubyGems and Bundler combo that is more “backwards compatible”?
