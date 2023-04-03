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

# Guide-level explanation

RubyGems.org tracks the number of times each gem & gem version is downloaded.
That data is available at a 15 minute granularity for the past week,
daily for the past 2 years,
and monthly and yearly for all time.

This data will be available under new API endpoints, which will be designed in a followup.

# Reference-level explanation

We will set up a second database that instances of the rubygems.org rails app will connect to.
We will use a [managed timescale cloud](https://www.timescale.com) instance, with a single table added:

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

Then on top of that table (and hypertable, which is how timescale efficiently stores timeseries data), we will define some continuous aggregates:

```ruby
class CreateDownloadsContinuousAggregates < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def quote_table_name(...)
    connection.quote_table_name(...)
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

    create_continuous_aggregate(
      name,
      Download.
        time_bucket(duration.iso8601, quote_table_name("#{from}.occurred_at"), select_alias: 'occurred_at').
        group(:rubygem_id, :version_id).
        select(:rubygem_id, :version_id).
        sum(quote_table_name("#{from}.downloads"), :downloads).
        reorder(nil).
        from(from).
        to_sql,
      force: true
    )
    add_continuous_aggregate_policy(name, start_offset&.iso8601, end_offset&.iso8601, schedule_window)
    add_hypertable_retention_policy(name, retention.iso8601) if retention

    all_versions_name = name + "_all_versions"
    create_continuous_aggregate(
      all_versions_name,
      Download.
        time_bucket(duration.iso8601, quote_table_name("#{name}.occurred_at"), select_alias: 'occurred_at').
        group(:rubygem_id).
        select(:rubygem_id).
        sum(quote_table_name("#{name}.downloads"), :downloads).
        from(name).
        to_sql,
        force: true
    )
    add_continuous_aggregate_policy(all_versions_name, start_offset&.iso8601, end_offset&.iso8601, schedule_window)
    add_hypertable_retention_policy(all_versions_name, retention.iso8601) if retention

    all_gems_name = name + "_all_gems"
    create_continuous_aggregate(
      all_gems_name,
      Download.
        time_bucket(duration.iso8601, quote_table_name("#{all_versions_name}.occurred_at"), select_alias: 'occurred_at').
        sum(quote_table_name("#{all_versions_name}.downloads"), :downloads).
        from(all_versions_name).
        to_sql,
        force: true
    )
    add_continuous_aggregate_policy(all_gems_name, start_offset&.iso8601, end_offset&.iso8601, schedule_window)
    add_hypertable_retention_policy(all_gems_name, retention.iso8601) if retention

    name
  end

  def change
    # https://github.com/timescale/timescaledb/issues/5474
    Download.create!(version_id: 0, rubygem_id: 0, occurred_at: Time.at(0), downloads: 0)

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

Each continuous aggregate will get its own subclass of `Download`, so we are able to query agains the matrix of timeframes x [single gem version, all versions for a gem, all gems] using a consistent interface.

For example, all of the following will be possible (and queriable scoped to particular timeframes, gems, gem versions, etc.)

```ruby
irb(main):004:0> Download::P1MAllVersion.all
2023-04-03 10:11:48.902642 D [920:8000 log_subscriber.rb:130] ActiveRecord::Base --   Download::P1MAllVersion Load (173.3ms)  SELECT "downloads_1_month_all_versions".* FROM "downloads_1_month_all_versions"
=>
[#<Download::P1MAllVersion:0x000000011cfd37a8 occurred_at: Sun, 01 Nov 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, downloads: 6>,
 #<Download::P1MAllVersion:0x000000011cfd3668 occurred_at: Tue, 01 Dec 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, downloads: 1>]
irb(main):005:0> Download::P1YAllGem.all
2023-04-03 10:11:57.916116 D [920:8000 log_subscriber.rb:130] ActiveRecord::Base --   Download::P1YAllGem Load (75.0ms)  SELECT "downloads_1_year_all_gems".* FROM "downloads_1_year_all_gems"
=> [#<Download::P1YAllGem:0x000000011c6de6c0 occurred_at: Thu, 01 Jan 2015 00:00:00.000000000 UTC +00:00, downloads: 7>]
irb(main):006:0> Download::P1D.all
2023-04-03 10:12:01.759285 D [920:8000 log_subscriber.rb:130] ActiveRecord::Base --   Download::P1D Load (43.5ms)  SELECT "downloads_1_day".* FROM "downloads_1_day"
=>
[#<Download::P1D:0x000000011cef9b20 occurred_at: Mon, 30 Nov 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, version_id: 21, downloads: 2>,
 #<Download::P1D:0x000000011cef9a80 occurred_at: Wed, 30 Dec 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, version_id: 23, downloads: 1>,
 #<Download::P1D:0x000000011cef99e0 occurred_at: Mon, 30 Nov 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, version_id: 22, downloads: 1>,
 #<Download::P1D:0x000000011cef9940 occurred_at: Sun, 29 Nov 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, version_id: 23, downloads: 1>,
 #<Download::P1D:0x000000011cef98a0 occurred_at: Mon, 30 Nov 2015 00:00:00.000000000 UTC +00:00, rubygem_id: 11, version_id: 23, downloads: 2>]
```

Note that the raw `Download` class will not be exposed anywhere -- as far as the rails app is concerned, it's a write-only table, and all queries will happen against the continuous aggregates.

# Drawbacks

- Adding another datastore complicates contributing to the application
- Another datastore means we can't natively write joins against the download data
- Timescale cloud costs money
- There is no terraform provider for timescale cloud (yet) requiring us to write one or manually provision
- This is still high cardinality data, and there's no knowing for sure how well timescale will compress it

# Rationale and Alternatives

- Timescale has support for "continuous aggregates" and data retention policies. Combined, this makes it possible for us to seamlessly simultaneously support "downloads per hour for the past month" and "downloads per day for the past year" without manually needing to handle rollups, writing queries that cross the timeframe boundaries
- If we don't do this, download data will continue to become less useful, as the mass of downloads moves further and further into the past
- We considered implementing rollups manually in postgres, but that seemed like it would be a lot of code we'd be on the hook maintaining for forever

# Unresolved questions

- How can we make having a timescale setup optional for contributors who are not working on downloads?
- What time period will we assign for migrated data?
- How many historical fastly logs will we re-parse?
- How do we want to present this data to users?
- What new API endpoints will we add?
- Do downloads (and the continuous aggregates defined on them) need an ID?
