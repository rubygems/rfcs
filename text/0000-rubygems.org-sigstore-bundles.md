- Feature Name: RubyGems.org accepts Sigstore Bundles with Gem Pushes
- Start Date: 2024-10-24
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

RubyGems.org will add a new variant to the gem push endpoint that allows gems authors to include sigstore bundles with their gem pushes.
RubyGems.org will verify that the sigstore bundle is valid and signed by a trusted publisher configured for the gem.

# Motivation

This is one of the first steps to bringing signed cryptographic attestations of gem provenance to the Ruby ecosystem.
Allowing gem authors to attach provenance information to the artifacts they publish is a prerequisite for downstream consumers to be able to verify the provenanve of the gems they install.

# Guide-level explanation

[Sigstore](https://sigstore.dev) is a suite of projects that aims to make signing & verifying artifacts easy and secure.
Sigstore bundles are a way to package verification materials (signing certificate, proof of inclusion in a tamper-evident trust log, signed time source) along with information about an artifact. In other words, a sigstore bundle will allow you to verify "GitHub repo segiddins/mygem at commit 1234567 with github action workflow 'release.yml' attests to mygem-1.0.0.gem having sha256 hash abcd1234".
There will be a new way of uploading gems, along with an array of such attestations that RubyGems.org will verify and store.

# Reference-level explanation

## `POST /api/v1/gems`

This is the existing endpoint for gem pushes.
Currently, rubygems.org assumes that the entire body of the request is the gem to be processed.

We will update the endpoint to behave slightly differently when the `Content-Type` header is set to `multipart/mixed` or `multipart/form-data`.
In that case:

The request MUST include a part with name `gem` that contains the gem file. The `gem` part SHOULD have a `Content-Type` header of `application/octet-stream`.
The request MAY include a part with name `attestations` that contains sigstore bundles. The `sigstore` part MUST have a `Content-Type` header of `application/json` and MUST be a valid JSON array of sigstore bundle object.

RubyGems.org will verify that any provided attestations are valid, match the uploaded gem, and signed by a trusted publisher configured for the gem.

## `GET /api/v1/attestations/:full_name.json`

This is a new endpoint that will allow users to download the attestations for a given gem.
This endpoint will return a JSON array of sigstore bundle objects.

# Drawbacks

This is adding a second variant to a critical endpoint, which could increase complexity and maintenance burden.

# Rationale and Alternatives

- This is step one of integrating sigstore into the rubygems ecosystem. Starting by accepting uploaded attestations will give us the basis to test out the changes needed to both the publishing flow and the gem installation flow to verify attestations and their compliance with a given provenance policy.
- See also:
  - [Build Provenance for All Package Registries](https://repos.openssf.org/build-provenance-for-all-package-registries)
  - [Sigstore](https://sigstore.dev/)
  - [PEP 740 – Index support for digital attestations](https://peps.python.org/pep-0740/)

# Unresolved questions

- Out of scope for this RFC, but the next step is integrate verification of attestations into the gem installation process, both for bundler & rubygems
- Out of scope, but we will eventually wish to expose the ability to attach sigstore bundles to gems via `gem push` command.
