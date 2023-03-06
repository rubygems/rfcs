- Feature Name: Ingest Gem File Contents
- Start Date: 2023-01-18
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Create a process for extracting and uploading the contents of a gem to make its content easily accessible via S3.
The process will run each time a gem is uploaded.
The saved output will include file size, content type, SHA256 checksum, and relative path within the gem.
This RFC focuses specifically on the extracting and uploading of content, and the yanking of the content.
Future RFCs will address usage of gem content for things such as browsing and searching gem content.

# Motivation

RubyGems.org hosts many packages that users install on their machines and execute.
It's important for users to be able to see what code they are about to run and view diffs between versions when upgrading.

Existing source code hosting services currently fill this space.
Many gems are available on GitHub, GitLab, or other hosting services.
However, rubygems.org can't refer users to GitHub to compare gem versions because gems are not required to provide their source publicly.
Even if an official source repository is linked, the gem build process can modify source files arbitrarily before creating the gem, and repositories are not guaranteed to match with gem content.

Gem content may also be viewed by manually downloading the gem with `gem fetch` and then examining the contents.
This requires downloading the content locally without knowing what is in the archive.
Extracting the gem file without installing it is not straight forward, requiring some understanding of the .gem file structure.
These challenges increase the barriers for someone interested in examining gem contents.
This likely skews code review of gems towards code hosting services and tools like GitHub's Dependabot, which only have the published source code and not the actual gem contents.

Rubygems.org hosts the primary gem repository for the Ruby language and is therefore uniquely positioned to offer authoritative gem information.
Creating a gem content ingestion and storage system will enable rubygems.org to support the security and useability of packages in the Ruby ecosystem.
The gem content repository will be the authoritative source for the content of gems held by rubygems.org.
It will provide individual file access without requiring downloading and unpacking a gem.

Future features could include the ability to compare gem versions during gem upgrades, browse gem content online or with the `gem` cli, share gem version diffs by URL, search indexed gem content across one or all gems or compare individual files across many versions.
Local file comparison and non-authoritative sources currently provide many these features, but there is no authoritative online source for the internals of gems.

We believe that rubygems.org is uniquely positioned to provide an authoritative source and an official interface for accessing the content of gems.
This will support the Ruby community and ensure the future security and maintainability of projects that rely on content from rubygems.org.

# Guide-level explanation

For the purpose of illustration, I will use the gem `rake` as an example.

When a gem is uploaded, rubygems.org saves it as a gem version file named `rake-13.0.6.gem`.
A gem may be installed with `gem install` or the `.gem` file can be downloaded using `gem fetch`.
Downloading the gem locally is currently the only way to access the actual contents of gem.

With the gem content ingestion system in place, gem contents will be saved to a content-only S3 bucket and indexed in a way that allows fetching gem content files by path or checksum.

Binary files will be recorded in metadata, but not stored in order to increase security and reduce abuse.
Gem contents will be stored at an S3 key using the gem name and SHA256 of the file.

Hashing the files allows de-duplicate files within a gem, since two files within the same gem name that have the same SHA256 are nearly guaranteed to have the same content.
Between versions, it's very common for many of the files in the gem to remain unchanged.
This allows us to save considerable space in content storage by deduplicating the most commonly duplicated files.
Deduplicating within the gem, but not globally across all gems, ensures the security and integrity of files matched within a gem.
Files contained in the content storage can only be changed or uploaded by owners of that gem, preventing carefully crafted global hash collisions or other malicious activity that could span across gems.

Yanking a gem will remove contents unique to the gem.
Any shared deduplicated files will be preserved using a process that checks the hashed files in all gem versions and preserving any files referenced by any other non-yanked version.

### Background process

In order to ingest the contents of a gem, a background process will be enqueued to index and store the gem contents.

At gem upload time, or on demand to ingest existing gems, a background worker will download, unpack, record, index, and upload each file to a content-only S3 bucket.

