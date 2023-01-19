- Feature Name: Ingest Gem File Contents
- Start Date: 2023-01-18
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Create a process for the purpose of processing and saving the contents of a gem.
The process will run each time a gem is uploaded to save information about each file in the gem.
The saved output will include file size, content type, SHA256 checksum, and relative path within the gem.
In the interest of simplifying this RFC and the feature work that follows, this RFC focuses on only the ingestion and storage of gem content.

# Motivation

It is our hope that we can provide access to individual file data from gem files across versions in a compact and efficient way that supports future features depending on gem content.

Creating a gem content ingestion and storage system will enable rubygems.org to support the security and useability gems in the Ruby Gems ecosystem.
The gem content repository will be the authoritative source for the content of gems, accessible as individual files without without requiring downloading and unpacking a gem.

In the near future, the indexed gem content will allow an API for accessing the gem content.
The API will support both rubygems.org and external projects that wish to use individual gem content.
Rubygems.org plans to add tools for browsing gem content on rubygems.org or through the `gem` command line interface.

# Guide-level explanation

For the purpose of illustration, I will use the gem `rake` as an example.

When a gem is uploaded, rubygems.org saves it as a tarball file named `rake-13.0.6.gem`.
A gem may be installed with `gem install` or the `.gem` file can be downloaded using `gem fetch`.
Downloading the entire gem locally is currently the only way to access the contents of gem.

With the gem content ingestion system in place, gem contents will be saved to S3 and indexed in the rubygems.org database.
Binary files will be recorded in metadata, but not stored.
Gem contents will be stored at an S3 key named for the SHA256 of the file to de-duplicate files within a gem.
Yanking will remove file contents unique to the gem, as expected, without altering unyanked gem contents.

### Background process

In order to ingest the contents of a gem, a background process can be enqueued to index and store the gem contents.

At gem upload time, or on demand, a background worker will download, unpack, record file metadata, index files in the database as rows, and upload each file to a content-only S3 bucket.
The process will result in a row in the database for each file in a gem, with the relative path, checksum, and size of every file.

### Indexing files to create a manifest

The background worker will create a row for each file in a file manifest table.

The table will hold the path, checksum, size, content type, binary determination, and plain text line count.
Each row in the table will be associated with the correct gem version, in this example `rake` version `13.0.6`.

The table will be indexed to support fast lookups for common operations.
The most common is likely "list files for a version of a gem" which will naturally index on the gem `version_id`.
Other common operations may surface, like finding all versions of the same file within a gem, or all files with the same checksum within a gem.

The table is likely to be large with hundreds of millions of rows.
Performance must be considered to reduce its impact on the rubygems.org database as a whole.
If performance becomes a concern, a generated json manifest can be stored along with the gem contents in S3 which would solve for the most common requests.

### Recording, but not storing binary files

Files that are identified as binary will be flagged in the index and not uploaded to the storage bucket.
This is both to save storage space and avoid potential abuse.
Large compiled binaries, image data, or other binary data is not easily browsed or diffed.
Both npmjs.com's code browsing and github.com excludes binary files from display.

### Using SHA256 to reduce storage size and increase cache hits

As of this writing, there is more than 5TB of gem files stored by rubygems.org.

Using `rake` as an example, the total size of all `.gem` files is currently 6.8MB.

The average compression across the entire history of the rake gem is about 84%.
Expanded fully, all versions of rake add up to 42MB.

Many files in rake do not change between versions, allowing us to optimize the storage.
If each file is instead saved at its SHA256 checksum, causing and identical file to be stored only once, the size of all uncompressed files in `rake` droms from 42MB to 10MB.
The resulting directory has 1226 files for all versions of `rake`.

In order to access a hashed filename, access the file content table to find all files in the version of rake.
Use the relative file path within the gem to find the SHA256 checksum.
Access the S3 bucket at a key similar to `/rake/files/c588eda0547f6fa72fd7cd9ba2ddc2e81b96e61d690a929e8f579412da27eebd`

### Yanking gems

The hashed filenames pose a slight challenge when yanking a gem.
The relative infrequency of yanking means that we can accept a more involved process removing files than adding them or hosting them.

When a version is yanked, the file manifest for that version will be used to search for any hashed files that are unique to the yanked gem.
If a SHA256 named file would be orphaned by yanking the gem (it has no gem versions that reference it) then the file will be deleted from S3 and purged from Fastly's cache.
If a file has other references, then the file was already a duplicate of a file in a previous version. Files with multiple references will be preserved until the last reference is removed.

When a gem is yanked the manifest table rows recording the file names with checksum, size could be preserved but no longer shown anywhere.

### Indexing older gem versions

Depending on the implementation of future fetaures, gems can be indexed on demand by request or in an automated way.

### Impact

The gem ingestion process should be invisible to end users.
Once the data is used to serve an API, users can expect a slight delay before a newly uploaded gem is available in the content API.
Yanking a gem might also have a slight delay before files are not available at the checksum urls, but this should be expected already.

Future features and external apps that use the contents API will have a reliable and authoritative source for gem contents that can be accessed on demand.

# Reference-level explanation

TODO...

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks

Why should we *not* do this? Please consider the impact on existing users, on the documentation, on the integration of this feature with other existing and planned features, on the impact on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Rationale and Alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before it is on by default?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
