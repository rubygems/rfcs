- Feature Name: `PostgreSQL major version upgrade - preparations`
- Start Date: 2023-09-30
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Phase 1 of the PostgreSQL upgrade process for RubyGems.org revolves around making the necessary preparations to transition from version 11 to version 12 with minimal disruptions.

# Motivation

The key goal of this phase is to ensure that all crucial tools, configurations, and scripts are in place, facilitating a seamless migration from PostgreSQL 11 to 12. By making the most of the mandatory downtime for enabling replication, we also include the "pg_stat_statements" for enhanced monitoring, maximizing the value derived from this period.

# Guide-level explanation

The initial phase of the PostgreSQL upgrade will include:

1. Setting up `pgbouncer` in the Kubernetes cluster with associated services to manage connections.
2. Initializing a new PostgreSQL 12 RDS instance using Terraform.
3. Configuring the existing PostgreSQL 11 instance, adjusting for logical replication and other settings, followed by a necessary restart.
4. Activating logical replication between PostgreSQL 11 and 12 using manually executed reference scripts (https://github.com/rubygems/pg-major-update).

# Reference-level explanation

For this upgrade's groundwork, a mix of Kubernetes, Terraform, and manually executed scripts will be utilized:

1. **`pgbouncer` Deployment and Service in Kubernetes**:
   - `pgbouncer` is deployed in the Kubernetes cluster via a straightforward deployment to address connection pooling requirements.
   - A Kubernetes service gets established, exposing `pgbouncer` on a specific port, ensuring easy connectivity.
   - Manual configurations are applied to `pgbouncer` using `pgbouncer.ini` and `userlist.txt`.

2. **PostgreSQL 12 Initialization using Terraform**:
   - Using Terraform, a new PostgreSQL 12 RDS instance is brought up using `db.m5.xlarge` instance size.

3. **Configuring PostgreSQL 11 with Terraform**:
   - Terraform adjusts the existing PostgreSQL 11 RDS instance to fit the upcoming steps, which include a required restart. This also introduces "pg_stat_statements" for improved DataDog monitoring.

4. **Downtime and Restart**:
   - A planned, brief downtime will be scheduled for the PostgreSQL 11 restart. Advance communication will be made to the user base to ensure minimal inconvenience.

5. **Replication setup**:
   - Post restart, logical replication between the two PostgreSQL instances is set using manually executed reference scripts.

# Drawbacks

1. There will be a short service disruption due to the required downtime for configuring the PostgreSQL 11 RDS instance for replication.
2. Operational costs will rise temporarily since two PostgreSQL instances will run in parallel during this phase.

# Rationale and Alternatives

Initiating with a solid preparation phase ensures that the subsequent upgrade steps can move without significant hitches. While there's an option to proceed without such a phased approach, the benefits of minimized disruption, enhanced monitoring, and controlled process make this strategy more appealing.
