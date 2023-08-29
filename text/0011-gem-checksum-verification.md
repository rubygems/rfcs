- Feature Name: `gem_checksum_verification`
- Start Date: 2023-08-29
- RFC PR: https://github.com/rubygems/rubygems/pull/6374
- Bundler Issue: (leave this empty)

# Summary

Bundler is adding checksum verification of gems when they are installed. It should be secure by default and easy to use. It should not break assumptions or unnecessarily break deployment or CI unless there is a security problem.

# Motivation

Verifying gem checksums with known checksums at install time is a stronger way to verify that the exact same gem source is being used in every environment where the bundle is installed. Using checksums, especially those sourced from rubygems.org, is a more strict way of locking to a specific gem version, and the feature should work mostly transparently in the same way that specific versions are locked currently.

# Guide-level explanation

Upon first upgrading to bundler 2.5.0, the checksum feature will be enabled.

First usage will fall into two use cases:
1. I want to immediately use the new checksums feature.
2. I’m not specifically using the checksums feature, but expect bundler to be secure automatically.

For users excited about the new feature, the usual commands and docs should show the new options for interacting with checksums. Docs and the release notice should include the most direct way to fully use the checksums.

For users that are specifically looking for the feature, common commands like `bundle install`, `bundle update`, `bundle lock`, `bundle add`, should automatically make use of checksums when available. When there are missing checksums that must be fetched with a specific interaction, the UI should say what that action is.

Example:

```
$ bundle install
Bundle complete! 88 Gemfile dependencies, 256 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
256 gems are missing checksums. Source integrity could not be verified.
Use `bundle lock` to add checksums to Gemfile.lock.
```

When a gem version is updated in the lockfile, the checksum should be recorded. Running `bundle update` or `bundle add` should record the checksum from the source into the Gemfile.lock. If a checksum is not available from the source because the source does not provide such info, then a checksum should be created during install, or a warning should be printed.

If a user wishes to ensure that the environment they are using contains only gems installed from verified sources that match the checksums in the Gemfile.lock, the command `bundle pristine` should be run. The `bundle pristine` command installs every gem fresh from the .gem source, but does not fetch gems from the server or read the source index. `pristine` should be performed after checksums are written using the above commands.

Example:

```
$ bundle pristine
Installing rake 13.0.6
…
Installing rspec 3.12.0
42 gems are missing checksums. Source integrity could not be verified.
Use `bundle lock` to add checksums to Gemfile.lock.
$ bundle lock --update
Writing lockfile to path/to/Gemfile.lock
$ bundle pristine
Installing rake 13.0.6 (verified)
…
Installing rspec 3.12.0 (verified)
$
```

If a gem that is about to be installed does not match the checksum, an error is generated.

Example:

```
$ bundle install
Installing rake 13.0.6
Bundler cannot continue installing rake-13.0.6.
The checksum for the downloaded `rake-13.0.6.gem` does not match the known checksum for the gem. 

Expected: sha256-814828c34f1315d7e7b7e8295184577cc4e969bad6156ac069d02d63f58d82e8 from: Gemfile.lock:21:1 CHECKSUMS rake (13.0.6)
Downloaded: sha256-69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b69b6 from:
path/to/rake-13.0.6.gem SHA256 hexdigest

To resolve this issue:
Delete the downloaded gem files referenced above and run `bundle install`

If you are sure that the downloaded gem is correct, you can run…
(RFC TODO: Currently undecided on what the command should be here to allow unverified gems.)
`bundle install --skip-verification` OR
`bundle config set unverified ignore` OR
Remove the offending line from CHECKSUMS in Gemfile.lock
```

Certain checksums will always be unavailable because the source does not provide a checksum. When a checksum is not available, a line can be added to the Gemfile.lock CHECKSUMS section that explicitly indicates that no checksum is recorded for that gem version. Future installs of gems with intentionally ignored checksums should not error or warn when the gem is installed. The (RFC TODO) command will allow writing ignored checksum versions. Alternatively, the SHA256 digest of the gem at install time can be recorded, which will allow the user to verify future installs do not deviate from the recorded checksum.

