- Feature Name: `gem_checksum_verification`
- Start Date: 2023-08-29
- RFC PR: https://github.com/rubygems/rubygems/pull/6374
- Bundler Issue: (leave this empty)

# Summary

Bundler is adding checksum verification of gems when they are installed. It should be secure by default and easy to use. It should not break assumptions or unnecessarily break deployment or CI unless there is a security problem.

# Motivation

Verifying gem checksums with known checksums at install time is a stronger way to verify that the exact same gem source is being used in every environment where the bundle is installed.
Using checksums, sourced from rubygems.org or from digests of local .gem file, is a more strict way of locking to a specific gem version.
The feature should work transparently in much the same way that bundler already locks to specific versions by ensuring that not just the same version, but the exact same rubygem file data, is installed on each environment.

# Guide-level explanation

Upon first upgrading to this version of Bundler, expect bundler to progressively add checksums to the lockfile without any extra configuration.

Common Bundler commands like `bundle install`, `bundle update`, `bundle lock`, `bundle add` will now automatically record and verify checksums.
Commands that rely on checksums for verification will silently succeed when checksums match and fail with a unique non-zero exit code when checksums do not match.

If you wish to immediately add all available checksums to your lockfile for your bundled gems, run `bundle lock`.
Bundle lock now fetches checksums from remote sources by default.
If you would like to bypass this behavior, run `bundle lock --no-checksums`.

Example:

```
$ bundle install
Bundle complete! 88 Gemfile dependencies, 256 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
```

Running `bundle update` or `bundle add` will record the checksum from the source (e.g. rubygems.org) into the Gemfile.lock, if it is available.
If a checksum is not available from the source because the source does not provide such info (e.g. private gemservers) then a checksum will be created during install using the .gem file.
If a checksum can't be created because the source is a path or git source, then only the gems name and version will be recorded with a blank checksum.

If you want to ensure that the bundled environment contains only gems matching the checksums in the lockfile, run `bundle pristine`.
The `bundle pristine` command installs every gem fresh from a newly downloaded .gem source.
The pristine install will trigger computation and comparison of the generated SHA256 checksum with the checksum stored in the lockfile.

Example:

```
$ bundle pristine
Installing rake 13.0.6
…
Installing rspec 3.12.0
42 gems without checksums.
Use `bundle lock` to add checksums to Gemfile.lock.
$ bundle lock
Writing lockfile to path/to/Gemfile.lock
$ bundle pristine
Installing rake 13.0.6
…
Installing rspec 3.12.0
$
```

When bundler installs a gem from a `.gem` file, it computes the SHA256 checksum of the file.
If an existing checksum is available from the lockfile or the remote source, it will be compared with the computed checksum at install time.
If the checksums do not match, an error is generated and installation is halted.
If no checksum is recorded in the lockfile, the computed checksum is saved to the lockfile where future installs can verify that the same gem is installed.

Example:

```
$ bundle install
Installing rake 13.0.6
Bundler found mismatched checksums. This is a potential security risk.
  rake (13.0.6) sha256=2222222222222222222222222222222222222222222222222222222222222222
    form the lockfile CHECKSUMS at Gemfile.lock:21:17
  rake (13.0.6) sha256=814828c34f1315d7e7b7e8295184577cc4e969bad6156ac069d02d63f58d82e8
    from the gem at path/to/rake-13.0.6.gem

  To resolve this issue you can either:
    1. remove the gem at path/to/rake-13.0.6.gem
    2. run `bundle install`
  or if you are sure that the new checksum from the gem at path/to/rake-13.0.6 is correct:
    1. remove the matching checksum in Gemfile.lock:21:17
    2. run `bundle install`

To ignore checksum security warnings, disable checksum validation with
  `bundle config set --local disable_checksum_validation true`
```

Certain checksums will always be unavailable because the source does not provide a checksum.
When a checksum is not available, the gem will be added to the Gemfile.lock CHECKSUMS section without a checksum.