The process relies entirely on S3 to index gems and their content, resulting in a number of files.

The background worker creates path and content files as it processes the gem.
At the end of the process, all of the hashes are collected into a .sha256 file.
The process also extracts and uploads the .gemspec file.
No database tables are created or read during this process after initially finding the name of the gem and version number with platform.

### Uploaded File Organization

There are 4 types of files created by the upload process.

#### Spec File

A spec file that contains the actual .gemspec in the .gem file.

S3 Key: `gems/rake/specs/rake-13.0.6.gemspec`.

The spec is generated using `Gem::Package#spec.to_ruby` using the original uploaded .gem file.

#### Path Files

A path file for each file in the gem data with attached metadata.

S3 Key: `gems/rake/paths/13.0.6/lib/rake.rb`.

The path files each contain metadata with the following information about the file and its content.

* The size of the content in bytes.
* The SHA256 checksum of the content, if available (not for symlinks or files larger than 500MB that were skipped)
* The MIME type and charset of the file content, if available (potentially not for extremely large files if the type could not be determined from the file header)
* The original path of the file in the gem.
* The original file mode, as a string of octal digits (`644`).
* The symlink target if the file is a symlink.

Directories are skipped and do not have path entries even though they may exist in the gem.

#### Content file

A content file for each indexed file.

S3 Key: `gems/rake/contents/8e43d75374cfcac83933176c32adb3a01d6488aa34c05bf91f6fe8dfe48c52a5`

The S3 object will contain the exact file contents read from the .gem file.
The content type, content length, and checksum will be saved on the S3 object as found by the content ingestion process, ensuring object integrity after being uploaded.

#### SHA256 file

A file containing the sha256 and file path for all hashed files in the gem.

S3 Key: `gems/rake/checksums/13.0.6.sha256`

The format of this file matches the file output of shasum -a 256 and could be used to check the expanded gem contents against these checksums.

This file is also used to find SHA256 hashes associated with a version for the purpose of the yank process.
When yanking a version, this is the reference used to find which hashes are still in use by indexed versions and which will be deleted during the yank process.

### Recording, but not storing binary files

Files that are identified as "binary" will not be uploaded to the storage bucket.
This is both to save storage space and avoid potential abuse.
Large compiled binaries, image data, or other binary data is not easily browsed or diffed.
Both npmjs.com's code browsing and github.com excludes binary files from display.

Non-indexed files will be determined using `libmagic` via the `ruby-magic` ruby gem.
Any file with a mime type that does not start with `text/` will not have content uploaded.
Files not exceeding 500MB will have all metadata calculated even if the file is not uploaded.

### Using SHA256 to reduce storage size and increase cache hits

As of this writing, there is more than 5TB of gem files stored by rubygems.org.

Using `rake` as an example, the total size of all `.gem` files is currently 6.8MB.

The average compression across the entire history of the rake gem is about 84%.
Expanded fully, all versions of rake add up to 42MB.

Many files in rake do not change between versions, allowing us to optimize the storage.
If each file is instead deduplicated by saving it as the SHA256 checksum, the uncompressed size of all files drops to 10MB.
The resulting directory for the rake gem has 1226 SHA256 named files for all versions of `rake`.

In order to access a hashed filename, two file requests are required.
First access the file path to retrieve the SHA256 checksum, then access the content at the checksum.

A list of all files in a gem is available by listing the keys within a version of a gem.

### Yanking gems

The hashed filenames pose an interesting challenge when yanking a gem.
The relative infrequency of yanking means that we can accept a slightly more intensive process when removing files than when adding them or serving them.

When a version is yanked, the .sha256 file for the yanked version is loaded.
Then, the .sha256 file for all other versions are loaded and "subtracted" from the yanked version.
The result is a list of all checksums that are unique to the gem.

To reduce race conditions, the yank processes will use the bulk delete S3 API to delete all files within a gem as atomically as S3 allows.
Since there are multiple overlapping processes for determining which files should and should not exist in the content store, it is always possible to recover the correct state of the content store if any race conditions or errors occur.

