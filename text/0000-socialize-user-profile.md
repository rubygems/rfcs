- Feature Name: Socialize user profile
- Start Date: 2023-09-30
- RFC PR:
- Bundler Issue:

# Summary

Enhance the RubyGems.org profile by allowing users to include a homepage URL and up to four social network links, inspired by GitHub's approach to profile customization.

# Motivation

The evolution and growth of the developer community underscore the importance of developers establishing a comprehensive online presence. Beyond just gem contributions, affiliations to other platforms can boost trust, promote collaborations, and enrich community interactions. This addition also sets the groundwork for future features, like leveraging these social links for enhanced verification mechanisms.

# Guide-level explanation

Within the settings of your RubyGems.org profile, you'll find options to:

- Add a **Homepage URL**: A space for your personal blog, portfolio, or any website you associate with your professional persona.
- Incorporate **Social Network Links**: Four fields where you can link to your social profiles. Recognized platforms (Twitter, Mastodon, LinkedIn, GitHub, GitLab) will display their distinct icons. Links from unrecognized platforms or generic URLs will showcase a universal link icon.

This enhancement transforms a user's RubyGems profile from merely a gems contribution summary to a broader representation of their professional and social spheres.

# Reference-level explanation

For implementation:

1. Add a new input field designated for the homepage URL.
2. Integrate four input fields specifically for social network links.
3. Initiate a validation mechanism on update:
   - Validate URLs for their authenticity and functionality.
   - Authenticate the URL structure of known platforms (Twitter, Mastodon, LinkedIn, GitHub, GitLab).
4. Display methodology:
   - Determine the domain of each provided link.
   - Display the appropriate icon beside each link. For unfamiliar URLs, use a default link icon.

# Drawbacks

**Verification Concerns**: The inclusion of external links raises concerns about users potentially linking to unsuitable or misleading content. This might necessitate mechanisms to oversee, report, or exclude harmful links.

# Rationale and Alternatives

- **Why this design?**: By mirroring GitHub's approach, this design offers a blend of personalization and clarity. Restricting it to one homepage and four other links ensures profile simplicity while allowing meaningful customization.
- **Alternatives considered**:
  - Allow an infinite number of links. (Could clutter profiles)
  - Limit to one or two social links. (Might be excessively restrictive)
- **Impact of not doing this**: Without this, RubyGems.org profiles risk being overly focused on gems only.

# Unresolved questions

- What mechanisms can we deploy to guarantee the appropriateness of external links? GitHub allows any link with no verification.
- How might these links factor into prospective verification processes in the future?