Users should be aware that when bundling on CI or production, new gems can be added for a platform not found in the Gemfile.lock.
Bundler will silently record the new checksums for the missing gem just like on a local development machine.
If you would like to ensure that only lockfile checksums are used, bundle install should use use the frozen or deployment configurations.

# Reference-level explanation

Gem checksums are fetched from rubygems.org as part of the compact index.
The checksums are stored in memory during the bundle process.

If any gems are recorded from a separate source or installed via .gem file, another checksum is recorded and compared with the original.

If any of the checksums for the same gem name, version and platform differ, an error is raised with instructions for resolving the error.

When the Gemfile.lock is written, a new CHECKSUMS section is written with all the gems in the bundle and their corresponding checksums.

Future compatibility for new checksum algorithms is supported by reading and writing existing checksums for installed gems even if the algorithm is unknown.
Checksum comparisons take into account the algorithm used and raises errors accordingly.

Bundler commands that interact with checksums either fetch checksums from sources and update the Gemfile.lock or compare checksums in the Gemfile.lock with the computed digest of the gem being installed.

Internal storage of checksums indexes the checksums by a gem's NameTuple (name, version, platform) and the checksum's algorithm.
The source of each checksum is stored with the checksum so that errors can describe how to fix conflicting checksums.

When two remote sources have a checksum for the same gem, they are compared.
If they are the same, bundler proceeds as normal

### Example CHECKSUM section

A freshly created Rails 7.0.7.2 app creates the following CHECKSUMS section (snipped for brevity):

```
CHECKSUMS
  actioncable (7.0.7.2) sha256=a921830a59ee314939955c9fc3b922d2b1f3ebc16fdf062370b9078aa0dc28c5
  actionmailbox (7.0.7.2) sha256=33aeae209fc876c072e5ad28c7ffc16ace533d391368ad6390bb6183c2b27a24
  actionmailer (7.0.7.2) sha256=0e9061159af8c220042b7714a2ba01e2d71d2904f308021ec714793e5f9811a0
  actionpack (7.0.7.2) sha256=c441ff3898bf5827540bcab929d2f5be6e75b64c101513629a3c88e269615561
  actiontext (7.0.7.2) sha256=d29eabbfbf0f084a0bddcfc6bd7e6245e209ec3a1def200e95b670e0cdfba033
  actionview (7.0.7.2) sha256=15ba2612efb484ec80d5b656b4ea16e02d34d3f9980cabc13bd8ac15ccea3f94
  activejob (7.0.7.2) sha256=6d8ebd81d29ce65bb57830640fa2d3f01e4cab0d71714a54c2b13763021023a4
  activemodel (7.0.7.2) sha256=45ba827986065ac273b59cb3b6c9ab3da412beca5d465f1acf7a51fb5bc032b3
  activerecord (7.0.7.2) sha256=425f84edb279c02fe2195eee166b20aabb36f51939087d040fa462859bd6790f
  activestorage (7.0.7.2) sha256=8f1d79266f148d74e1cc7fcc91f3f04171e0d10c68f8a31ac95d11644114f4f0
  activesupport (7.0.7.2) sha256=62e01393689c8514a65e2cf8be6f4781d1e6c7d9adc25b1056902d8abd659fee
  addressable (2.8.5) sha256=63f0fbcde42edf116d6da98a9437f19dd1692152f1efa3fcc4741e443c772117
  # ... SNIP ...
  xpath (3.2.0) sha256=6dfda79d91bb3b949b947ecc5919f042ef2f399b904013eb3ef6d20dd3a4082e
  zeitwerk (2.6.11) sha256=ade72f223a75c91f3b02b2c941a57fb697bc443d615f38c28773185e08698dd7
```

During the `rails new` command, `bundle install` pulled all the checksums from the compact index on rubygems.org, then computed checksums for each gem as it was installed.

# Drawbacks

### Excessive failures

