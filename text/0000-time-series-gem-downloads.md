- Feature Name: Store gem download counts in a timeseries database
- Start Date: 2023-03-16
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Today, RubyGems only stores one "all time" download count per gem. Users and maintainers would like to see download counts for each gem or version per day, week, month, or year in order to see usage changes over time. In order to provide these counts, we will start storing download counts in TimescaleDB, a time-series database.

# Motivation

- Users and maintainers would like to see gem usage over time.
- We would like to make insertion of download counts require no contention on the production postgres database.
- We would like to avoid maintaining bespoke infrastructure for download count tracking.
- We want to migrate to the new storage system without losing historical download counts.

As a followup, we will work on displaying the new data in RubyGems.org, but that design work is out of scope for this RFC.

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation

RubyGems.org tracks the number of times each gem & gem version is downloaded.
That data is available at a 15 minute granularity for the past week,
daily for the past 2 years,
and monthly and yearly for all time.

# Reference-level explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

We will set up a second database that instances of the rubygems.org rails app will connect to.
We will use a timescale cloud instance, with a single table added:

```ruby
class CreateDownloads < ActiveRecord::Migration[7.0]
  def change
    create_table :downloads, id: false do |t|
      t.integer :rubygem_id, null: false
      t.integer :version_id, null: false
      t.integer :downloads, null: false
      t.integer :log_ticket_id, null: true
      t.timestamptz :occurred_at, null: false
    end

    add_index :downloads, [:rubygem_id, :version_id, :occurred_at, :log_ticket_id], unique: true, name: 'idx_downloads_by_version_log_ticket'
    create_hypertable :downloads, :occurred_at, chunk_time_interval: '7 days', partitioning_column: 'rubygem_id'
  end
end
```

That on top of that table (and hypertable, which is how timescale efficiently stores timeseries data), we will define some continuous aggregates:

```ruby
class CreateDownloadsContinuousAggregates < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def create_continuous_aggregate(name, ...)
    execute "DROP MATERIALIZED VIEW IF EXISTS #{connection.quote_table_name(name)} CASCADE;"
    super
  rescue ActiveRecord::StatementInvalid => e
    raise unless e.cause.is_a?(PG::DatetimeFieldOverflow)
    say "WARNING: #{e}"
    nil
  end

  def continuous_aggregate(
    duration:,
    from: Download.table_name,
    start_offset:,
    end_offset:,
    schedule_window:,
    retention: start_offset
  )
    name = "downloads_" + duration.inspect.parameterize(separator: '_')
    start_offset ||= 20.years

    create_continuous_aggregate(
      name,
      Download.
        time_bucket(duration.iso8601, select_alias: :occurred_at).
        group(:rubygem_id, :version_id).
        select(:rubygem_id, :version_id).
        sum(:downloads, :downloads).
        reorder(nil).
        from(from).
        to_sql
    )
    add_continuous_aggregate_policy(name, start_offset&.iso8601, end_offset&.iso8601, schedule_window)
    add_hypertable_retention_policy(name, retention.iso8601) if retention

    all_versions_name = name + "_all_versions"
    create_continuous_aggregate(
      all_versions_name,
      Download.
        time_bucket(duration.iso8601, select_alias: :occurred_at).
        group(:rubygem_id).
        select(:rubygem_id).
        sum(:downloads, :downloads).
        from(name).
        to_sql
    )
    add_continuous_aggregate_policy(all_versions_name, start_offset&.iso8601, end_offset&.iso8601, schedule_window)
    add_hypertable_retention_policy(all_versions_name, retention.iso8601) if retention

    all_gems_name = name + "_all_gems"
    create_continuous_aggregate(
      all_gems_name,
      Download.
        time_bucket(duration.iso8601, select_alias: :occurred_at).
        sum(:downloads, :downloads).
        from(all_versions_name).
        to_sql
    )
    add_continuous_aggregate_policy(all_gems_name, start_offset&.iso8601, end_offset&.iso8601, schedule_window)
    add_hypertable_retention_policy(all_gems_name, retention.iso8601) if retention

    name
  end

  def change
    ActiveRecord::Base.logger = Logger.new(STDERR)
    from = continuous_aggregate(
      duration: 15.minutes,
      start_offset: 1.week,
      end_offset: 1.hour,
      schedule_window: 1.hour
    )

    from = continuous_aggregate(
      duration: 1.day,
      start_offset: 2.years,
      end_offset: 1.day,
      schedule_window: 12.hours,
      from:
    )

    from = continuous_aggregate(
      duration: 1.month,
      start_offset: nil,
      end_offset: 1.month,
      schedule_window: 1.day,
      retention: nil,
      from:
    )

    from = continuous_aggregate(
      duration: 1.year,
      start_offset: nil,
      end_offset: 1.year,
      schedule_window: 1.month,
      retention: nil,
      from:
    )
  end
end
```

Then, in the `FastlyLogProcessorJob`, we will aggregate all downloads in a log by (version full name, 15 minute time bucket),
and find_or_create a Download record for that version (with its version ID and rubygem ID) with the download count, log ticket ID, and time bucket.

# Drawbacks

Why should we _not_ do this? Please consider the impact on existing users, on the documentation, on the integration of this feature with other existing and planned features, on the impact on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

- Adding another datastore complicates contributing to the application
- Another datastore means we can't natively write joins against the download data
- Timescale cloud costs money

# Rationale and Alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

- Timescale has support for "continuous aggregates" and data retention policies. Combined, this makes it possible for us to seamlessly simultaneously support "downloads per hour for the past month" and "downloads per day for the past year" without manually needing to handle rollups, writing queries that cross the timeframe boundaries
- If we don't do this, download data will continue to become less useful, as the mass of downloads moves further and further into the past
- We considered implementing rollups manually in postgres, but that seemed like it would be a lot of code we'd be on the hook maintaining for forever

# Unresolved questions

- How can we make having a timescale setup optional for contributors who are not working on downloads?
- What time period will we assign for migrated data?
- How many historical fastly logs will we re-parse?
- How do we want to present this data to users?
- What new API endpoints will we add?
- Do downloads need an ID?

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before it is on by default?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
