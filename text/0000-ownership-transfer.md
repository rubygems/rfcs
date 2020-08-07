- Feature Name: ownership-transfer
- Start Date: 2020-3-19
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

gem owner -a/r command is used to add or remove owners by an email. We will add an option of adding/removing gem owners via the Web UI. We will add email notification to the owner add/remove events. Owners can opt-out of receive email notification about ownership changes of a gem. We will make owner additions auditable by storing when the new owner was added and by whom. We will require a confirmation from the user added as an owner.

We will also add a new flow for ownership transfers. Any gem owner will be able to create an `ownership call`. Any user will be able to make an `ownership request` for any gem with less than 100000 downloads and older than 12 months _updated_at_ time or gems which have an associated `ownership call`. When a gem owner approves an `ownership request`, the requester must be added as an owner to the gem. The `ownership calls` will be searchable and listed on a site-wide link.

# Motivation

Adding and removing owners through the web UI would facilitate the frictionless transfer of gem ownership without dependency on the gem CLI installation and configuration on the user-side.

Email notification and audit trail of owner additions will improve the mitigation time of unintended access grants.
Adding a new owner without any confirmation from the user being added as an owner, potentially lead to email notification spamming. Making the user being added as an owner confirm the addition, we will avoid sending unsolicited emails.

New ownership transfer flow will automate the process of requesting gem ownership transfers and additions. Gem owners who are looking for new maintainers/owners will be able to open `ownership calls` and let interested users apply with `ownership request` to show their interest. A New searchable view of `ownership calls` will make it easier for the users to find the gems which need maintainers.

# Guide-level explanation

### Change in the ownership flow

When a new owner is added to a gem, the following types of notifications are sent via email.

1. A notification to the new user to confirm if he/she wants to be added as a gem owner. The new user must click on the link sent via email to confirm within 48 hours.
2. A notification to the existing gem owners about the addition of new owner after the confirmation from the new user.
3. A notification to the gem owner who is removed from the gem CLI or rubygems.org UI.

This will require modifications in the `POST - /api/v1/gems/[GEM NAME]/owners` endpoint.

### Creating an ownership call:

The gem owner will create an `ownership call` using Web UI from ownerships page by navigating via **Ownerships** link in the **Links** section on the gems show page.

E.g., Alice owns a gem called foobar. If she is no longer interested in maintaining it, she will create an `ownership call` by clicking on `Ownership Call`. When Bob filters the gems using `ownership calls` filter in advanced search, foobar will be listed.

### An example of new ownership transfer flow:

#### Case 1: A gem with less than 100,000 _downloads_ and older than 12 months _rubygems.updated_at_ time

Any logged-in user makes an `ownership request` for the gem. The existing owners will be notified by an email.

Here the limit on _downloads_ and _updated_at_ is used to prevent spamming of popular gems with `ownership requests`. These limits may be relaxed in the future.

E.g., When Alice, who owns a gem called foobar with 500 downloads, which was last updated at 2019-01-01 (today: 2020-03-19), any user of rubygems.org will be able to make an `ownership request` by filling a form. Bob, who is interested in maintaining the gem, will fill the form, and Alice will be notified. Alice should then accept or decline the request.

Here a case may occur where the existing owner may not reply to the email notification. This scenario has to be further discussed and will be a separate RFC.

#### Case 2: A gem has associated `ownership call`

A gem owner will create `ownership call` as mentioned above.
Any user will be able to make an `ownership request` to the gems with associated `ownership call`.

After the `ownership request` is created, any of the existing gem owners should accept the `ownership request`. If accepted, the requester will be added as an owner to the gem and will be notified via an email, a new `ownership` record will be created for corresponding _rubygem_ and new owner's _user_id_. The user who accepts the `ownership request` will be tracked in _approver_id_ column. The existing owners will get email notification about new owner being added by `ownership request` approval.
If the gem owner declines the `ownership request`, the requester will be notified via an email of the same.

### Rate Limits

With these new features, many new processes are added, which will need rate limits to control spamming and abuse of our email service and limit unnecessary email notifications. Following rate limits will be added to the mentioned processes:

1. Confirmation notification to a new user to be added as a gem owner:

   RATE_LIMIT: 10 emails per 10 minutes.

   KEY: `params['user_id'] in { controller: "owners", action: "confirm" }`