If checksum verification failures happen more often than expected, it could cause the feature to be ignored or derided as poorly designed and implemented.
Bundler should progressively update the Gemfile.lock transparently without too much interaction or excessive warnings and failures.
Checksums verification is just a more strict approach to version verification, so this feature fits with the existing expectations of Bundler’s features and should be unobtrusive.
The feature is not a "big deal" that needs lots of warnings or errors to encourage usage.
It should not fail unless bundler is configured to be strict (a frozen bundle) or there is an actual verification failure (a corrupt or malicious gem).

### Increased installation time

Gems will be SHA256 digested during install, slightly adding to the install time.
This could be a larger burden for some bundles on slower machines.

Future proposals could attempt to address this slowdown, if necessary.

### Unverifiable gems

When gems come from private gem servers that do not implement the compact index, checksums will not be available.
Bundler will calculate digests from .gem files from sources that don’t supply checksums.

An additional flag for `bundle lock` could be provided that allows reinstalling, and thus calculating the checksum for, gems that don't have a checksum.

Gems from a git source are verified by nature of matching a specific git SHA and should be excluded from checksum verification.

Path source gems cannot be verified because the checksum of the entire path would be complicated to calculate and unreliable.

### GitHub Dependabot and other automations are at risk of failing.

In the normal case with a non-frozen bundle, Dependabot will continue to work as normal.
Dependabot will not immediately add checksums to Dependabot PRs, but this is not an error.
The next user install after a dependabot commit will add the checksum.

However, if CI uses a frozen bundle, then all dependabot pull requests will fail due to missing checksums.
This should be addressed by printing clear messaging about how to fix this problem: checkout the branch and run `bundle install`.

Until patched, dependabot update branches will be less useful because they will require manual intervention before the build will work.
This might lead users to disable frozen bundles or disable the checksums feature to avoid the CI failures and continue with their existing workflow.
In this case, leaving these features disabled, maybe for longer than necessary, could impact the security of these user's application.

Dependabot maintainers will need to update Dependabot to write the corresponding checksum to the Gemfile to prevent CI build failures caused by frozen bundle missing checksums.
This makes Dependabot the defacto source of the checksums in the Gemfile for updated gems, which is hopefully already clear to users merging these PRs and should be considered part of the trust extended to GitHub already.

We are open to working with the dependabot team to provide the information necessary to add checksums to the lockfile.

### Older versions of Bundler

Old versions of Bundler should ignore the CHECKSUMS section.

# Rationale and Alternatives

Rubygems.org already stores SHA256 checksums for gems and returns them in the compact index response.
All the information is already present for checksum verification on the client side.

Verifying .gem files at install time offers nearly complete protection against hacked, altered and corrupted gems.
Bundler already ensures that only the bundled gems are available to the app.
This feature adds the assurance that only verified gems were installed with the bundle.

One alternative to this solution is including (vendoring) the bundled gems in the repository.
This effectively has the same result, since the gems that will be installed during the production deploy will be verifiably the same gems that were used during CI and development.
The downside of this vendored approach is the increase in the repository size.
The upside is installation that doesn't depend on a remote source.
Checksums allow for a similar level of confidence without the larger repository size of vendoring gems.
It can be enabled by default so that more users will benefit.

# Unresolved questions

### What are the unexpected errors that may happen the first time developers interact with this feature?

In particular, the first deploy for users with a frozen bundle in CI and/or production may produce errors that might not have obvious solutions to someone unfamiliar with the feature.

### How do we handle confusion about the authority of checksums written to the Gemfile.lock

The source of checksums in the Gemfile.lock becomes a matter of trust once it's written.
Did the checksum come from the API or was it calculated from a .gem file on a developers computer.
If a checksum error is resolved by one developer in a way that saves an incorrect checksum, how should people know when to approve these changes or not.
It may not be common practice for most teams to look at the Gemfile.lock when reviewing code.
Gemfile.lock changes can be hidden in pull request reviews.
Without a process for checking that the checksums are trustworthy, it's left to every development team to decide on a process.

One solution would be a bundle command that could be run in CI every time the gems are installed that verifies the authenticity of checksums in the Gemfile.lock.