### Indexing older gem versions

Depending on the implementation of future features, gems can be indexed on demand by request or in an automated way.

### Impact

The gem ingestion process should be invisible to end users.
Once the data is used to serve an API, users can expect a slight delay before a newly uploaded gem is available in the content API, usually a matter of seconds if there is nothing in the queue.
Yanking a gem might also have a slight delay before files are not available at the checksum urls, but this should be expected already given the current yank process timing.

# Reference-level explanation

### Ingestion process

The following process will happen when a gem is uploaded or manually triggered:

1. A background job is enqueued with the gem version.
2. The worker pulls down the gem from S3.
3. The gem in read in memory using existing and newly modified behaviors on rubygems Gem::Package.
4. Each file is enumerated, skipping directories.
5. The worker uses a libmagic to indentify all files that contain `text/*` content.
6. All files will have size, content_type and SHA256 checksums recorded.
7. A path file is created, recording the metadata above:
8. Files containing text are uploaded to a gem contents s3 bucket keyed by the hash of the content.
  (The content S3 bucket is checked to see if a file already exists at the key before uploading.)
9. The spec file is generated and uploaded.

### Yank process

When a gem is yanked, additional steps will be performed to remove files and stop serving manifests.

1. Load all stored checksums from all versions of the gem.
2. Find checksums in the yanked version that do not match with any file in another version of the gem.
3. Remove all files unique to the yanked version of the gem.

There are potential race conditions when the upload and yank process runs simultaneously for two gems.
The faster the yank process, the less likely it will be to encounter race condition.
Ultimately, S3 is not immediately consistent and we may want to add an additional job that checks yanked gems daily to ensure that the files in the content store are as expected.


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

One trade-off would be an on-demand caching approach.
Only the first "live" request is streamed, then the rest of the gem is stored for future requests.
Another trade-off could be caching of the manifest, but always streaming files from the gem.

Most of the trade-offs increase complexity compared to a front-loaded approach.
The proposed solution in this RFC aims only to process a gem a single time.
The files are stored only once and the process that stores them is not performance or time constrained.
The files are always accessed directly from block storage through a cache and the manifest is always available before hand.

In another situation, this transparent proxy could make sense.
It allows the system to store less data and the costs scale exactly with the demand for the feature.
For rubygems.org, which is does not have a full time staff to maintain a more complex solution, we believe a trade-off towards storage cost and away from operational complexity is warranted.

#### AWS Lambda

We considered running these on AWS Lambda, using a S3 trigger event when a `.gem` was uploaded.
The Lambda solution optimizes for higher throughput, but adds costs and complexity.
For example, while the code in the Lambda function may be similar to a background job, it would still be a separate code base from the rails app.
Testing lambda functions requires more steps and is more involved than testing Rails background jobs.
The production and staging infrastructure also becomes more complicated, increasing the number of repositories and systems involved in running rubygems.org.

### SHA256 Checksum File Storage

Given the immense size of the uncompressed rubygems.org gem database, it benefits us to make optimizations.

During our research, we were inspired by the solution arrived at by npmjs.com.
Both the manifest structure and file storage approach herein is influenced by npmjs.com's design.

We use a manifest to convert between paths and file checksums.
The checksums serve a dual purpose on both npmjs.com and in our solution, to both verify and deduplicate content.

Not every file changes between every version of a gem.
Most of the files in a gem stay the same between two consecutive releases.
In our testing, for example, the entire history of rake is about 4 times smaller when deduplicated by matching checksums.
We therefore propose a solution that uses the checksum of a file as the filename for content storage.
It both increases the likelihood of a cache hit on any given request and reduces the number of files and storage size of almost every gem.

#### Global Checksums

For a time we considered applying this checksum approach globally.
The apparent benefit would be in deduplicating files across gems.
Files like LICENSE and other boilerplate could be deduplicated across the entire gem repository.
Some gems vendor other gems to reduce dependencies (e.g. bundler).
It seems like there could be enough benefit to warrant this solution.

