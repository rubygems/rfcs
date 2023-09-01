- Feature Name: `gem_checksum_verification`
- Start Date: 2023-08-29
- RFC PR: https://github.com/rubygems/rubygems/pull/6374
- Bundler Issue: (leave this empty)

# Summary

Bundler is adding checksum verification of gems when they are installed. It should be secure by default and easy to use. It should not break assumptions or unnecessarily break deployment or CI unless there is a security problem.

# Motivation

Verifying gem checksums with known checksums at install time is a stronger way to verify that the exact same gem source is being used in every environment where the bundle is installed. Using checksums, especially those sourced from rubygems.org, is a more strict way of locking to a specific gem version. The feature should work transparently in much the same way that bundler already locks to specific versions.

# Guide-level explanation

Upon first upgrading to the version of Bundler with this feature, the checksum feature will be enabled.

First usage will fall into two use cases:
1. Users that want to immediately use the new checksums feature.
2. Users that are not specifically using the checksums feature, but expect bundler to work securely without interference.

For users excited about the new feature, the usual commands and docs should show the new options for securing the bundle with checksums. Documentation and the release notice should include instructions for directly using the new feature.

For users that are not specifically looking for the feature, common commands like `bundle install`, `bundle update`, `bundle lock`, `bundle add`, should automatically make use of checksums when available. When there are missing checksums that must be fetched with a specific interaction, the UI should say what that action is.

Example:

```
$ bundle install
Bundle complete! 88 Gemfile dependencies, 256 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
256 gems are missing checksums. Source integrity could not be verified.
Use `bundle lock` to add checksums to Gemfile.lock.
```

When a gem version is updated in the lockfile using a source that has checksums (like rubygems.org), the checksum should be recorded. Running `bundle update` or `bundle add` should record the checksum from the source into the Gemfile.lock. If a checksum is not available from the source because the source does not provide such info, then a checksum should be created during install using the .gem file or a warning should be printed.

If a user wishes to ensure that the environment they are using contains only gems installed from verified sources that match the checksums in the Gemfile.lock, the command `bundle pristine` should be run. The `bundle pristine` command installs every gem fresh from the .gem source, but does not fetch gems from the server or read the source index. Therefore, if a users wants all gems to be verified upon install, `bundle pristine` should be run after checksums are written to the Gemfile.lock.

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

Certain checksums will always be unavailable because the source does not provide a checksum. When a checksum is not available, a line can be added to the Gemfile.lock CHECKSUMS section that explicitly indicates that no checksum is recorded for that gem version. Future installs of gems with intentionally ignored checksums should not error or warn when the gem is installed. The **(RFC TODO)** command will allow idicating that a gem version should ignore checksum verifications. Alternatively, the SHA256 digest of the gem at install time can be recorded allowing the user to verify future installs do not deviate from the recorded checksum.

Platforms may pose a problem for Gemfile.lock checksums. If a user has not added one of the platforms where they bundle their app, then checksum verification could be skipped. In order to avoid this security vulnerability, the following should happen automatically.

1. When a gem is installed and no checksum is available in the lockfile, a warning should be printed indicating that unverified gems were installed.
2. When a gem is installed where a checksum is available for the same gem on a different platform, an error should be raised. (i.e. if nokogiri for darwin has a checksum, but nokogiri for musl does not, then an error should be raised).
3. When bundle install is configured with frozen, an error should be raised for any gem installed that does not have a checksum unless the checksum is explicitly ignored.

When platform or missing checksum issues are raised, the error should always indicate what command or process should be performed to fix the error.

Bundling the app on another platform, in CI for example, should highlight any platform issues. This will make the bundler upgrade easier for users.

To encourage uptake of this security feature, it should be enabled by default. Any actions that fetch authoritative checksums should silently record them in the Gemfile.lock. Progressive upgrading to verified checksum behavior should happen naturally unless explicitly disabled.

Gem upgrades in the Gemfile.lock happen in two ways currently. Each way will behave differently.

1. The user explicitly runs `bundle update`.
2. Dependabot, or another automation, changes the version of a locked gem directly in the Gemfile.lock (assuming current Dependabot behavior without a corresponding upgrade to match this feature).

Explicit gem updates will record the new checksum for the gem and remove any old checksums for gems that are no longer in the Gemfile.lock.

Direct updates to the Gemfile.lock will print a warning during install. The warning should say that X number of gem(s) could not be verified.
If the bundle is frozen, then the warning should instead print an error and halt gem installation.

# Reference-level explanation

Gem checksums are fetched from rubygems.org as part of the compact index. The checksums are stored in memory during the bundle process.

If any gems are recorded from a separate source or installed via .gem file, another checksum is recorded and compared with the original.

If any of the checksums for the same gem name, version and platform differ, an error is raised.

When the Gemfile.lock is written, a new CHECKSUMS section is written with all the gems in the bundle and their corresponding checksums.

Future compatibility for new checksum algorithms is supported by reading and writing existing checksums for installed gems even if the algorithm is unknown. Checksum comparisons take into account the algorithm used and raises errors accordingly.

Bundler commands that interact with checksums either fetch checksums from sources and update the Gemfile.lock or compare checksums in the Gemfile.lock with the computed digest of the gem being installed.

Internal storage of checksums indexes the checksums by a gem's full name (name-version-platform) and the checksum's algorithm. The source of each checksum is stored with the checksum so that errors can detail where conflicting checksums came from.

When two sources have a checksum for the same gem full name, the are compared. If they are the same, they are merged into a single record that associates the checksum with all of the sources for that checksum.

### Example CHECKSUM section