Platforms may pose a problem for Gemfile.lock checksums. If a user has not added one of the platforms where they bundle their app, then checksum verification could be skipped. In order to avoid this security vulnerability, the following should happen automatically.

1. When a gem is installed and no checksum is available in the lockfile, a warning should be printed indicating that unverified gems were installed.
2. When a gem is installed where a checksum is available for the same gem on a different platform, an error should be raised. (i.e. if nokogiri for darwin has a checksum, but nokogiri for musl does not, then an error should be raised).
3. When bundle install is configured with frozen, any gem installed that does not have a checksum (and it was not explicitly ignored) should cause an error.

When platform or missing checksum issues are raised, the error should always indicate what command should be run to fix the error.

Bundling the app in CI should highlight any platform issues if the CI environment matches the production environment. This will make the bundler upgrade progress more smoothly for users.

To encourage uptake of this security feature, it should be enabled by default and any actions that have already fetched checksums should record them in the Gemfile.lock. Progressive upgrading to verified checksum behavior should happen naturally unless explicitly disabled.

Gem upgrades in the Gemfile.lock happen in two ways currently. Each way will behave differently.

1. The user explicitly runs `bundle update`.
2. Dependabot, or another automation, changes the version of a locked gem directly in the Gemfile.lock (assuming that dependabot is not upgraded to also change the checksum)

Explicit gem updates will record the new checksum for the gem and remove any old checksums for gems that are no longer in the Gemfile.lock.

Direct updates will print a warning during install that X number of gem(s) could not be verified. If the bundle is frozen, then the warning will instead print an error and halt gem installation.

# Reference-level explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks


Excessive failures

If checksum verification failures happen more often than expected, it could cause the feature to be ignored or derided as poorly designed and implemented. Bundler should progressively update the Gemfile.lock transparently without too much interaction or  excessive warnings and failures. Since checksums verification is just a more strict approach to version verification, this feature fits with the existing expectations of Bundler’s features. This means that checksum verification should be unobtrusive. The feature is not a big deal that needs lots of messaging, and should not fail unless bundler is configured to be strict (in the case of missing checksums) or there is an actual verification failure (in the case of corrupt or malicious gems)

Increased installation time

Gems will be SHA256 digested during install, adding to the install time. This could be a significant burden for some bundles. Future proposals such as content addressable storage for gem data could address this slowdown, but this is not immediately realistic.

Unverifiable gems

When gems come from private gem servers that do not implement the compact index, checksums will not be available. Bundler must provide configuration that can automatically store digests for gems from sources that don’t supply checksums so that the feature is not a burden to users relying on these sources. An option could be created to configure a specified source to always compute checksums for updated versions. This would mean any change in version from the source would be trusted at the time, but any locked version that is installed later would be verified to match the first install.

GitHub Dependabot version upgrades are at risk of failing.

If a user configures CI to use locked gem versions, it would be necessary for Dependabot to write the corresponding checksum line to the Gemfile to prevent every CI build from failing. This could cause many users to disable the feature because it interferes with an existing workflow or causes excessive CI failure notices. This can be addressed by making clear messaging about how to fix this problem.

# Rationale and Alternatives

- Rubygems.org already stores SHA256 checksums for gems and returns them in the compact index response. All the information is already present for checksum verification on the client side.
- Verifying .gem files at install time offers nearly complete protection against hacked, altered and corrupted gems. Bundler already ensures that only the bundled gems are available to the app. This feature adds the assurance that only verified gems were installed with the bundle.
- One alternative to this solution is including (vendoring) the bundled gems in the repository. This effectively has the same result, since the gems that will be installed during the production deploy will be verifiably the same gems that were used during CI and development. The downside of this vendored approach is the increase in the repository size. Checksums allow for a similar level of confidence without the downsides of vendoring gems and can be enabled by default so that more users will benefit.


# Unresolved questions

- What are the exact commands, options, warning and error messages that are used to interact with the feature?
- What are the unexpected errors that may happen the first time people verify the gems they are downloading? Are there any errors or problems in the existing rubygems.org repository?
