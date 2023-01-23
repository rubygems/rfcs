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

The background worker will create a json document manifest of all the files and store it in S3 alongside the files themselves.

The manifest will hold the path, checksum, size, content type, binary determination, and plain text line count.

In order to support yanking gems, which requires comparing checksums across all versions of a gem, a table will be added to the rubygems.org database.
The table will consist of one row per gem version with a foreign key for the `version_id` and a jsonb column holding only the SHA256 checksums of all files in that version.

The table will be indexed on `version_id` to support looking up all checksums in all versions of a gem.
This table is for the sole purpose of yanking a gem, discussed below.

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

Rubygems.org is maintained by open source contributors that may not be working on this system full-time.
Therefore, many of the decisions made herein try to optimize for reducing complexity.
Reduced complexity will make this feature easier to maintain, even if it is not the most optimal solution for price or performance.

### Processing gems

We considered a number of alternate approaches to processing gems.
We believe the approach presented herein, using background workers within the rubygems.org Rails app, is the simplest and lowest maintenance solution.

#### Transparent Proxy

We considered creating an API that would respond to requests for individual files in a version of a gem.
The API would transparently load the gem and stream the file from the gem back during the request.
This would allow us to optimize storage space by unpacking gems and files only as needed.

However, this solution is more complex and more demanding on our infrastructure.
Streaming content would still require that we know what content is in the gem.

A potential solution using this approach would need to accept requests for any version of a gem.
It would then try to fetch the `.gem` file, unzip it, ingest the file structure, find the requested file and return it.
This request would need to reliably happen within a reasonable response time.
If it was hosted within the rails app, it would need to be fast so that it didn't cause request queuing.
If it was instead hosted as a standalone app, the necessary infrastructure would add significant complexity.

The trade-off necessary to make this solution performant and reliable could make it very hard to maintain.

One trade-off would be an on-demand caching approach. Only the first "live" request is streamed, then the rest of the gem is stored for future requests.
Another trade-off could be caching of the manifest, but always streaming gems.

Most of the trade-offs increase complexity compared to a front-loaded approach.
The proposed solution in this RFC aims only to process a gem a single time.
The files are stored only once and the process that stores them is not performance or time constrained.
The files are always accessed directly from block storage through a cache and the manifest is always available before hand.

In another situation, this transparent proxy could make sense.
It allows the system to store less data and the costs scale exactly with the demand for the feature.
For rubygems.org, which is does not have a full time staff to maintain a more complex solution, we believe a trade-off towards storage cost and away from operational complexity is warranted.

#### AWS Lambda?

We considered running these on AWS Lambda, using a S3 trigger event when a `.gem` was uploaded.
The Lambda solution optimizes for higher throughput, but adds costs and complexity.
For example, while the code in the Lambda function may be similar, it must maintain its own connection to the rubygems.org database in order to save data (whether directly or via an API provided by rubygems.org).
Testing lambda functions requires more steps and is more involved than testing Rails background jobs.
The production and staging infrastructure also becomes more complicated, increasing the number of repositories and systems involved in running rubygems.org.

### SHA256 Checksum File Storage

Given the immense size of the uncompressed rubygems.org gem database, it benefits us to make optimizations.

During our research, we were inspired by the solution arrived at by npmjs.com.
Both the manifest structure and file storage approach is influenced by npmjs.com's design.

When returning files, we serve a manifest that links pathnames to checksums.
The checksums serve a dual purpose on both npmjs.com and in our solution.
The first purpose is the common solution of verifiability of file contents.
The second purpose, which I find clever, is to deduplicate files within a package.

Not every gem version changes every file.
Most of the files in a gem stay the same between two consecutive releases.
In our testing, for example, the entire history of rake is about 4 times smaller when deduplicated by matching checksums.
We therefore propose a solution that uses the checksum of a file as the filename.
It both increases the likelihood of a cache hit on any given request and reduces the number of files and storage size of almost every gem.

