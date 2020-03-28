- Feature Name: ownership-transfer
- Start Date: 2020-3-19
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

gem owner -a/r command is used to add or remove owners by an email. We will add options for adding/removing gem owners via Web UI. Additionally, we will make owner add/remove events notifiable, i.e. users can opt-in to receive email notification about ownership changes of a gem. 

We will also add a new flow for ownership transfers. Any gem owner `looking for maintainers` will be able to create `ownership request`. Any user can make an `ownership application` for any gem with less than 100,000 downloads and older than 12 months updated at time or gems which are marked as looking for maintainers. When a gem owner approves an `ownership application`, the requester would be added as an owner to the gem. These `ownership requests` will be searchable and listed on a site wide link.

# Motivation

Option of adding and removing owners through the web UI would facilitate friction-less transfer of gem ownership without dependency on CLI configuration on the user-side.

In the current ownership flow, a gem owner can add a new owner without any confirmation from the user being added as an owner. To avoid this spamming, we will add a requirement of confirmation from the user being added.

New ownership transfer flow will make the process more transparent for audits. Gem owners who are looking for new maintainers/owners would be able to open `ownership requests` and let other interested users fill it to show their interest. New searchable view of ownership transfers will make it easier for the users to find the gems which need maintainers.

# Guide-level explanation

### Change in the ownership flow
When a new owner is added to a gem, two types of notifications will be sent via email.
1. A notification to all the existing gem owners about addition of new owner.
2. A notification to the new user to confirm if he/she wants to be added as a gem owner. The new user will have to click on the link sent via email to confirm within 48 hours.

This will require modifications in the `POST - /api/v1/gems/[GEM NAME]/owners` endpoint.

### Marking a gem as looking for maintainers:

A gem owner marks the gem as looking for maintainers using Web UI by clicking a button called `Create Ownership Request` on gems show page in the **Admin** section which will be added above the **Links** section.

An `ownership request` is created with _user_id_ as owner who marked it. 
It will be available in the search with a filter of `looking for maintainers`.

E.g. Alice owns a gem called foobar. If she is no longer interested in maintaining it, she can mark it as `looking for maintainers` by clicking on `Create Ownership Request`. When Bob filters the gems using `looking for maintainers` filter in advanced search, foobar will be listed.

### An example of new ownership transfer flow:

#### Case 1: A gem has less than 100_000 downloads and older than 12 months rubygems.updated_at time

Any user can make an `adoption request` for the gem. The existing owner will be notified by email.
When any of the existing gem owners accepts the `adoption request`, the requester will be added as an owner to the gem and will be notified via an email.
If the gem owner declines the `adoption request`, the requester will be notified via an email.

E.g. When Alice who owns a gem called foobar with 500 downloads which was last updated at 2019-01-01 (today: 2020-03-19), any user of rubygems.org will be able to make an `adoption request` by filling a form. Bob, who is interested in maintaining the gem will fill the form and Alice will be notified. Alice can then accept or decline the request.

Here a case can occur where existing owner may not reply to the email notification. This scenario has to be further discussed and will be a separate project.

#### Case 2: A gem is marked as `looking for maintainers`

A gem owner will mark the gem as `looking for maintainers` as mentioned above.
Any user will be able to make an `ownership application` to the gems marked as `looking for maintainers`.
Any of the existing gem owners can accept the request. The applicant will be added to the gem owners and will be notified via email. The gem will be unmarked as `looking for maintainers` by deleting the `adoption record`.

# Reference-level explanation

For implementing the above features, we will modify the database schema as follows:

Table Name: **ownerships** [existing]

|Column Name|Type|Details|
|:-----------:|:----:|:-------:|
|state |boolean|pending/confirmed|
|token_expires_at|datetime|confirmation token expiry date|

Above changes will be required to implement notification events with confirmation to add new owner. Existing column of `token` will be used to store the generated confirmation token and `token_expires_at` to store the expiry.
If the owner clicks on confirmation link and `token_expires_at > Time.zone.now`, state will change to confirmed.
The link will have to be recreated if it is expired.

Table Name: **ownership_requests** [new]

|Column Name|Type|Details|
|:-----------:|:----:|:-------:|
|rubygem_id |integer|fk to rubygems|
|user_id    |integer|fk to users|
|note       |string |a message from existing owner|
|email_id   |string |in case a gem owner doesn't want to share his personal email id|
|created_at |datetime|timestamp|
|updated_at |datetime|timestamp|

Adoptions table will be for storing if the gem is `looking for maintainers`. A new record is created when an existing owner marks the gem as looking for maintainers.
When an `ownership application` is accepted by the owner `ownership request` record of the gem will be deleted.

Table Name: **ownership_application** [new]

|Column Name|Type|Details|
|:-----------:|:----:|:-------:|
|rubygem_id |integer|fk to rubygems|
|user_id    |integer|fk to users|
|note       |string |a message from existing owner|
|status     |integer|opened: 0, approved: 1, closed: 2|
|approver_id|integer|user_id of owner who approved the request|
|created_at |datetime|timestamp|
|updated_at |datetime|timestamp|

`Ownership application` will store all the users requesting/applying to be maintainers. Users logged in will be allowed to apply if a gem is marked as looking for maintainers i.e. an `ownership request` exists or a gem has less than 100,000 downloads and gem is not updated for more than 12 months.

When user creates new `ownership application`, new record with status open will be created.
When `ownership application` is accepted by any existing gem owner, his/her user_id will be updated in approver_id and status will be updated to approved.
When `ownership application` is rejected by any existing gem owner, his/her user_id will be updated in approver_id and status will be updated to closed.

View additions/modifications are as follows:
- Add a filter to search gems which are `looking for maintainers`. This includes both of the above cases.
- Add a button to apply for adoption in the gem show page.
- Add view to list all adoption requests and allow the owner to accept / decline the requests.

## Drawbacks
1. An extra step will be added to existing simple process of adding a gem owner. With this new ownership flow, the user being added as an owner will have to accept the confirmation sent via email.
This step is necessary to avoid spamming by the users. A user may create a gem and add any random user as an owner.

2. If a user is added as a new maintainer of a gem, he/she might change the complete gem source with something entirely different. 
This drawback has to be addressed via necessary code audits during version updates.
 Probably solutions to this issue may be:
    
    a. We can yank all the previous versions of the gem and give an option to the gem owners to keep the gem name locked for a few days before approving the `adoption request`. Gem install will fail for the users as the versions are yanked.
    
    b. A trusted pool of users from the community can be asked for a review of ownership transfer and gem updates after the ownership transfer. The transfer is approved only after *x* users have approved. This can be done in the period when gem name is locked.