However, we decided against it for a few reasons, the primary being yanking gems.
If we had to consider every file as a possible orphan, we would have to search an index of every file hash, without constraints, for a matching checksum before deleting a file.
This would require building a reverse index, hash => version, for every hash of every file.
It's possible that this additional burden could negate the space savings entirely.

If we instead constrain our deduplication to only files within the same gem, we get most of the benefit without the increase in file overlaps.
When only files within the same gem are considered for deduplication, we may miss a few small optimizations, but we reduce complexity while still gaining a large reduction in storage size.
Since the files within a gem can only change with versions of that gem, there's less likelihood for race conditions during yank.
If one gem adds a file, while another yanks the only other reference to that file, we could end up with the incorrect state.
This is a much smaller problem within the same gem where we already assume consistency between versions and do not have to consider concurrent uploads of any other gems.

Npmjs.com lends some validity to this approach. File contents are stored at a path which includes the package name and checksum.
Since we have no information about how much space it would save, we also can't appropriately measure the benefit of global hashing.

Once enough gems are indexed, we could estimate the total savings of this approach and consider whether it makes sense to change.
Starting with the solution proposed in this RFC will reduce the complexity of changing to global deduplication, if it is ever warranted.
Moving from global index to gem-scoped index is more complicated than gem-scoped to global, the former requiring only a few s3 cli commands, whereas the latter requires careful reindexing and duplication of every file.

### Binary Files

There are many ways to detect binary files and choices to be made for serving them.

In the interest of providing gem contents for the purpose of browsing on the web, we think it is best not to serve the binary portions of a gem.
Binary files are not able to be browsed or diff'd in any meaningful way on a web page.

Avoiding binaries solves many of the problems listed in drawbacks and reduces storage size significantly.
For example, the gem `libv8-node-18.13.0.0-arm64-darwin.gem` we examined is 30MB.
Uncompressed, the whole gem is about 98MB, with a single binary `.a` file that is 97MB.
Not storing the only binary file in this gem saves 99% of the uncompressed storage space while providing almost all the benefit of offering gem browsing.

A user that wants to examine a binary file in a gem is effectively downloading more than the file size of the gem.
They would be better served downloading the gem directly for the purpose of examining the binary files contained within.

#### Detecting Binary Files

There are many solutions that can be found for detecting binary files.
We intend to use `libmagic` in memory via the ruby gem `ruby-magic` to generate mime types for every file.
We will exclude files with mime types starting with anything other than `text/`.

### Delete Yanked Gem Manifests

Originally an unresolved question about deleting the file manifest.
Given that when gems are yanked, almost all of their information is deleted or hidden, it doesn't make sense to maintain the manifest of files for a yanked gem.
All gem file contents and all manifest info will be deleted and purged upon completion of the yank process.

# Unresolved questions

- Should we try to trigger gem ingestion on-demand when a manifest is requested for a gem that doesn't have one yet?
  This could accidentally cause us to index every gem if a web crawler attempts to view every "browse" page on every gem, once that is available.
- The total size of expanded gems and the processing time to achieve adequate coverage is unknown.
- What subset of gems that should be indexed at the start?
  We should probably index at least some recent versions of common gems to start.
  I don't have a heuristic for deciding which gems meet that criteria.
- Should we aim to index all past gems?
- Should we host _any_ binary files? Images? I lean towards "no" because image viewing is not the intention of this feature and increases the likelihood that rubygems.org is abused for free hosting.
- Is there any reason to store compiled binaries? Would it support any tooling that might otherwise be difficult to implement such as virus scanning?

### Out-of-scope for this RFC

- Format and hosting of manifests and files at an API.
- Web UI for content browsing.
- CLI support for content browsing.
- API, Web, or CLI support for file diffs.
- API, Web, or CLI for code search.
