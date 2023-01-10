- Feature Name: gem_contents_api
- Start Date: 2022-12-20
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Provide an API for accessing gem contents.
Support code browsing, security, and enable future features that rely on per-file access to gem source.
The gem contents could be rendered on rubygems.org or accessed directly via the API.
The API consists of two endpoints: a gem "manifest" endpoint and a file endpoint.
The API should be available for a substantial number of gem versions, if not all.

# Motivation

Ensuring the security and stability of an application requires reviewing the contents of dependencies.
Currently, to review the contents of a gem, you must download the gem locally or find the gem's source online.

Online representations of the gem's source code have a number of shortcomings.
Source control tags help with viewing and comparing files as they exist in a certain version. Changelogs, diffs and commit logs can help compare versions.
However, there is no guarantee that the online source code is a reliable representation of the gem contents.

Rubygems supports selectively omitting files from the packaged build. This means that even well maintained source may differ from the contents of the gem.
Gems are often built and released from local developer machines which introduces additional build environment variability.
More nefariously, a malicious gem publisher could publish source code that is different from the gem package contents in an attempt to disguise malicious code.

Currently, the only way to inspect the true contents of a gem is to download the gem to a machine and examine the installed source.
Rubygems.org therefore has the opportunity to offer a trusted source of gem contents on the website and via an API that does not require downloading and extracting the gem package.

In addition to supporting the security of rubygems, developers considering using a gem may appreciate a single first-party source for the actual contents of a gem.
Providing immediate access to gem source from rubygems.org will simplify previewing of gem behavior without having to find the correct version or commit on an external site.

# Guide-level explanation

> Explain the proposal as if it was already implemented and released, and you were teaching it to another developer. That generally means:
>
> - Introducing new named concepts.
> - Explaining the feature largely in terms of examples.
> - Explaining how users should _think_ about the feature, and how it should impact the way they use Bundler. It should explain the impact as concretely as possible.
> - If applicable, provide sample error messages, deprecation warnings, or migration guidance.
> - If applicable, describe the differences between teaching this to existing users and new users.
>
> For implementation-oriented RFCs, this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

---

Access gem contents via the new contents url.

This feature introduces 2 new endpoints that serve the file contents of a gem package.
The files returned from the API are extracted from the official gems store at rubygems.org.
The contents will exactly match the contents of the gem when installed (before any install scripts are run).

In order to access a gem's contents, first load the manifest.

Manifest path:
`https://rubygems.org/api/v1/contents/rake-13.0.6.json`

A file and directory structure manifest is available at a url (something like /gem-name-1.2.3/contents/manifest.json). This manifest provides information about the files in the gem, with things like content type, file size, checksum, and other relevant details.

Source are returned raw inline in the body. Binary files, including images and compiled output, are not stored or returned.

Content path
`https://rubygems.org/api/v1/files/f71e8ed126b46346494aad5486874cd8f0aafe95092ed67d2e3cb6110f939abc`.

The hex string for looking up a file is the SHA256 checksum of the file. This cuts down on storage and increases cache hits by deduplicating identical files. Many files in gems do not change between releases, and some (like LICENSE files) will have significant overlap across gems.

## Manifest

In order to access a file, you'll need to fetch the manifest for the gem.

curl https://rubygems.org/api/v1/contents/rake-13.0.6.json

The manifest contains a reference to every file with the following information:

(TODO: should we add number of lines? anything else?)

```
{
  "/path/to/file.rb": {
    "size": 123, # in bytes
    "contentType": "text/plain",
    "path": "/path/to/file.rb",
    "binary": false,
    "key": "abcdef1234567789094aad5486874cd8f0aafe95092ed67d2e3cb6110f939abc"
  },
  "/LICENSE": {
    "size": 222,
    "contentType": "text/plain",
    "path": "/LICENSE",
    "binary": false,
    "key": "f71e8ed126b46346494aad5486874cd8f0aafe95092ed67d2e3abcabcabcabca"
  },
}
```






## What's in a gem?

A Ruby gem is a tarball (.tar) containing 3 files:

```
metadata.gz
data.tar.gz
checksums.yaml.gz
```

`metadata.gz` contains the computed gemspec in yaml format. `data.tar.gz` contains the gem's files (the contents we are interested in), and `checksums.yaml.gz` holds 2 checksums of each of metadata.gz and data.tar.gz.

A simple list of files in the gem can be obtained via (using bundler as an example)

    tar --to-stdout -xf bundler-2.4.2.gem data.tar.gz | tar -zvt

This command outputs something similar to `ls -l`.

In order to prepare a gem for the content API, we will need to do a few steps.

1. Expand the data.tar.gz.
2. Record for each file:
  - File path.
  - Content type from `file` command. Filter for binary based on content type.
  - File size
  - Number of lines
  - SHA256 checksum
3. Store each file to an S3 bucket with filename as SHA256 checksum


# Reference-level explanation