2. Email notifications about new `ownership requests` to the gem owner:

   RATE_LIMIT: 1 email per day

   Here a single email notification will be sent only if new `ownership requests` have been created in the last 24 hours by grouping by the owner and by gem to each unique owner. This will send aggregated feed of requests to owners instead of multiple emails each time a user creates a new `ownership request`.

3. Limit on creating `ownership requests` by a user:

   We will limit the number of `ownership requests` a user can create to any gem. It will prevent any user from spamming with requests.

   RATE_LIMIT: 10 new requests per day

   KEY: `params['user_id'] in { controller: "ownership_request", action: "create" }`

# Reference-level explanation

For implementing the above features, we will modify the database schema as follows:

Table Name: **ownerships** [existing]

|   Column Name    |   Type   |                 Details                  |
| :--------------: | :------: | :--------------------------------------: |
|    confirmed     | boolean  |     true if new owner has confirmed      |
| token_expires_at | datetime |      confirmation token expiry date      |
|  push_notifier   | boolean  | existing notifier column will be renamed |
|  owner_notifier  | boolean  |    notifier for add/remove gem owner     |
|  authorizer_id   | integer  |      user who added this ownership       |
|   confirmed_at   | integer  |    timestamp of confirmation by user     |

The above changes will be required to implement notification events with confirmation to add a new owner. The existing column of `token` will be used to store the generated confirmation token and `token_expires_at` to store the expiry.
If the owner clicks on the confirmation link and `token_expires_at > Time.zone.now`, state should change to confirmed.
The link will have to be recreated if it is expired.

A default scope will be added to show only confirmed gem owners.

`push_notifier` column will be used to store email preference for gem push
`owner_notifier` column will be used to store email preference for owner addition or remove.

Table Name: **ownership_calls** [new]

| Column Name |   Type   |                             Details                             |
| :---------: | :------: | :-------------------------------------------------------------: |
| rubygem_id  | integer  |                         fk to rubygems                          |
|   user_id   | integer  |                           fk to users                           |
|    note     |   text   |                  a message from existing owner                  |
|   status    | boolean  |                    open: true, close: false                     |
| created_at  | datetime |                            timestamp                            |
| updated_at  | datetime |                            timestamp                            |

`ownership_calls` table will be for storing `ownership calls` created by any gem owners along with a note from the owner. A new record is created when an existing owner creates an `ownership call`.
When the owner wants to stop accepting `ownership call`, he/she can mark it as closed.

Table Name: **ownership_requests** [new]

|     Column Name      |   Type   |                            Details                            |
| :------------------: | :------: | :-----------------------------------------------------------: |
|      rubygem_id      | integer  |                        fk to rubygems                         |
|   ownership_call_id  | integer  |                     fk to ownership_call                      |
|       user_id        | integer  |                          fk to users                          |
|         note         |   text   |   a message for gem owner from user who created the request   |
|        status        | integer  |               opened: 0, approved: 1, closed: 2               |
|     approver_id      | integer  |           user_id of owner who approved the request           |
|      created_at      | datetime |                           timestamp                           |
|      updated_at      | datetime |                           timestamp                           |

`Ownership request` will store requests from users who are applying for ownership of a gem.
When user creates new `ownership request`, new record with status _open_ will be created.
When `ownership request` is accepted by any existing gem owner, his/her _user_id_ will be updated in _approver_id_ and _status_ will be updated to _approved_.
When `ownership request` is rejected by any existing gem owner, his/her _user_id_ will be updated in _approver_id_ and status will be updated to _closed_.

View additions/modifications are as follows:

- Add view for adding/removing an owner to the gem.
- Add view for easy audits with details about owners like MFA, user who approved the ownership, timestamp of ownership addition.
- Add a filter to search gems which have `ownership call`. This includes both of the above cases.
- Add a button to apply for adoption in the gem show page.
- Add view to list all ownership requests and allow the owner to accept / decline the requests.
- Add a label to gem show page for gems with `ownership calls`.
- Add new preferences to the notifier view.

## Drawbacks

1. An extra step will be added to the existing simple process of adding a gem owner. With this new ownership flow, the user being added as an owner will have to accept the confirmation sent via email.
   This step is necessary to avoid spamming of gem push notifications by the addition of random users as maintainers.

2. If a user is added as a new maintainer of a gem, he/she may change the complete gem source with something entirely different or yank existing versions. This is not a particularly new issue and exists with current process of owner addition.
