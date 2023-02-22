- Feature Name: Deprecating the RubyGems.org dependency API
- Start Date: 2023-01-27
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Plan for the deprecation and removal of the "dependency API" (`/api/v1/dependencies`) on rubygems.org.

# Motivation

The dependency API was the primary way Bundler fetched dependency info for Gemfile resolution until the release of [Bundler 1.12](https://bundler.io/blog/2016/04/28/the-new-index-format-fastly-and-bundler-1-12.html) in April 2016, which introduced the ["new" Compact Index API](https://andre.arko.net/2014/03/28/the-new-rubygems-index-format/).

The Compact Index was designed to be completely cacheable and served at the edge by Fastly. In contrast, the dependency API is completely _uncacheable_, and requires RubyGems.org to run a complex database query for each request, accounting for almost 50% of the total uncached request volume to RubyGems.org with nearly 3000qps.

Because the Compact Index has been out for over half a decade and Bundler & RubyGems both support fallback to the "full" index (`specs.4.8.gz` files) when the Dependency API is unavailable, we plan to deprecate and delete the dependency API over the coming months.

# Guide-level explanation

This will be a multi-phase transition.

1. Post on the RubyGems.org blog with a link to this RFC and this timeline
2. Implement a "brownout" functionality for the API on RubyGems.org that will 404 all requests to the API, with a link to (1) in the response body
3. Brownout the API as follows:
   a. 30 days after (1) for 5 minutes
   b. 37 days after (1) for 10 minutes, at the top of every hour
   c. 44 days after (1) for the day
4. Remove the API 51 days after (1)

# Drawbacks

It is possible that there are other users of the dependency API besides RubyGems and Bundler that lack fallback behavior. According to Honeycomb, non-(Bundler, RubyGems, Artifactory, Nexus) user agents make up approximately 0.34% ([6 million](https://ui.honeycomb.io/ruby-together/datasets/rubygems.org/result/6ESCRMf3r7K?useStackedGraphs) / [1.781 billion](https://ui.honeycomb.io/ruby-together/datasets/rubygems.org/result/ekuxbDWbAq9?useStackedGraphs)) of requests over the past 7 days.

We plan to ease the transition by announcing the deprecation and folllowing the above schedule for planned & widely announced brownouts of the API, to attempt to give those other consumers time to migrate to other, supported RubyGems.org APIs.

# Rationale and Alternatives

Leaving the dependency API in place increases the load on the RubyGems.org production service, requiring both more resources to host and more maintainer attention to keep running.

For non-dependency resolution usage of the API, we advise users transition to the compact index API or the gem versions API, whichever is more appropriate for their use cases.
