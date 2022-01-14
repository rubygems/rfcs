- Feature Name: mfa_rollout
- Start Date: 2022-01-12
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Multi-factor authentication (MFA) is a method of authentication that requires at least two verification factors to authenticate the user. MFA in the form of TOTPs (Time-Based One-Time Passwords) is currently implemented in Rubygems.

We will add improvements to current API keys and the gem publishing process such as keys scoped to a gem, pushing multiple gems and setting expiries on keys. We will create a policy so that at a certain date, the 100 most downloaded gems (which covers 31 billion out of the 82 billion total downloads as of November 23, 2021) will be required to have MFA enabled either on the `UI and API` or the `UI and gem signin` level. They will have the option to change and set up another device. To help with a smooth transition, there will be a time period of 6 months before the requirement where we will encourage adoption by displaying the setup MFA page after login or sign up and info messages on CLI commands. If the rollout for the 100 most downloaded gems is successful, we can continue this rollout to other existing user accounts.

# Motivation

From a [recent study](https://arxiv.org/pdf/2002.01139.pdf), account takeover is the second most common attack on software packages after typosquatting. Multi-factor authentication is a well known method in mitigating this type of attack.

Over the years, Rubygems has grown to serve over 180,000 gems and has over 82 billion gem downloads. We want to make sure that consumers of well known gems are downloading owner published code, and that owner accounts are protected. Other package managers are following this pursuit. Github recently announced their commitment to [requiring MFA for a cohort of packages in npm](https://github.blog/2021-11-15-githubs-commitment-to-npm-ecosystem-security/#automated-malware-detection-and-requiring-2fa-to-combat-maintainer-account-takeovers) and we should do the same.

To reduce most of the risk, users would have MFA enabled on the `UI and API` level. However, due to the use of automated systems and scripts, it would be difficult to enforce. The API key improvements will reduce the risk as much as possible.

# Guide-level explanation

This policy will be planned in phases and it will  be critical to send communications (eg. announcements and emails) outlining the policy (especially for Phase 2 and 3).

## Phase 1: API Key and Gem Publishing Improvements (est. Q1 2022 - Q2 2022)

Along with the current features being implemented (setting MFA on certain keys and WebAuthn support), here is the guide level explanation of the further features we will implement for API keys.

### Keys scoped to a gem

When creating a new key, the user will have the option to scope the API key to a specific gem. If selected, there will be a dropdown of the owner’s gems they can select from. On the CLI, if they choose the scope to a gem, they will need to type the name of the gem they want to scope it to.

If the user is removed as an owner of a gem, the API key will be marked as invalid on the API keys page. Editing the key would be prohibited and the user can delete the key.

### Setting an expiry on keys

When creating a key, users will be able to select the validity period upon creation of new keys (1 day, 7 days, 30 days, and 90 days) with the default of 30 days. Users won’t be able to edit the expiration after the API key has been created.

When the API key is expired, the API key will be marked as invalid on the API keys page. Editing the key would be prohibited and the user can delete the key.

Up for discussion, but we propose that all existing keys will be subject to a certain expiry time of around 6 months from the time of the rollout, and all new push keys are required to set an expiry date. These expirations would be staggered to avoid a thundering herd effect.

### Push multiple gems at once

When publishing a gem, authors can push multiple gems at once using `gem push <gems> [options]`. If the user has MFA enabled, an OTP code is to be entered once. Only one host may be specified for all gems. The response would include the results of pushing each gem (success or error). This is for gem owners that own many gems and would need to enter an OTP per gem push which can be cumbersome.

## Phase 2: Recommending MFA on Most Downloaded Gems (est. Q2 2022 - Q3 2022)

Given that a user is an owner of the 100 most downloaded gems, there are multiple ways we will encourage users to set up MFA.

On the UI, after login, the setup MFA page will be displayed for users to setup MFA. They will have the option to back out and come back to it. A banner will also appear describing why users are seeing this.
	
Example message
```
For protection of your account and your gems, we encourage you to set up multi-factor authentication.
```

On the CLI, on `gem signin`, `gem push/yank`, and `gem owner -a/r`, a warning message will display to encourage users to setup MFA on rubygems.org and that at a point of time, this will be a requirement . Command specific behavior would not be affected.

Example message for gem signin (general audience it can also be targeted to the specific audience)
```
For protection of your account and gems, we encourage you to set up multi-factor authentication at https://rubygems.org/multifactor_auth/new.
```

Example message for critical gem commands (`gem push/yank`, and `gem owner -a/r`) for the 100 most downloaded gem accounts
```
[WARNING] For protection of your account and gems, we encourage you to set up multi-factor authentication at https://rubygems.org/multifactor_auth/new. Your account will be required to have MFA enabled in the future.
```

## Phase 3: Enforcing MFA on Most Downloaded Gems (est. Q4 2022)

At a future date, yet to be determined, we will require MFA to be enabled in order to interact with Rubygems.org. Enforcement will include the following changes.

On the UI, after login, the setup MFA page will be displayed for users to setup MFA. They are able to back out, but the user will be directed to set up MFA whenever they choose to visit a page that requires authentication. A banner will also appear describing the intent. 

Example message
```
For protection of your account and your gems, you are required to set up multi-factor authentication.
```

On the CLI, on `gem signin`, `gem push/yank`, and `gem owner -a/r`, an error message will display to require users to setup MFA on rubygems.org and will block the request from succeeding. 

Example message
```
For protection of your account and your gems, you are required to set up multi-factor authentication at https://rubygems.org/multifactor_auth/new
```

## Phase 4: Rolling out to the General Public (2023)

Once the rollout of the top 100 gems is successfully rolled out, a similar rollout will be done to another cohort of gems until all accounts are required to have MFA enabled. 

# Reference-level explanation

## API Key and Gem Publishing Improvements
The API key model will need to be changed in order to implement these improvements. For new features like adding gem scope and expiry date on gems, columns will be added for expiry and gem scope and additional options will need to be added to the CLI and the UI. Setting an expiry on existing keys can be considered a breaking change.

Gem publishing will be improved by allowing gem owners to push mulitple gems at once, so that they only need to enter their OTP code one time. A prototype for pushing multiple gems has been briefly explored here:  https://github.com/Shopify/rubygems/pull/2

## Policy Rollout
A group of proofs of concept are listed below that demonstrate what areas of the Rubygems codebase will be modified in order to implement the policy. More detailed explanations and implementation alternatives are available in their PR descriptions.

- UI flow: https://github.com/Shopify/rubygems.org/pull/10
- CLI: https://github.com/Shopify/rubygems.org/pull/9 

Currently [WebAuthn support](https://github.com/rubygems/rubygems.org/pull/2865) is being added. This is a solution to add security key support as well as giving users the ability to add multiple forms of authentication in addition to the TOTP method. These implementations will need to change slightly based on this feature to check if the user has one form of MFA enabled but the general flow will still be the same. 

# Drawbacks

The main drawback of this feature is the potential amount of feedback and support demand from end-users that can arise with requiring MFA. Users may not recognize the risks associated with not enabling MFA and think this extra step to be burdensome on their workflow. This goes for the API key features we’re implementing as well. Most users will need to adopt a habit of creating keys more often due to these added restrictions.

# Rationale and Alternatives

One alternative to MFA raised by gem maintainers is gem signing. Signing in theory does provide peace in verifying the integrity of the package, but in practice, the current system has [room for improvement](https://emboss.github.io/blog/2013/02/15/we-need-to-sign-ruby-gems-but-how/). More directly, gem signing does not prevent account takeovers, so we consider it out of scope for this RFC.

Another simple alternative to MFA is enforcing the usage of better passwords. However, there are [many types of attacks](https://techcommunity.microsoft.com/t5/azure-active-directory-identity/your-pa-word-doesn-t-matter/ba-p/731984) against complex passwords. Stronger passwords remain vulnerable against reuse and leaks.  MFA in addition to these countermeasures will further reduce the risk of malicious code injection in gems.

In light of critical account takeovers in the package ecosystem that has led to malicious package versions, many package maintainers are enabling MFA and [requiring all publishers to enable MFA](https://github.com/search?l=&p=1&q=%22rubygems_mfa_required%22+extension%3A.gemspec&ref=advsearch&type=Code). Some consumers now want to factor in whether gem owners have MFA enabled when they want to install a gem ([discussion](https://github.com/rubygems/rubygems.org/discussions/2889)).

Without requiring MFA on user accounts, there is a greater risk of malicious gem releases due to account takeovers on the Rubygems platform. This will cause a larger burden to the owners of the malicious package version, end users and Rubygems maintainers when these attacks occur.

# Unresolved questions

- Legacy keys are a relic of pre Rubygems 3.2 but still are in existence. What would the impact be of removing legacy key support (it comes with removing Rubygems <3.2 also)?  Legacy keys bring a larger risk to accounts and can bypass the restrictions we are placing on API keys. More specifically, how many legacy keys are still being used recently among the 100 most downloaded gem accounts and the greater ecosystem? Are there any metrics determining the amount of people in Rubygems <3.2?
- Risks are not fully resolved if users have the `UI and gem signin` MFA level enabled. How can we better deal with the CI/CD case so that we have a safer plan for them? Could we enforce non-MFA keys to have restrictive scopes like per gem and a short expiry?
- How do we feel about the number of gems in the initial rollout? We don’t know how many folks in the specified cohort there are, and what portion does not have MFA enabled. It would be insightful to know these metrics and have access to these metrics during the rollout.
- Should all accounts have MFA enabled by default? There are many gems that have little to no downloads. It would be a lot to ask of these accounts to enable MFA right from the beginning. For the long term, a better solution would be to require MFA for accounts in which their gem downloads reach a certain threshold ([current gem download distribution per gem](https://github.com/Shopify/rubygems.org/pull/7)). The positive of having MFA for all accounts is that we’re setting a higher bar of security if users want to join/stay on the platform. A [prototype](https://github.com/jenshenny/rubygems.org/pull/3) of setting up MFA as a sign up process has been briefly explored.
