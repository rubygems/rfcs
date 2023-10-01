- Feature Name: `PostgreSQL major version upgrade`
- Start Date: 2023-09-30
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

A structured four-phase approach to upgrade the PostgreSQL database for RubyGems.org from version 11 to version 12 with minimal downtime.

# Motivation

The primary goal is to ensure RubyGems.org benefits from the latest features, performance improvements, and security patches in PostgreSQL 12. Furthermore, by adopting this systematic approach, we lay down a foundation for streamlined future database upgrades, ensuring consistent minimal disruptions. Currently running PostgreSQL 11 reaches EOL on RDS at February 2024.

# Guide-level explanation

The PostgreSQL upgrade is organized into four distinct phases:

1. **[Preparations](https://github.com/rubygems/rfcs/pull/53)**: 
   - Set up the necessary tools and configurations, including initializing a PostgreSQL 12 RDS instance and deploying `pgbouncer` in Kubernetes.
   - Configure the current PostgreSQL 11 RDS instance to support logical replication, accounting for the brief downtime required to enable this feature.
   - Initiate logical replication from the PostgreSQL 11 instance to the newly created PostgreSQL 12 instance, ensuring data synchronization begins.
   - Backup both PostgreSQLs instances.
  
2. **Sanity Checks**: 
   - Test and verify logical replication.
   - Ensure `pgbouncer` functionalities are working as expected.
   - Simulate and test the effects of `goodjob` worker shutdowns, pgbouncer pool pausing and measuring time needed to get in full sync.
  
3. **Final Migration**: 
   - Pause `goodjob` workers.
   - Confirm synchronization between the PostgreSQL instances including manual synchronization of sequences.
   - Switch to the PostgreSQL 12 instance.
   - Deploy using direct connection to resume `goodjob` workers.

4. **Cleanup**: 
   - Decommission outdated resources.
   - Ensure backups target the new PostgreSQL 12 instance.
   - Monitor to ensure stable operations and performance.

# Reference-level explanation

This meta RFC offers an overview of the entire upgrade process. Each phase's specific technical details, challenges, solutions, and examples will be elaborated in its dedicated RFC.

**Note**: Scripted implementation of the migration process will be available for reference.

# Drawbacks

The primary challenge is the brief downtime required to configure the existing PostgreSQL 11 RDS instance for logical replication. Furthermore, while gem availability will not be affected, background jobs, specifically those handled by `goodjob`, will be paused. This introduces a temporary halt (estimated at around 1 minute) of gem indexing activities.

# Rationale and Alternatives

The proposed phased approach ensures a systematic upgrade with predictable and minimal disruptions. An alternative would be a direct AWS upgrade, resulting in an approximate 10-minute downtime. However, this method poses unpredictable challenges, such as uncertain start times for the downtime and potential complications during the upgrade process.

# Unresolved questions

- Are there any interdependencies between the phases that need special attention?
- What additional resources or tools might be required for each phase?
- How will `shoryuken` workers, which seem to utilize pooled connections, be affected, and what provisions need to be made for them during the migration?
- Is the effort and complexity of this phased approach more advantageous when compared to a straightforward AWS RDS upgrade, considering potential benefits and challenges of both methods?