A freshly created Rails 7.0.7.2 app creates the following CHECKSUMS section (snipped for brevity):

```
CHECKSUMS
  actioncable (7.0.7.2) sha256-a921830a59ee314939955c9fc3b922d2b1f3ebc16fdf062370b9078aa0dc28c5
  actionmailbox (7.0.7.2) sha256-33aeae209fc876c072e5ad28c7ffc16ace533d391368ad6390bb6183c2b27a24
  actionmailer (7.0.7.2) sha256-0e9061159af8c220042b7714a2ba01e2d71d2904f308021ec714793e5f9811a0
  actionpack (7.0.7.2) sha256-c441ff3898bf5827540bcab929d2f5be6e75b64c101513629a3c88e269615561
  actiontext (7.0.7.2) sha256-d29eabbfbf0f084a0bddcfc6bd7e6245e209ec3a1def200e95b670e0cdfba033
  actionview (7.0.7.2) sha256-15ba2612efb484ec80d5b656b4ea16e02d34d3f9980cabc13bd8ac15ccea3f94
  activejob (7.0.7.2) sha256-6d8ebd81d29ce65bb57830640fa2d3f01e4cab0d71714a54c2b13763021023a4
  activemodel (7.0.7.2) sha256-45ba827986065ac273b59cb3b6c9ab3da412beca5d465f1acf7a51fb5bc032b3
  activerecord (7.0.7.2) sha256-425f84edb279c02fe2195eee166b20aabb36f51939087d040fa462859bd6790f
  activestorage (7.0.7.2) sha256-8f1d79266f148d74e1cc7fcc91f3f04171e0d10c68f8a31ac95d11644114f4f0
  activesupport (7.0.7.2) sha256-62e01393689c8514a65e2cf8be6f4781d1e6c7d9adc25b1056902d8abd659fee
  addressable (2.8.5) sha256-63f0fbcde42edf116d6da98a9437f19dd1692152f1efa3fcc4741e443c772117
  # ... SNIP ...
  xpath (3.2.0) sha256-6dfda79d91bb3b949b947ecc5919f042ef2f399b904013eb3ef6d20dd3a4082e
  zeitwerk (2.6.11) sha256-ade72f223a75c91f3b02b2c941a57fb697bc443d615f38c28773185e08698dd7
```

During the `rails new` command, `bundle install` pulled all the checksums from the compact index on rubygems.org, then computed checksums for each gem as it was installed.

# Drawbacks

### Excessive failures

If checksum verification failures happen more often than expected, it could cause the feature to be ignored or derided as poorly designed and implemented. Bundler should progressively update the Gemfile.lock transparently without too much interaction or  excessive warnings and failures. Since checksums verification is just a more strict approach to version verification, this feature fits with the existing expectations of Bundler’s features. This means that checksum verification should be unobtrusive. The feature is not a big deal that needs lots of messaging, and should not fail unless bundler is configured to be strict (in the case of missing checksums) or there is an actual verification failure (in the case of corrupt or malicious gems)

### Increased installation time

Gems will be SHA256 digested during install, adding to the install time. This could be a significant burden for some bundles. Future proposals such as content addressable storage for gem data could address this slowdown, but this is not immediately realistic.

### Unverifiable gems

When gems come from private gem servers that do not implement the compact index, checksums will not be available. Bundler must provide configuration that can automatically store digests for gems from sources that don’t supply checksums so that the feature is not a burden to users relying on these sources. An option could be created to configure a specified source to always compute checksums for updated versions. This would mean any change in version from the source would be trusted at the time, but any locked version that is installed later would be verified to match the first install.

### GitHub Dependabot version upgrades are at risk of failing.

If a user configures CI to use locked gem versions, it would be necessary for Dependabot to write the corresponding checksum line to the Gemfile to prevent every CI build from failing. This could cause many users to disable the feature because it interferes with an existing workflow or causes excessive CI failure notices. This can be addressed by making clear messaging about how to fix this problem.

### Older versions of Bundler

Old versions of Bundler should ignore the CHECKSUMS section. We will need to check older versions to be sure.

# Rationale and Alternatives

- Rubygems.org already stores SHA256 checksums for gems and returns them in the compact index response. All the information is already present for checksum verification on the client side.
- Verifying .gem files at install time offers nearly complete protection against hacked, altered and corrupted gems. Bundler already ensures that only the bundled gems are available to the app. This feature adds the assurance that only verified gems were installed with the bundle.
- One alternative to this solution is including (vendoring) the bundled gems in the repository. This effectively has the same result, since the gems that will be installed during the production deploy will be verifiably the same gems that were used during CI and development. The downside of this vendored approach is the increase in the repository size. Checksums allow for a similar level of confidence without the downsides of vendoring gems and can be enabled by default so that more users will benefit.

# Unresolved questions

- What are the exact commands, options, warning and error messages that are used to interact with the feature?
- What are the unexpected errors that may happen the first time people verify the gems they are downloading? Are there any errors or problems in the existing rubygems.org repository?

### How do we handle confusion about the authority of checksums written to the Gemfile.lock

The source of checksums in the Gemfile.lock becomes a matter of trust once it's written. Did the checksum come from the API or was it calculated from a .gem file on a developers computer. If a checksum error is resolved by one developer in a way that saves an incorrect checksum, how should people know when to approve these changes or not. It may not even be common practice for most teams to look at the Gemfile.lock, and changes can often be hidden in pull request reviews. Without a process for checking that the checksums are trustworthy, it's left to every development team to decide on a process. One solution would be a bundle command that could be run in CI every time the gems are installed that verifies the authenticity of checksums in the Gemfile.lock.
