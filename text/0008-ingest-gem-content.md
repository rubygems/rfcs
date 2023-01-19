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

### Ingestion process

The following process will happen when a gem is uploaded or manually triggered:

1. A background job is enqueued with the gem version.
2. The worker pulls down the gem from S3.
3. The gem in expanded into a temporary directory.
4. Each file is enumerated, skipping directories.
5. The worker uses a tool (grep is good at this) to mark all files that contain binary data.
6. Textual files (not binaries) will have lines counted.
7. All files will have size, content_type and SHA256 checksums recorded.
8. A record is created for each file, recording the attributes:
  ```
  version_id
  path (relative to the gem root)
  size
  binary (boolean)
  content_type (mime type determined by `file` command or libmagic)
  sha256 (checksum)
  created_at
  yanked_at (optional, see unresolved questions)
  ```
9. New text files are uploaded to a gem contents s3 bucket with key: `/(gemname)/files/(sha256)`
  (The content S3 bucket is checked to see if a file already exists at the key before uploading.)
10. An attribute is updated on the Version record to indicate the gem content has been successfully uploaded.
  (This is necessary because a partially ingested gem would show a partial file list. Better to show all or nothing.)

### Yank process

When a gem is yanked, additional steps will be performed to remove files and stop serving manifests.



This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks

### Storage and Processing Costs

The most obvious drawback is storage and processing costs.
The increased storage on S3 will cost more.
The increased processing, disk and network usage on the worker will increase worker load and may require additional resources.
The database will become much larger, with the files row using a large and increasing amount of the total database usage.
The public database dumps will become larger as the number of indexed files increases.

The increases in usage will be particularly large if we decide to index all past gems.

Optimizations are planned to reduce total usage, and not all gems will be indexed.
We have discussed allowing a gem to be ingested on demand if the content is not already available.
This could allow users to trigger ingestion of gems they want to browse but were not yet indexed, but avoid ingesting many gems that are not in demand.
The triggered ingestion could also be performed on-demand when the manifest is requested, returning a 202 indicating that the manifest will be returned later. (see Unresolved Questions)


### Potential for Abuse

Another drawback is potential abuse.
Once gem content is made publicly accessible it could be used to host highly available files through Fastly's CDN.
Users might also take advantage of the free, highly available hosting of files to use rubygems.org as a CDN.
It could also be possible to host copyrighted content, exposing rubygems.org to increased handling of DMCA takedowns or other complaints.
Rubygems.org may already hosts these files.
We would not be aware of hosting them currently since they are packaged in a way that accessing them or distributing them widely would require some knowledge of rubygems.
However, once the files are expanded and their raw content readily available at consistent URLs, we may find that abuse increases.
These effects are already mitigated by avoiding the hosting of binary files, since this cuts down on hosting of ebooks, movies, or other copyrighted material that is almost always packaged in binaries.
For reference, GitHub published information stating that [they complied with 1828 DMCA takedown requests in 2021](https://github.blog/2022-01-27-2021-transparency-report/#:~:text=In%202021%2C%20GitHub%20received%20and%20processed%201%2C828%20valid%20DMCA%20takedown,our%20users%20to%20remove%20content.).
Rubygems.org, though smaller than GitHub, could still face an increased burden when contents are exposed directly to the internet.


# Rationale and Alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions

- Feedback on naming. We can tackle column names in individual PRs, but if there is any structural names we'd like to refine, let's address them here.
- Should we keep the file manifest of yanked gems?
  Is a yanked gem intended to be trashed completely or just hidden?
  One possible downside to keeping the file manifest would be if files were committed that leaked private info just by their name or content type.
  I'm leaning towards not saving yanked file manifests, but happy to hear comments.
- Should we try to trigger gem ingestion on-demand when a manifest is requested for a gem that doesn't have one yet?
  This could accidentally cause us to index every gem if a web crawler attempts to view every "browse" page on every gem, once that is available.
- The total size of expanded gems and the processing time to achieve adequate coverage is unknown.
- What subset of gems that should be indexed at the start?
  We should probably index at least some recent versions of common gems to start.
  I don't have a heuristic for deciding which gems meet that criteria.
- Should we aim to index all past gems?
- Should we host _any_ binary files? Images? I lean towards "no" because image viewing is not the intention of this feature.
- Is there a reason to store compiled binaries? Would it support any tooling that might otherwise be difficult to implement?

### Out-of-scope for this RFC

- Format and hosting of manifests and files at an API. While I have a design in mind, I'm holding off on creating the design until a future RFC focused on serving this content.
- Web UI for content browsing.
- CLI support for content browsing.
- API, Web, or CLI support for file diffs.

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before it is on by default?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
