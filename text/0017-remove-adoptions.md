- Feature Name: remove\_adoptions
- Start Date: 2024-11-22
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Remove the Adoptions feature from RubyGems.org due to its high maintenance burden and limited benefit.

# Motivation

Adoptions aimed to address the lack of maintenance in older Ruby gems but has proven unsuccessful.

Implementing Pundit authorization and Organizations features for RubyGems.org highlighted this issue. About 50% of my authorization efforts were spent handling the complexity of adoptions.

However, the feature's low effectiveness does not justify this effort.

- 45 "Ownership Calls" (requests for new maintainers) created.
- 140 "Ownership Requests" submitted:
  - 40 tied to Ownership Calls, with only 6 approved.
  - 100 unsolicited, with 63 ignored, 8 approved, and 29 denied.

Only 14 successful ownership transfers occurred, reflecting a mismatch between the low-trust adoption process and the high-trust requirements of gem ownership.

Its low-trust, unmoderated nature risks compromising critical software. Standard ownership transfers, supported by command-line and web tools, are easier, more robust, and allow finer-grained control with the Maintainer Role on forthcoming Organizations.

Given its low usage, high maintenance burden, and incompatibility with RubyGems.org's goals, I propose fully removing Adoptions.

# Guide-level explanation

The impact of removing Adoptions can be shown by comparing two scenarios:

### With Adoptions

Adoptions facilitate low-trust gem transfers:

1. Gem owners create a "call" seeking a new maintainer.
2. Interested parties submit requests.
3. The owner reviews and approves a request, transferring ownership.

Unsolicited requests are allowed for some gems but the requirements limit it to the least used gems.

### Without Adoptions

Without Adoptions, ownership transfers rely on direct communication and trust:

1. Encounter an unmaintained gem of interest.
2. Open an issue on the gem's repository (e.g., GitHub).
3. Communicate with the owner to build trust and arrange access:
   - Ownership may be granted with conditions (e.g., maintainer access only) or unconditionally.
   - Owners can retain control through an organization, add maintainers, or transfer full ownership.

Owners seeking maintainers can share their needs in the README or on the repository, facilitating necessary conversations.

# Reference-level explanation

To implement this change, remove code and update guides. Replace Adoptions guides with recommendations for direct communication and startandard ownership processes.

# Drawbacks

Adoptions have enabled a few successful transfers. While these transfers seem unproblematic, their low frequency, ready alternatives, and slight added risk do not justify retaining the feature.

# Rationale and Alternatives

Adoptions face inherent conflicts:
1. Popular gems attract attention but are unlikely to permit unsolicited transfers.
2. Standard ownership processes already handle transfers negotiated outside RubyGems.org.
3. Low-trust adoption processes conflict with the high trust needed for critical software.
4. The system replaces personal conversations with rigid workflows, contrary to desired transparency and collaboration.

### Why not expand eligible gems?

Expanding eligibility increases security risks and spam. Those seeking low-trust ownership transfer may be least trustworthy.

### Why not make adoptions public?

Publicizing adoptions is unlikely to increase engagement.

### Why not support maintainer roles during adoption?

Communicating outside the adoption system and using existing tools achieves this more readily and requires no additional development.

## Alternatives

For unmaintained gems with unreachable owners, RubyGems.org staff can facilitate communication and ownership decisions.

Gem owners can indicate adoption availability in READMEs or other channels, using standard RubyGems.org tools to handle transfers. Maintainers can offer help via repository-hosted communication tools.

# Unresolved questions

How do we maintain abandoned but necessary packages?

Building a sustainable maintenance community remains challenging. Unfortunately, the adoption system has not proven to be the solution.
