- Feature Name: ownership-transfer
- Start Date: 2020-3-19
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

gem owner -a/r command is used to add or remove owners by an email. We will add options for adding/removing gem owners via Web UI. Additionally, we will make owner add/remove events notifiable, i.e. users can opt-in to receive email notification about ownership changes of a gem.

We will also add a new flow for ownership transfers, where users can make an `adoption request` for any gem with less than 100,000 downloads and older than 12 months updated at time or gems which are marked as looking for maintainers. When a gem owner approves an `adoption request`, the requester would be added as an owner to the gem. These requests will be searchable and listed on a site wide link.

# Motivation

Option of adding and removing owners through the web UI would facilitate friction-less transfer of gem ownership without dependency on CLI configuration on the user-side.

New ownership transfer flow will make the process more transparent for audits. Gem owners who are looking for new maintainers/owners would be able to open adoption requests and let other interested users fill it to show their interest. New searchable view of ownership transfers will make it easier for the users to find the gems which need maintainers.

# Guide-level explanation

### An example for marking a gem as looking for maintainers:

User marks the gem as looking for maintainers using Web UI by clicking a button on gems show page.
An `adoption record` is created with user_id as owner who marked it. 
It will be available in the search with a filter of `looking for maintainers`.
E.g. Alice owns a gem called foobar. If she is no longer interested in maintaining it, she can mark it as `looking for maintainers` from the show page of the gem. When Bob filters the gems using `looking for maintainers` filter in advanced search, foobar will be listed.

### An example of new ownership transfer flow:

#### Case 1: A gem has less than 100_000 downloads and older than 12 months rubygems.updated_at time

Any user can make an `adoption request` for the gem. The existing owner will be notified by email.
When any of the existing gem owners accepts the `adoption request`, the requester will be added as an owner to the gem and will be notified via an email. The owner who accepted the `adoption request` will no longer be an owner.
If the gem owner declines the `adoption request`, the requester will be notified via an email.

Note: Ownership transfer will happen from user who accepted `adoption request` to the user who created `adoption request`.

E.g. When Alice who owns a gem called foobar with 500 downloads which was last updated at 2019-01-01 (today: 2020-03-19), any user of rubygems.org will be able to make an owner request by filling a form. Bob, who is interested in maintaining the gem will fill the form and Alice will be notified. Alice can then accept or decline the request.

#### Case 2: A gem is marked as `looking for maintainers`

A gem owner will mark the gem as `looking for maintainers` as mentioned above.
Any user will be able to make an `adoption request` to the gems marked as `looking for maintainers`.
Any of the existing gem owners can accept the request. The applicant will be added to the gem owners list and will be notified via email. The gem will be unmarked as `looking for maintainers`.

Note: Ownership transfer will happen from user who created `adoption record` to the user who created `adoption request `.


# Reference-level explanation

For implementing the above features, we will modify the database schema as follows:

Table Name: adoptions [new]

|Column Name|Type|Details|
|:-----------:|:----:|:-------:|
|rubygem_id |integer|fk to rubygems|
|user_id    |integer|fk to users|
|status     |integer|accepted/rejected/pending|
|created_at |datetime|timestamp|
|updated_at |datetime|timestamp|

Table Name: adoption_requests [new]

|Column Name|Type|Details|
|:-----------:|:----:|:-------:|
|rubygem_id |integer|fk to rubygems|
|user_id    |integer|fk to users|
|created_at |datetime|timestamp|
|updated_at |datetime|timestamp|

Table Name: ownerships [existing]

|Column Name|Type|Details|
|:-----------:|:----:|:-------:|
|archived |boolean|true/false|

- Adoptions table will be for storing if the gem is `looking for maintainers`.
- Adoption requests will store all the users requesting/applying to be maintainers
- When an ownership transfer is done, the previous owner will be marked as archived. This will help knowing previous owners for the audit trail.

Logic changes and view additions/modifications are as follows:
- Add a filter to search gems which are `looking for maintainers`. This includes both of the above cases.
- Add a button to apply for adoption in the gem show page.
- Add view to list all adoption requests and allow the owner to accept / decline the requests.

# Timeline

## Community Bonding Phase
### Week 1 (27th April - 4th May 2020)
- Finish the RFC and get reviews from mentors.
- Wait 1 week for getting suggestions on the RFC.

### Week 2 (5th May - 11th May 2020)
- Finalise the workflow and RFC.
- Finalise the schema changes required.
- Finish the migrations and model design for adoptions                                                                                                                 

### Week 3 (12th May - 18th May 2020)
- Add `looking for maintainers` filter to search
- Add badge to UI to show if gem is `looking for maintainers`
- Add search tests for above filter and integration test for UI changes                          

## Phase 1
### Week 4 (19th May - 25th May 2020)
- Get mentor reviews, fix bugs and do code cleanup                                                                                                                                                    

### Week 5 (25th May - 1st June 2020)
- Work on model design and flow logic for adoption requests
- Add UI buttons for users to request
- Add new view for owners to accept / decline requests
- Modify the existing ownership flow to fit new logic 

### Week 6 (2nd June - 8th June 2020)
- Get mentor review and fix bugs and concerns                                                                                                                                                         

### Week 7 (9th June - 15th June 2020)
- Write factories for new models
- Write unit tests for new ownership transfer flow
- Write integration tests for new view for owners
- Write functional tests for adoption controllers                        

## Phase 2
### Week 8 (15th June - 22nd June 2020)
- Add mail notifications to the flow as needed.
- Add UI for getting ownership trail
- Final Code clean up                                                                                                  

### Week 9 (23rd June - 29th June 2020)
- Update guides about new ownership transfer flow
- Add blog article about the update                                                                                                                    

### Week 10 (30th June - 6th July 2020)
- My end semester exams were postponed due to recent Corona outbreak. So I won't be able to work for this week.

### Week 11 - 12 (7th July - 20th July 2020) 
- Get final mentor reviews
- Buffer period

# Rationale and Alternatives
In case when an owner marks the gem as `looking for maintainers` and a few users have submitted `adoption requests`, there are following possible cases:
1. Any existing of the gem owners can accept the `adoption request` but the transfer will happen only between the user who marked `looking for maintainers` and the applicant.
2. Only the gem owner who created `adoption record` / marked gem as `looking for maintainers` will be able to see the `adoption requests` and accept / decline them. The transfer will happen only among them.

In case when user creates `adoption request` for a gem with less than 100,000 downloads and more than 12 months of updated time, which owner should be changed?