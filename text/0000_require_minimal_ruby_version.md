- Feature Name: (fill me in with a unique ident, my\_awesome\_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Calling `spec.required_ruby_version` in a gemspec should be mandatory.

# Motivation

My current problem is that many gems are not publishing their `required_ruby_version`.

This means that a Gemfile/gemspec must know what is the maximum version of a gem for a given version of Ruby, instead of bundler figuring it out for us.

It takes about one minute for the gem author to add the line. When the author doesn't, everyone that uses the gem, directly or indirectly, may have to figure out the maximum version and specify it in their own gemfiles.

Note that there is currently no good way to correct this mistake. Once it is done, the only option [is to yank the gem](https://github.com/rubygems/rubygems/issues/1506#issuecomment-188472423) which is typically not what one wants to do.

`required_ruby_version` is currently in the "Optional" section of [the spec](https://guides.rubygems.org/specification-reference/). It's not even in the "Recommended" section!

It should be mandatory and enforced.

# Guide-level explanation

Building a gem that does not specify it would fail, same as the other required fields.

# Reference-level explanation

Building a gem that does not specify it would fail, same as the other required fields.

# Drawbacks

I can not think of a single valid reason not to specify a required ruby version.

Gems that don't want to specify one can simply call `spec.required_ruby_version = '> 0'`

Only impact is potential confusion of user upgrading their rubygems and now their gem doesn't build, but error message should be obvious enough to fix in a few seconds.