> This is the technical portion of the RFC. Explain the design in sufficient detail that:
>
> - Its interaction with other features is clear.
> - It is reasonably clear how the feature would be implemented.
> - Corner cases are dissected by example.
>
> The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

---

1. Expand the contents of gems
2. Store a manifest that indicates directory structure and file types (e.g. flagging binary files so they are not displayed in the browser).
3. Provide an API for users to access the manifest and load individual files.

### Expand the gems

The volume of traffic at rubygems.org is high enough that we should anticipate and build for high traffic from the start.

Expanding all gems allows us to avoid a complicated proxy process and instead serve content in a consistent way, potentially using a flag to indicate whether the gem has content browsing available.

If there is a way to create a subset of gems that covers an acceptable percentage of likely usage (ignoring sufficiently old gems or gems that have not been downloaded recently) then we could avoid storage costs for gem contents that will never be browsed.

The S3 bucket should be exclusively for expanded gem contents.
The contents of a gem will never change, so the storage should be treated as read-only except to remove yanked gems.

It will be necessary to store a manifest that indexes all the files in the gem, their size, checksum, and other information. While file contents will never change, the calculated information inside the manifest may need to be updated as our information about the usage of this feature improves or other needs arise. Making this information queryable will improve the feature. Therefore, it makes sense to use the database to store the manifest information.

Until a gem version has its manifest created, indicating that the files are available in the cache, rubygems.org can show a placeholder for the content browsing page.

### Store the Manifest

The manifest should be stored in the database, either as a json document or rows for each file. The manifest should never change once it is generated.


Using postgres jsonb column could allow searching and indexing columns in the document.

### Yanking a gem

One of the challenges of the content system will arise when a gem is yanked.

In addition to the existing behaviors, we will need to delete and invalidate cache on any file that is _unique_ to the gem. This is the "cost" incurred for the benefit of deduplication of files.

In order to delete files unique to the gem, we must search the manifests of all gems to see if any other gem references the same checksum. If the checksum is unique in the database, then the file is deleted and cache invalidated. If not unique, then it is kept.
It would make sense therefore to have an index on files by checksum to speed up this process.

The process for cleaning yanked gems will be slowed by this reference counting behavior.

# Drawbacks

> Why should we _not_ do this? Please consider the impact on existing users, on the documentation, on the integration of this feature with other existing and planned features, on the impact on existing apps, etc.
>
> There are tradeoffs to choosing any path, please attempt to identify them here.

## Storage size

Initial back of the napkin math landed on about 3TB of storage for the current know sizes of hosted rubygems, assuming about 80% compression ratio and 20% duplicates or binaries that can be thrown out.

A first look at some of the largest gems seems to indicate that removing binaries (such as images) would save a significant amount of space out of the largest gems. It's not clear if this space savings would result in a significant change in expanded size.

We could approach expanding the gems progressively, allowing only the smallest gems to be expanded at first and checking space usage. We could also filter for plain text file times and exclude anything other than plain text.

## Potential for abuse

There is potential for abuse. If rubygems.org now serves public versions of any file that anyone uploads, supported by an edge cache with high availability, it could enable bad actors to host highly available files for free. While this is already possible on github and in many places on the internet, it may become difficult or impossible to discover abuses. Rubygems.org currently only serves compressed archives uploaded as gems, so this change opens a new avenue for storing almost any file of any size.

