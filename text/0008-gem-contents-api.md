- Feature Name: gem_contents_api
- Start Date: 2022-12-20
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Provide an API for accessing gem contents. The gem contents could be rendered on rubygems.org or accessed directly view api calls.
Support security and enable future behavior that could verify gem contents against versions in source control.
The API should return the directory structure and contents of files inside the actual gem package.
The API should be available for any version of any gem that is currently available for download.

# Motivation

Ensuring the security and stability of applications requires reviewing the contents of dependencies.
Currently, to review the contents of a gem, you must either download the gem locally or find the gem's source online.

Online representations of the gem's source code have a number of shortcomings.
Good practices like changelogs, releases, source control tags, and version diffs do not guarantee that the online source is a reliable representation of the gem package.
Rubygems supports selectively omitting files from the packaged build, which means that even well maintained source may differ from the contents of the gem.
Gems are often built and released from local developer machines which introduces additional build environment variability.
More nefariously, a malicious gem publisher could publish source code that is different from gem package contents as an attempt to disguise malicious code.

Currently, the only way to inspect the true contents of a gem is to download the gem to a machine and examine the installed source.
Rubygems.org therefore has the opportunity to offer a trusted source of gem contents on the website and via an API that does not require downloading and extracting the gem package.
Offering a safe, official gem content browsing feature will support the security and transparency of Ruby gems and the apps built on them.

# Guide-level explanation

Access gem contents via the new contents url. (TODO: something like /gem-name-1.2.3/contents/files/path/to/file.rb or an obfuscated thing likes contents/files/abcdef1234567890)

A file and directory structure manifest is available at a url (TODO: something like /gem-name-1.2.3/contents/manifest.json)

Requests to the api may or may not 302 redirect to an S3 bucket or use fastly, depending on how we implement it.

Source should be returned raw inline in the body. Binary files should either not be served or should return in 

-----

Explain the proposal as if it was already implemented and released, and you were teaching it to another developer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how users should _think_ about the feature, and how it should impact the way they use Bundler. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing users and new users.

For implementation-oriented RFCs, this section should focus on how contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation

1. Expand the contents of gems
2. Store a manifest that indicates directory structure and file types (e.g. flagging binary files so they are not displayed in the browser)
3. Provide an API for users to access the manifest and load individual files.

### Expand the gems

The volume of traffic at rubygems.org is high enough that we must anticipate and build for high traffic from the start.

There are many options for exposing gem contents, from on-demand expansion to edge-cached block storage. On-demand expansion requires more processing time, whereas the storage requirements for uncompressed gems increases the total storage usage of rubygems.org.

Given that demand for this feature is unknown, the simplest approach is to expand all (or a subset of all) gems into a new S3 bucket.
Expanding all gems allows serving content to simply assume that a gems source is available in a consistent way.
If there is a way to create a subset of gems that covers an acceptable percentage of likely usage (ignoring sufficiently old gems or gems that have not been downloaded recently) then we could avoid storage costs for gem contents that will never be browsed.

The S3 bucket should be exclusively for expanded gem contents.
The contents of a gem will never change, so the storage should be treated as read-only except to remove yanked gems.

Another location should be used for the manifest. While file contents will never change, the calculated information inside the manifest may need to be updated as our information about the usage of this feature improves.

We propose a solution that uses block storage (S3) to store expanded source code.



This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks

There is potential for abuse. If rubygems.org now serves public versions of any file that anyone uploads, supported by an edge cache with high availability, it could enable bad actors to host highly available files for free. While this is already possible on github and in many places on the internet, it may become difficult or impossible to discover abuses. Rubygems.org currently only serves compressed archives uploaded as gems, so this change opens a new avenue for storing almost any file of any size.

Hosting uploaded content the raw form could expose rubygems.org to responsibility for DMCA takedown requests. This could already happen with the archives as is, but when the file contents are served by rubygems.org directly, it may become a larger issue. GitHub [indicates](https://github.blog/2022-01-27-2021-transparency-report/#:~:text=In%202021%2C%20GitHub%20received%20and%20processed%201%2C828%20valid%20DMCA%20takedown,our%20users%20to%20remove%20content.) that they complied with 1828 DMCA takedown requests in 2021. Rubygems.org could still face an increased burden when contents are exposed directly to the internet.

Examining npm's code browsing shows that they deliver a manifest that references files at a single file serving URL rather than serving files at their original path in the package. They use a 64 character hex string, possibly a SHA256 hash of the file, as a key to retrieve a file. There are probably many reasons for this approach. File paths could be long or could contain non-url friendly characters. It limits directly accessing files by obscuring their location. It makes the url for accessing files more uniform which might make access controls or caching simpler.

Limiting the serving of binary files could reduce potential abuse.
When reviewing npm package browsing, I could not find any packages that contained binary files.
The manifest format used by npm records when a file is binary, but I could not find any packages that contained a binary file in order to test how they are rendered

--

Why should we _not_ do this? Please consider the impact on existing users, on the documentation, on the integration of this feature with other existing and planned features, on the impact on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Rationale and Alternatives

## On-demand vs stored contents

We must either decompress files on demand through a proxy or serve them from a storage bucket that contains all package contents already expanded.

Rubygems.org already handles considerable twraffic, serving the entire ruby ecosystem with edge-cached packages. However, this does not necessarily translate into similar access patterns for browsing the source code contained in packages. Until the feature is released we will have only a guess of how popular the feature is. (

Web crawlers, if allowed to access the files, could ultimately force the on-demand system to decompress every file on a regular basis, making the additional complexity of a decompression proxy moot.
This would also thwart a caching proxy that decompresses to a permanent location on-demand if all files are crawled soon after the feature is released.

We therefore suggest the approach of batch decompressing all gems before launching the feature.
This allows control over the process and permits running it outside of the request/response cycle.

Newly uploaded gems would be decompressed and stored when they are uploaded.

## Manifests

The file structure and file metadata of the gem contents is not currently stored by rubygems.org. Serving gem contents requires listing the files that can be accessed and referencing the locations of those files.

We could store the gem's manifest in the database, as a file on s3, or directly access the directory structure in the S3 bucket at request time.

## Binary files

A user will want to know the full contents of a gem, including any binary files. However, viewing the binary files in the browser or command line is not useful.

Many gems include compiled binaries for multiple architectures. Knowing that the compiled binaries are present is useful. Viewing the binary data is less useful.

If a user wants to confirm the contents of the gem with their local copy, they could compare checksums. This implies that providing checksums of files could be useful for binary files.

Downloading an individual binary file is of limited utility since most gems are not so large that extracting a single file from a gem is better than downloading the entire gem.

The primary usage of the contents API is expected to be viewing or comparing plain text contents. Knowing that binaries are present and having access to their checksums should be enough to verify binary files.

-----

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions

- How much storage will be required? Is there such a thing as too much? Do we need to try to limit the costs?
- Do we obscure filenames behind a hash? Does it provide a benefit, discourage misuse? Does it make serving the files easier or harder?
- How do we display binary files? Do we display that they are in the package but not link to download them? Do we allow them to be downloaded? Could we reduce risk of abuse by not hosting binary files, thus limiting the potential by sharply reducing the types of files that would be hosted for free?


- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before it is on by default?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