#### Global Checksums

For a time we considered applying this checksum approach globally.
The apparent benefit would be in deduplicating files across gems.
Files like LICENSE and other boilerplate could be deduplicated across the entire gem repository.
Some gems vendor other gems to reduce dependencies (e.g. bundler).
It seems like there could be enough benefit to warrant this solution.

However, we decided against it for a few reasons, the primary being yanking gems.
If we had to consider every file as a possible orphan, we would have to search the entire file table without constraints for a matching checksum before deleting a file.
If we instead constrain our deduplication to only files within the same gem, we get most of the benefit without the increase in file overlaps.
When only files within the same gem are considered for deduplication, we may miss a few small optimizations, but we reduce complexity.
Since the files within a gem can only change with versions of that gem, there's less likelihood for race conditions.
If one gem adds a file, while another yanks the only other reference to that file, we could end up with the incorrect state.
This is a much smaller problem within the same gem where we already assume consistency between versions and do not have to consider concurrent uploads of any other gems.

Supporting this approach is npmjs.com, which uses package name and checksum to fetch file content.
Since we have no information about how much space it would save, we also can't appropriately measure the benefit of global hashing.

Once enough gems are indexed, we could estimate the total savings of this approach and consider whether it makes sense to change.
Starting with the solution as proposed will reduce the complexity of changing.
Moving from global index to gem-scoped index is more complicated than gem-scoped to global.
The solution we propose in this RFC could be converted to hash globally with a few simple aws cli commands.
The reverse, global to gem-scoped migration, would require re-parsing every gem manifest to copy files to the appropriate gem directory.

### Binary Files

There are many ways to detect binary files and choices to be made for serving them.

In the interest of providing gem contents for the purpose of browsing on the web, we think it is best not to serve the binary portions of a gem.
Binary files are not able to be browsed or diff'd in any meaningful way on a web page.

Avoiding binaries solves many of the problems listed in drawbacks and reduces storage size significantly.
For example, the `libv8-node-18.13.0.0-arm64-darwin.gem` we examined is 30MB.
Uncompressed, the whole gem is about 98MB, while the single binary `.a` file is 97MB.
Not storing the only binary file in this gem saves 99% of the uncompressed storage space while providing almost all the benefit of offering gem browsing.

A user that wants to examine a binary file in a gem is effectively downloading more than the file size of the gem.
They would be better served downloading the gem for that purpose.

#### Detecting Binary Files

There are many solutions that can be found for detecting binary files.
We intend at this time to use `grep` to quickly print a list of files that it thinks are binary, and therefore not text searchable.
We do this by relying on `grep` to output when a binary file matches.

```
$ grep -RHm1 '^' libv8-node-18.13.0.0-arm64-darwin.gem/* | grep 'Binary'
Binary file data/vendor/v8/arm64-darwin/libv8/obj/libv8_monolith.a matches
```

Alternate solutions involve using `file` (the implementation of which varies between different linux and unix based systems) or libmagic (which requires us to add and compile and additional dependency).
By using `grep` to separate binaries, we use a tool available on every system that behaves in way that is already familiar to most developers.

We still may need to use `file` or `libmagic` to discover the content type of a file.
The output from `file` on MacOS says it will always have the word `text` in a text file, but it warns that other implementations may not be consistent.
Using `grep` for determining the `binary` formatted files ensures that we reliably identify files that we don't intend to store.

### Delete Yanked Gem Manifests

Originally an unresolved question about deleting the file manifest.
Given that when gems are yanked, almost all of their information is deleted or hidden.
We therefore conclude that it doesn't make sense to maintain the manifest of files for a yanked gem.
All gem file contents and all manifest info will be deleted and purged upon yanking a gem.
The database table containing the version_id and a list of checksums for each version will be used to determine orphaned files that will be removed and purged.

# Unresolved questions

- Feedback on naming. We can tackle column names in individual PRs, but if there is any structural names we'd like to refine, let's address them here.
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