Hosting uploaded content the raw form could expose rubygems.org to responsibility for DMCA takedown requests. This could already happen with the archives as is, but when the file contents are served by rubygems.org directly, it may become a larger issue. GitHub [indicates](https://github.blog/2022-01-27-2021-transparency-report/#:~:text=In%202021%2C%20GitHub%20received%20and%20processed%201%2C828%20valid%20DMCA%20takedown,our%20users%20to%20remove%20content.) that they complied with 1828 DMCA takedown requests in 2021. Rubygems.org could still face an increased burden when contents are exposed directly to the internet.

Examining npm's code browsing shows that they deliver a manifest that references files at a single file serving URL rather than serving files at their original path in the package. They use a 64 character hex string, possibly a SHA256 hash of the file, as a key to retrieve a file. There are probably many reasons for this approach. File paths could be long or could contain non-url friendly characters. It limits directly accessing files by obscuring their location. It makes the url for accessing files more uniform which might make access controls or caching simpler.

Limiting the serving of binary files could reduce potential abuse.
When reviewing npm package browsing, I could not find any packages that contained binary files.
The manifest format used by npm records when a file is binary, but I could not find any packages that contained a binary file in order to test how they are rendered

# Rationale and Alternatives

> - Why is this design the best in the space of possible designs?
> - What other designs have been considered and what is the rationale for not choosing them?
> - What is the impact of not doing this?

## On-demand vs stored contents

We must either decompress files on demand through a proxy or serve them from a storage bucket that contains all package contents already expanded.

Rubygems.org already handles considerable traffic, serving the entire ruby ecosystem with edge-cached packages. However, this does not necessarily translate into similar access patterns for browsing the source code contained in packages. Until the feature is released we can only guess its popularity and usage.

Web crawlers, if allowed to access the files, could ultimately force the on-demand system to decompress every file on a regular basis or all at once. This could render the additional complexity of a decompression proxy moot.

We therefore suggest the approach of batch decompressing all, or a suitable subset of gems, before launching the feature.
This allows control over the process and permits running it outside of the request/response cycle.

Alternative: We could start this feature on a few hand-selected gems that receive higher traffic. Statistics would be gathered to measure how often the feature is accessed.

Alternative: The limitation of testing a few high traffic gems will be the "long-tail" of smaller gems. It's conceivable that most of the gem content traffic will come from a small number of hits on a large number of different gems. This is also the situation that would cause the most load on a content proxy. An alternative approach to testing for this long-tail is to create the UI for the feature on rubygems.org, but have most of the gems respond with a placeholder. This will indicate how often a feature is clicked but not how many files would have been accessed.

Newly uploaded gems would be decompressed and stored when they are uploaded.

## Manifests

The file structure and file metadata of the gem contents is not currently stored by rubygems.org. Serving gem contents requires listing the files that can be accessed and referencing the locations of those files.

We could store the gem's manifest in the database, as a file on s3, or directly access the directory structure in the S3 bucket at request time.

The manifest should be able to return information about each file: content type, size, file_path, checksum, binary, file mode.

## Binary files

A user will want to know the full contents of a gem, including any binary files.
Many gems include compiled binaries for multiple architectures.
However, viewing the binary files in the browser or command line is not useful.

If a user wants to confirm the contents of the gem with their local copy, they could compare checksums.
This indicates that providing checksums of files is probably useful for binary files.

Downloading an individual binary file is of limited utility since most gems are not so large that extracting a single file from a gem is better than downloading the entire gem. In a gem like libv8-node, the one binary file is almost the entire download size of the gem. In this case, the gem is 30MB, and the individual uncompressed binary is 99MB out of the 100MB uncompressed gem. If we assume this is representative of most gems with binaries, it's a better idea to download the whole gem rather than the individual binary.

We therefore decided that including binary files in the manifest, with a checksum and size, is the best way to show that the file is present without serving the file from the content cache.

## Detecting binary files

Detecting binary files is another challenge. As we process the files we probably want to remove binaries from hosting.

One possible approach is the `file` command:

    $ file -b --mime-encoding image.png
    binary
    $ file -b --mime-encoding file-with-emoji.json
    utf-8
    $ file -b --mime-encoding file.rb
    us-ascii

The limitation here is that we must find all encodings we consider to be plain text.
We could combine, or instead use `file --mime-type` which returns the following for the above files:

    image/png
    application/json
    text/x-ruby

Clearly there is a need to come up with a comprehensive list of types.

Another alternative is to use grep, which detects binary files. The advantage of grep is that it's binary heuristic is likely to be familiar to most developers. I found this command and it seems to behave as expected.

    grep -Hm1 '^' < file.rb | grep -q 'Binary'

This exits 0 for binary files, 1 for searchable text files.

Grep offers potentially a more complete solution without compiling a list of mime types, but I don't yet know if there will be speed implications that could slow the process.

## File path urls vs hash urls

Depending on how we serve the files, we could host the files by file path within version or host them at a generic checksum hash.

There is one precedent for each version. NPM hosts files at a hex key. CPAN hosts files by package and path.

Hashing the files and storing each file at its hash would allow deduplication. At least some of the files in a gem will not change between versions of a gem, and although most gems will not contain the same file as other gems, there will be some cases where this is true.

Storing the checksum in the manifest will allow us to identify files by checksum.

One challenge of checksum based shared storage is yank gems. Not all files from a gem could be deleted from the content bucket upon yank. It would be necessary to count the number of times a file was referenced by a manifest.

Depending on how often gems are yanked, this could add an additional layer of complexity to the gem yanking process, requiring every file to be reference counted and deleted individually. This is a one time process per gem yank, but it could add more load and time required when deleting gems.


# Unresolved questions

> - What parts of the design do you expect to resolve through the RFC process before this gets merged?
> - What parts of the design do you expect to resolve through the implementation of this feature before it is on by default?
> - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

---

- How much storage will be required? Is there a cost limit? When do we reach a trade-off between complexity and cost?
- Do we obscure filenames behind a hash? Does it provide a benefit, discourage misuse? Does it make serving the files easier or harder?
- How do we display binary files? Do we display that they are in the package but not link to download them? Do we allow them to be downloaded? Could we reduce risk of abuse by not hosting binary files, thus limiting the potential by sharply reducing the types of files that would be hosted for free?
- Do we expand gems when they are first accessed or expand everything at the start and as uploaded?
- Do we implement a size or content type constraint to save space? Don't serve binaries or files larger than a few MB?
- How much time should be spent on protecting from abuse in the initial release? Do we respond as needed? Do we already have tools in place to detect and block abuse? Is it even going to be a problem if we limit our exposure only to text files?
