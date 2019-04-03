- Feature Name: extended_gem_errors
- Start Date: April 3, 2019
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Bundler expresses gem installation issues as they are presented from the gems themselves. Gems, however, often integrate with system-level dependencies. When these fail, they give GCC dumps that are indecipherable to even some of the most experienced developers and absolute gibberish to new developers. This RFC intends to address this issue and take an opportunity to better educate Ruby developers while enabling them to work more productively.

# Motivation

Experiences and new developers, alike, are confused by GCC dumps from gems. Like this one:

![RMagick failures are cryptic and confusing](https://user-images.githubusercontent.com/3074765/55492679-9c61ef80-5605-11e9-83ec-184933916abc.png)

These kinds of errors often send users directly to Google and StackOverflow, searching for answers in the depths of obscurity. [This tweet](https://twitter.com/andrewmcodes/status/1111788054593835010) is a perfect example of a user having this issue with Nokogiri 1.10.2 and finding a year and a half old issue 3 pages into Google, in some unrelated project, that finally had an answer a year after the original post.

To address this, I created [Extended Bundler Errors](https://github.com/jules2689/extended_bundler-errors), a bundler plugin. However, the plugin system is not discoverable and is buggy (issue incoming for that). The main point here is that this is a feature that is useful to *most* developers and therefore would be good to include by default.

# Guide-level explanation

To use this feature, you would use Bundler as per normal. There are no changes to the end user except in the case we match error output against something we know about. The error handler should be considered a part of Bundler internals and you should know nothing about it.

In the case we do match, 2 things happen:

1. GCC output is saved to a file (in case you want to read the nitty gritty details)
2. The error is replaced with a JIT (just-in-time) education principled error including:
 - Title ("Gem [Version] could not be installed")
 - What went wrong (gem failed to install because of X)
 - Why it went wrong (you didn't have Y)
 - How to fix it (based on current OS architecture)
 - File path with full GCC output

This will provide the end user with a quick overview of what broke, expand why it broke, and tell them how to fix it. It will also include a file path to the full output in the case that is needed.

For reference, this is what the plugin did to the above RMagick error, and what I intend to implement in Bundler:

![A much better error for rmagick](https://user-images.githubusercontent.com/3074765/55492685-9e2bb300-5605-11e9-9e67-bb555cb2537c.png)

# Reference-level explanation

The plugin works in a fairly simple way. Where the `after-install` hook is triggered, we run this check:

```ruby
Bundler::Plugin.add_hook('after-install') do |spec_install|
  troubleshoot(spec_install) if spec_install.state != :installed
end
```

We can see this check will auto-troubleshoot any spec that is not installed correctly against a collection of handlers.

For context, here is an example of a handler: https://github.com/jules2689/extended_bundler-errors/blob/master/lib/extended_bundler/handlers/gda.yml

Troubleshoot happens like this:

```ruby
def troubleshoot(spec_install)
  # Determine a path in the plugin gem to a "handlers" folder, including the name of the gem
  # E.g. ~/path/to/gem/lib/handlers/nokogiri.yml
  path = handler_path(spec_install.name)
  return nil unless File.exist?(path)

  # If we have a handler, then load it
  # Each handler may contain multiple matchable errors, iterate and match
  # Each matchable error is based on the gem version and a pattern to match
  # If we match the gem version and the pattern is found in the original error message, then we extract out the build error and replace it. 
  troubleshooted = false
  yaml = YAML.load_file(path)
  yaml.each do |handler|
    next unless version_match?(spec_install.spec.version, handler['versions'])
    next unless handler['matching'].any? { |m| spec_install.error =~ Regexp.new(m) }
    spec_install.error = build_error(spec_install, handler)
    troubleshooted = true
  end

  # If we did not troubleshoot, and this was a native extension issue
  # then tell the user to file an issue to help us track down more bugs
  if !troubleshooted && spec_install.error.include?('Failed to build gem native extension')
    spec_install.error = spec_install.error + "\n\n" + NATIVE_EXTENSION_MSG
  end
end
```

In Bundler, we would largely do the same.

- We would have a collection of handlers which match on name, version, and pattern in the error.
- Instead of a plugin hook, we would just call at the same point that the plugin hook is called.

Potential improvements
---
- A new plugin hook could be added that allows plugins to register their own handlers
- Gemspecs could specify error handlers for itself. This would allow gems to debug their own installation issues.

# Drawbacks

1. We would need to field issues that include GCC dumps. We would then need to make handlers and include them in Bundler. If you don't update bundler, then you don't get new errors (unless we include the plugin hook and make errors updateable that way, or make some remote index).
2. It might be better to go deeper into the Ruby ecosystem and ultimately include this in Rubygems itself
3. It may obfuscate error messages a bit more and people won't ever deal with them, making the knowledge of GCC dumps even more niche

# Rationale and Alternatives

- This is a fairly simple integration point that is fairly easily removed if needed
- This maintains all current information (in the file) and provides quick and easy understanding for GCC errors by developers

### Alternatives

- An alternative would be to drastically improve the plugin system, but I suspect that will take a lot of time and this issue affects everyone - so why not make it core?
- An alternative would be Rubygems, but Rubygems responsibility is to install gems and get out of the way - not to debug and manage multiple levels of things.

### What is the impact of not doing this?

Developers will continue to waste time and we will continue to see errors being solved 3 pages deep in a 2 year old issue that took 1 year to answer.

# Unresolved questions

- Should we look at remote indexes for errors?
- Should we add the plugin point for new handlers to be added (I think yes).
