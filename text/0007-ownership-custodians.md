# Ownership Custodians

The health of <https://RubyGems.org> is important for the community and there are a large number of factors which decide this. Having a large number of essentially abandoned gems contributes negatively towards this and we would like to continue improving the mechanisms for dealing with this problem.

We propose to introduce gem custodians and a process for approving **ownership request**s where the original owner is unresponsive or uncooperative. This would be an accountable process to deal with abandoned gem namespace.

## Motivation

There are several cases where the current gem **ownership request** process can be inadequate.

- **Unresponsive/Deceased Owners**: Sadly, from time to time, gem owners will pass on. In this case, it is not expected that there would be any response from the owner. We need to consider both popular and unpopular gems. Appropriate mitigating factors (like multiple authors) may not be in place when the unfortunate situation occurs.

- **Uncooperative Owners**: Unfortunately, even if contactable, some gem owners are uncooperative. This is understandable as the **ownership request** might come from someone who is unknown to the original owner, and the various issues surrounding gem ownership (including hijacking important gems, etc).

I propose we introduce a new clearly documented and audited process to deal with these cases. Some members of the RubyGems community are nominated as custodians (could include members of Ruby core team for example, admins on RubyGems, etc). Some pre-requisites like experience, activity, etc may be considered. Those members have the ability to go through a gem adoption process either for themselves or on behalf of another member of the community.

## Guide-level explanation

### Gem Adoption Process

- Given that a user makes an **ownership request** on a specific gem, after a certain grace period (e.g. 7 days) they can request a gem custodian review.
- The gem custodian can review all details of the user making the request, including all other gem **ownership request**s they have made, and the details of the current gem **ownership request**.
- Within certain guidelines, they have the ability to approve a gem **ownership request** without the express consent of the current owner.
	- The gem must meet the general criteria of being abandoned.
- The gem custodian can attempt to communicate with the current owner on behalf of the **ownership request**.
	- The current owner can contest the ownership transfer but this is not a veto power.
	- The gem custodian is advised to give sufficient time, at least 30 days, for the current owner to respond.
- Based on this investigation, the gem custodian can approve the **ownership request**.
- Requests of this nature should be peer-reviewed, and at least one gem custodian independent of the user making the **ownership request** should confirm the **ownership request** - this means that gem custodians cannot approve their own requests.
	- Gems which do not meet the requirements for being abandoned require the approval of 3 gem custodians.
	- Any gem custodian can veto a gem **ownership request**.

If this process fails because the current owner rejected the transfer, the **ownership request** owner can re-request a gem custodian after a period of 12 months. The gem custodian should consider this history in their decision making process.

### Abandoned Gem Criteria

A gem is considered "in use" if the gem itself, or any of its direct reverse dependencies (version specific) have more than 10,000 downloads in the past year.

A gem is considered "maintained" if the author has logged into <https://RubyGems.org> within the past 12 months.

A gem is considered to be "valid" if the homepage and other related metadata is valid (e.g. URLs return relevant pages).

A gem is considered to be "working" if a user can check out and run the unit tests on a supported, non-EOL Ruby implementation.

A gem is considered to be "stable" if it has a published release `>= 1.0.0`.

Given the above, we compute an "adoption_threshold". If a gem is "in use" and "maintained", the adoption threshold is 6 years. Otherwise it is 4 years.

A gem is considered "abandoned" if the the most recent release of the gem exceeds the "adoption threshold".

### Example Cases

#### Deceased Owner of Active Gem

A gem does not meet the requirements of being abandoned, but is a popular gem used in the community. A user has forked and maintained the gem in some form and wants to release it. This user has demonstrated their commitment and reliability, and their application may be supported by other reputable members of the community.

They make an **ownership request** explaining the circumstances. After 7 day grace period, a gem custodian is requested.

The gem custodian attempts to contact the original owner, who is unresponsive.

The gem custodian accepts the merits of the original proposal, but they must confirm that the original owner is deceased. Evidence such as GitHub activity, obituaries, etc could be sufficient. Because this gem does not meet the abandoned gem criteria, 3 gem custodians must approve the **ownership request**.

They can also propose the alternative to the user making the **ownership request** that they wait until the gem is considered abandoned.

#### Deceased Owner of Abandoned Gem

A gem meets the requirements for being abandoned. A user wants to reuse the gem name and has demonstrated a basic commitment to following through with this.

They make an **ownership request** explaining the circumstances. After 7 day grace period, a gem custodian is requested.

The gem custodian attempts to contact the original owner, who is unresponsive.

The gem custodian accepts the merits of the original proposal and has sufficient evidence that the original owner is unresponsive. Because the gem meets the abandoned gem criteria, they can approve the request.

#### Uncooperative Owner of Active Gem

A gem does not meet the requirements of being abandoned, and is a popular gem used in the community.

A user makes an **ownership request** explaining the circumstances. This should be automatically marked as having insufficient criteria for approval. After 7 day grace period, a gem custodian is requested.

The original gem owner may ignore the **ownership request** since they consider it spam. They have the option to explicitly reject it, but they may also just ignore it.

The gem custodian can review the situation including past **ownership requests** made by the user and their outcomes.

The gem custodian should reject the **ownership request** and may also choose to provide a note regarding unacceptable **ownership requests**. Continued activity of this nature may result in further corrective action if it is deemed to be abusive.

#### Uncooperative Owner of Abandoned Gem

A gem meets the requirements for being abandoned. A user wants to reuse the gem name and has demonstrated a basic commitment to following through with this.

They make an **ownership request** explaining the circumstances. After 7 day grace period, a gem custodian is requested.

During this time, the original owner might make attempts to update or release trivial changes to the gem so that it no longer meets the criteria for being abandoned.

The gem custodian attempts to contact the original owner. The owner may or may not be responsive. If the original owner is responsive, they can reject the **ownership request**. However, the gem custodian should take note of this.

Within 12 months, no further changes to the gem have occurred, and the gem is still effectively abandoned. A gem custodian can be requested again, and in this situation, they should consider whether this is a case of name squatting.

Because of the efforts of the original owner, the gem may no longer meet the criteria for being abandoned. In this case, 3 gem custodians are required to approve the **ownership request**.

## Reference-level explanation

TBD.
