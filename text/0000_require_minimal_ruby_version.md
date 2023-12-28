- Feature Name: mandatory_required_ruby_version
- Start Date: 2020-04-13
- RFC PR: 
- Bundler Issue: 

# Summary

Calling `spec.required_ruby_version` in a gemspec should be mandatory.

# Motivation

My current problem is that many gems are not publishing their `required_ruby_version`.

This means that a Gemfile/gemspec must know what is the maximum version of a gem for a given version of Ruby, instead of bundler figuring it out for us.

Assuming the author has thought of what version of Ruby is the target, it takes about one minute to add the right line. If the author has not thought of that, this important concern will now be in focus.

Currently an author may not specify it at all and everyone that uses the gem, directly or indirectly, may have to figure out the maximum version and specify it in their own gemfiles.

Note that there is currently no good way to correct a forgotten `required_ruby_version`. Once the gem is published, the only option [is to yank the gem](https://github.com/rubygems/rubygems/issues/1506#issuecomment-188472423) which is typically not what one wants to do.

`required_ruby_version` is currently in the "Optional" section of [the spec](https://guides.rubygems.org/specification-reference/). It's not even in the "Recommended" section!

It should be mandatory and enforced.

# Guide-level explanation

Building a gem that does not specify it would fail, same as the other required fields.

# Reference-level explanation

Building a gem that does not specify it would fail, same as the other required fields.

# Drawbacks

I can not think of a single valid reason not to specify a required ruby version.

Gem authors that refuse to decide on a particular version can always decide to circumvent the check by calling `spec.required_ruby_version = '> 0'`

Potential impact is confusion/frustration of the author upgrading their rubygems and now their gem doesn't build. The error message would hopefully contain enough information for the fix to be quickly done.
