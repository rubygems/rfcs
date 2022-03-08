- Feature Name: A new gem signing and verifying mechanism
- Start Date: 2022-01-28
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

https://github.com/rubygems/rfcs/blob/master/0000-template.md 

**_As this is a lengthy document, you may find [Github’s automated Table of Contents feature](https://github.blog/changelog/2021-04-13-table-of-contents-support-in-markdown-files/) helpful to navigate._**

## Summary

Gem signing as it exists today is unwieldy and little-used, even though signatures form a vital part of ensuring the security of software supply chains. This RFC proposes a replacement system for signing gems and verifying gem signatures. The new scheme will be based on [sigstore](https://www.sigstore.dev/), a widely-backed open source service for creating and storing signature information in a public transparency log. This functionality would be rolled out in several phases to smooth adoption. Ultimately, we intend to make signing and verifying gems an everyday experience, analogous to how Let’s Encrypt has made TLS certificates simple and ubiquitous.

## Motivation

A Ruby program is no more secure than its gems. An important risk to the ecosystem is that gems are unsigned, leaving them without valuable guarantees of authenticity (that the gem was signed by whomever claimed to have signed it) and integrity (that the gem was not altered between signing and verifying).

Gem signing is already possible, but in its current incarnation is almost entirely ignored. Of the top 10,000 most-downloaded gems, fewer than 1% have current, valid signatures. These gems are signed by just 65 identities, a miniscule fraction of the thousands of gem owners registered on rubygems.org. ([Source for data](https://docs.google.com/spreadsheets/d/1emSn0fg-cwUrdBcGHgGGLymeVN1rwiCr9DnPmBnmCwk/edit#gid=1832658119))

Why is gem signing so widely ignored? We believe that there are several contributing factors:
* Gem signing is a largely manual process, requiring the gem maintainer to take several steps, including generating a self-signed certificate and a private key.
* This private key needs to be safely stored and protected by the maintainer.
* Since gem signatures rely on self-generated certificates, there is no chain of trust to rely on. This means that each maintainer’s certificate needs to be manually trusted by each user. The burden of trust is quite high, as certificate authors may use _any_ email address as the subject.
* Gem certificates expire after 1 year by default. Once a certificate has expired, associated signatures can no longer be verified.

([More details on gem signing’s current design can be found here](https://docs.google.com/document/d/1wUQR6Y0-6fopjh0ot-ByHXvgYI2Q2MCh3Z96WQcsU_Q/edit#)).

This combination of factors means that both gem maintainers and gem users are disincentivized to either sign gems or verify gem signatures. Indeed, [Rubygem’s own security documentation](https://guides.rubygems.org/security/#general) notes that:

> … this method of securing gems is not widely used. It requires a number of manual steps on the part of the developer, and there is no well-established chain of trust for gem signing keys. … The goal is to improve (or replace) the signing system so that it is easy for authors and transparent for users.

In an ideal world, gem maintainers would find gem signing safe and trivial, so that they are inclined to sign their gems. Gem users would be able to rely on nearly all gems being signed, giving them increased confidence that they are using gems that have neither been falsified nor tampered with. In order to achieve the ideal world, we need an easier method of signing gems and verifying signatures that are valid forever.

## Guide-level explanation

This RFC introduces changes in technology and policy. To ease adoption, this RFC proposes three main rollout phases:

1. **Opt-in build-time signing of gems.** This allows us to test, polish and harden signing and verifying without disrupting all existing workflows. During this phase maintainers will need to deliberately choose to sign their gems, and they can be nudged with a message to do so at build time. Similarly, users will need to deliberately verify gem signatures, and we can also nudge them to do so. The `gem cert` command will be deprecated.
1. **Opt-out build-time signing of gems.** At this phase we will have gained experience and confidence with the new system. The RubyGems client will switch policy to make gem signing the default behavior, meaning that deliberate effort is needed to _not_ sign a gem. We expect this would create a rising tide of signed gems on rubygems.org as gem maintainers release new versions. Verification would happen automatically, but the lack of a signature would not be fatal. Gem authors would be able to set a policy that all their gem releases must be signed ([analogous to MFA configuration](https://guides.rubygems.org/mfa-requirement-opt-in/)). We would introduce a policy that the top most-downloaded gems must be signed ([analogous to the proposal for MFA](https://github.com/rubygems/rfcs/pull/36)). The `gem cert` command would be removed.
1. **Opt-out install-time verification of gem signatures with strict policy.** At this phase signature verification is automated and gem signatures are the norm. We change verification policy to require signatures, treating the lack of a signature as an error. Users would be able to suppress this behavior on a gem-by-gem basis. This would provide further incentive for gem maintainers to sign their gems, and it would mean that the absence of a signature is a meaningful security signal. The server would warn older clients that unsigned gems will be subject to strict policy.

In Phase 1, signing will be opt-in, meaning that signatures will require `build --sign` and verification will require `build install --verify-signatures` (or equivalent `gem signatures` commands). But by Phase 3, signing has become default in `build` and verifying is default in `install`. This guide explanation will assume that we have reached Phase 3, as this is the most complete scenario.

### Signing a gem at build time

Gems are automatically signed at build time using the `gem build` command. For example, to build and sign `foo`:

```console
$ gem build foo
Sign your gem by selecting an identity provider in your browser. Press Ctrl-C to abort.
Gem `foo` signed as person@example.com
```

The `gem build` command opens a browser for the signature authentication flow (see [_Signature authentication flow_](#signature-authentication-flow)).

Before you build the gem, you must ensure that your email address is listed in the gemspec's [`email`](https://guides.rubygems.org/specification-reference/#email) field.

You may also use `gem build --sign` as in previous versions of RubyGems, but this is no longer necessary to trigger the signature process.

If you do not wish to sign a gem when building, you can use the `--no-sign` parameter. For example, to build `foo` without signing it:

```console
$ gem build --no-sign foo
WARN: gem `foo` was not signed. Clients > 9.9.99 will not install your gem by default. You can sign the gem with the `gem signatures` command.
```

You may then perform signing separately, as described in the section [_Signing a gem after build time_](#signing-a-gem-after-build-time). This functionality is intended for maintainers of multiple related gems, who may wish to build a number of gems and then sign them in a single operation.

You should _always_ sign your gems, whether at build time or afterwards. Unsigned gems cannot be verified by signature, presenting a higher security risk to end users. For this reason, unsigned gems are treated as an error at installation by default.

#### Signature authentication flow

As part of the `gem build` command, the gem client will open the sigstore login page using your system web browser. To sign a gem, you must choose one of the identity providers that sigstore supports.

<img width="60%" alt="sigstore sign-in" src="https://user-images.githubusercontent.com/33674553/151442880-563d3eaf-e965-4d98-88c9-b8c0dc1cbaed.png">
If you are not already signed in with your selected provider, you will need to log in.

<img width="60%" alt="Sign in to GitHub" src="https://user-images.githubusercontent.com/33674553/151442855-bc68dbcd-757d-40be-b38b-88d2be61e386.png">

You then need to grant sigstore permission to request your email address from your provider account. This step only occurs on your first attempt to sign a gem using this provider. Only provider-verified email addresses may be used to sign gems.

<img width="60%" alt="Authorize sigstore" src="https://user-images.githubusercontent.com/33674553/151442864-51d311fd-a740-4ba5-b052-36c8fb25f6de.png">

Once the authorization step is complete, you may close your browser tab. The gem signing flow completes without further input.

### Signing a gem after build time

Gems can be signed after build time by using the `gem signatures --sign GEMNAME`. For example, to sign gem `foo` after it has been built:

```console
$ gem signatures --sign foo
Sign your gem by selecting an identity provider in your browser. Press Ctrl-C to abort.
Gem `foo` signed as person@example.com
```

Before signing a gem, ensure that the email address you use to sign into your identity provider also appears in your gemspec's `email` field. The gem client will return an error otherwise.  Upon installing the gem later on, only signatures associated with emails listed in the gemspec will be considered valid.

Multiple gems can be signed in a single command. For example, to sign `foo` and `bar`:

```console
$ gem signatures --sign foo bar
Sign your gem by selecting an identity provider in your browser. Press Ctrl-C to abort.
Gem `foo` signed as person@example.com
Gem `bar` signed as person@example.com
```

This functionality is intended for maintainers of multiple related gems, so they do not need to go through the browser-to-CLI flow multiple times.

Otherwise, using `gem signatures --sign` has the same signing behavior as `gem build`, including the browser authentication flow.

### Verifying a gem at install time

Gem signatures are automatically verified during installation. For example, to install and verify `foo`:

```console
$ gem install foo
Fetching foo-1.0.0.gem
Verified signature(s) for foo-1.0.0.gem
Successfully installed foo-1.0.0
[… documentation output …]
1 gem installed
```

You may also use `gem install --verify-signatures` as in previous versions of RubyGems, but this is no longer necessary to trigger the verification process.

Multiple gems can be installed in a single command, and each gem will be independently verified. For example, to install two signed gems `foo` and `bar`:

```console
$ gem install foo bar
Fetching foo-1.0.0.gem
Fetching bar-1.0.0.gem
Verified signature(s) for foo-1.0.0.gem
Verified signature(s) for bar-1.0.0.gem
Successfully installed foo-1.0.0
Successfully installed bar-1.0.0
[… documentation output …]
2 gems installed
```

Some gems are unsigned. If a gem is unsigned, the installation will fail with an error message. For example:

```console
$ gem install unsigned-foo
ERROR: `unsigned-foo` is not signed.
```

This behavior can be suppressed on a per-gem basis. For example:

```console
$ gem install unsigned-foo unsigned-bar --no-verify-signature unsigned-foo --no-verify-signature unsigned-bar
Fetching unsigned-foo-1.0.0.gem
Fetching unsigned-bar-1.0.0.gem
WARN: unsigned-foo-1.0.0 is unsigned
WARN: unsigned-bar-1.0.0 is unsigned
Successfully installed unsigned-foo-1.0.0
Successfully installed unsigned-bar-1.0.0
[… documentation output …]
2 gems installed
```

### Verifying a gem without installing

Gem signatures can be verified using the `gem signatures --verify GEMNAME` command. For example, to verify the signature of `foo`:

```console
$ gem fetch foo
Fetching foo-1.0.0.gem
Downloaded foo-1.0.0.gem

$ gem signatures --verify foo
Verified signature(s) for foo-1.0.0.gem
```

Verifying using `gem signatures --verify`, including verifying multiple gem signatures and the treatment of unsigned gems, is otherwise the same as `gem install`.

## Reference-level explanation

Our discussion at the reference level is assumed to be at Phase 1. This is for two reasons. Firstly, Phase 1 most closely resembles the [proof of concept RubyGems plugin](https://github.com/Shopify/ruby-sigstore) that we have developed from an early sigstore prototype. Secondly, we don’t expect radical changes in implementation for phases 2 and 3, except to improve performance by caching signatures. The explanations below will still be largely accurate throughout the rollout.

To achieve the “one click” gem signing experience, the RubyGems client orchestrates a multi-step flow with several services. The design of this flow means that signing keys are ephemeral and that signatures are published to a trusted public transparency log. First we will give background about ephemeral keys and transparency logs, then we will describe the gem signing and gem signature verification flows.

### Background concepts

#### Ephemeral signing keys

Historically, digital signature schemes have been based on public-key cryptography (also called asymmetric cryptography). The signer possesses a _private signing key_, which is not shared with anyone else. This key is used to sign artifacts, creating _signatures_. The signer then publishes a _public certificate_, which includes a _public verification key_. The public certificate binds information about the signer’s identity together with the public verification key into a single unit. The public certificate can be distributed widely. Verifiers can use the public certificate to verify that the private signing key was used to create a signature for a given artifact. In gem signing today, most certificates are self-signed, meaning that there is no independent root of trust for the claims of identity made in
the public certificates.

The security of this approach relies on the secrecy of the signing key. If the signing key is stolen, the thief can pose as the signer. If the key is lost, the signer will need to create a new signing key, publish a new certificate and convince end users to change which certificate they trust. When the key expires, or its associated public certificate does, the signer will also need to perform the same “key rotation” process. Because of the difficulty of rotating keys, it is typical for keys to be given a long lifetime, increasing the time during which they may be lost or stolen. Ultimately, the management of such keys is a high-stakes business.

Ephemeral keys change the traditional approach slightly. The private and public keys are still generated by the signer. But instead of creating a self-signed certificate which includes the public key, the signer sends their public key to a trusted Certificate Authority. The Certificate Authority then generates and signs a certificate with a short lifetime (by default, 20 minutes). The certificate is published to a certificate transparency log with a trusted timestamp. Since the private key and certificate are short-lived, they can only be used for one or a few signing operations before expiring. Once the signing is complete, the private signing key can be destroyed (in fact, it can be kept in memory for the whole process). This means that outside of the 20 minute window, it cannot be lost or stolen.

Because the certificate has been published with a trusted timestamp, it will always be possible to use the published certificate to verify a historical signature in the future. This means that certificate or key expiry becomes immaterial.

The practical upshot is that with an ephemeral key scheme, all the difficulties of managing, protecting and rotating private keys are removed. End users also have a simplified experience, because they need only trust the Certificate Authority’s root certificates, instead of each signed public certificate individually.

sigstore develops and operates [Fulcio](https://github.com/sigstore/fulcio), which is a Certificate Authority intended for software signatures. [This blog post gives a deep dive on Fulcio](https://blog.chainguard.dev/a-fulcio-deep-dive/).

#### Transparency logs

To be effective, both signatures and certificates need to be retrievable from a known, public, trustworthy service. This service needs to be the source of truth for all such signatures and certificates, so that meaningful conclusions can be drawn from the absence of entries in the log.

Transparency logs meet these requirements. A transparency log is a service which accepts log entries via an API. For each entry, and for the entire log taken at any given point in its history, the transparency log maintains a cryptographic structure which can detect the unauthorized addition of entries, the unauthorized deletion of entries, unauthorized modification of entries and entries appearing out of order. Put simply: attackers cannot secretly edit a transparency log. Once an entry is published, it stays published, without modification.

Transparency logs were first developed in the context of domain certificate issuance. Current browsers will not trust any certificate that does not have a corresponding entry in a certificate transparency log. This prevents malicious Certificate Authorities from issuing false certificates (e.g. for google.com).

sigstore develops and operates [Rekor](​​https://github.com/sigstore/rekor) (see [OpenAPI definition](https://github.com/sigstore/rekor/blob/main/openapi.yaml)), which is a transparency log intended for software signatures. This service can act as the source of truth for gem signatures.

### The signing flow

The signing flow can be understood at two levels of detail, a high level and a detailed level. The high level is sufficient for a first reading of this RFC. The detailed level is mostly intended for implementation reference/tech review and can be skipped (to [_Drawbacks_](#drawbacks)) until a second or later reading.

At a very high level the steps are:
1. Identify the signer via sigstore’s authentication gateway, using a browser
1. Generate a private signing key and public verification key
1. Use the signer’s identity and public key to get a public certificate from Fulcio
1. Sign the gem with the private key
1. Publish the signature and public certificate to Rekor
1. Discard the private key

The above description is accurate, but abstracts many steps. If you are an end user, the high level description is sufficient to form a correct intuition of how signing works. For implementers, there is substantially more detail to grasp. This is not directly due to the design of signing, rather it is because signing ultimately relies on [OpenID Connect (OIDC) standards](https://openid.net/developers/specs/), which in turn rely on [OAuth2 standards](https://oauth.net/2/). OIDC and OAuth2 are designed to defend against a variety of attacks and to support a variety of use cases. While the detailed description seems to involve many steps, it is important to remember that the bulk of interactions are actually part of a standardized, widely-adopted design.

<img src="https://user-images.githubusercontent.com/33674553/151225714-c7d356b7-5628-4eeb-902a-22c943a86adc.png" width="45%"/> <img src="https://user-images.githubusercontent.com/33674553/151225724-fb7300ee-06f4-4a73-ab6e-6acc6d0e5bd5.png" width="45%"/>

(Click on images for full size versions).

Steps 1 through 10 follow the standard OpenID Connect flow. The gem client is deemed public, and therefore uses no persistent client secrets. We use the OAuth 2.0 authorization code grant type (see [Authorization Code Grant](https://oauth.net/2/grant-types/authorization-code/)) with PKCE (see [Proof Key for Code Exchange](https://oauth.net/2/pkce/)) in order to acquire an authorization code. We then trade the authorization code for the identity token required by Fulcio.

The process is:

1. Read the OIDC configuration from a well-known endpoint in sigstore’s authorization gateway:

   ```
   GET https://oauth2.sigstore.dev/auth/.well-known/openid-configuration
   ```
   
   Abbreviated response, with the fields we care about:
   	
   ```jsonc
   {
     "issuer": "https://oauth2.sigstore.dev/auth",
     "authorization_endpoint": "https://oauth2.sigstore.dev/auth/auth",
     "token_endpoint": "https://oauth2.sigstore.dev/auth/token",
     "jwks_uri": "https://oauth2.sigstore.dev/auth/keys",
     // …
   }
   ```

1. In a separate thread, listen on an available port for an upcoming authorization code redirect response, for example:

   ```
   (LISTEN) http://localhost:58316
   ```

1. Generate a random nonce parameter, state parameter, and PKCE challenge parameter. In this example:

   ```
   PKCE value: 791008a63f93cac8893eac7268bf347b791ad64d290a6b68
   PKCE challenge: wwAV-EOiSQjU6lCPXgVDXrwojdHcZ7a9TRvCGEcQgVk
   ```

1. Open a browser and navigate to sigstore's authorization gateway endpoint, sending the values from the previous step:

   ```
   GET https://oauth2.sigstore.dev/auth/auth?client_id=sigstore&code_challenge=wwAV-EOiSQjU6lCPXgVDXrwojdHcZ7a9TRvCGEcQgVk& code_challenge_method=S256&nonce=c71d4fcc0ef8c6379a89682c79a8dc07&redirect_uri=http%3A%2F%2Flocalhost%3A58316&response_type=code& scope=openid%20email&state=7e1a05b6aa2c1f81f891d4c5f682bbed
   ```
 
   This is where the user will see a web page asking them to select an identity provider.

1. The user selects an identity provider, signs in, and grants sigstore access to their account's email address. This only needs to be done once, after which identity providers will cache a login session using browser cookies.

1. The identity provider makes a GET request for the redirect URI provided by the gem client. The identity provider sets a new authorization code parameter and sets the state parameter to the original value which was provided by the client in step 4. For example:

   ```
   GET http://localhost:58316/?code=njigmb5yr6z7nc4x3xz3vznw3&state=7e1a05b6aa2c1f81f891d4c5f682bbed
   ```

1. Receive the authorization code and state parameters in the listening thread, join with the main thread, and verify that the state parameter sent in step 4 and returned in step 6 has not changed between 4 and 6.

1. At the sigstore authorization gateway’s token endpoint, trade in the authorization code for an access token:

   ```
   POST
   https://oauth2.sigstore.dev/auth/token?grant_type=authorization_code&code=njigmb5yr6z7nc4x3xz3vznw3&   code_verifier=791008a63f93cac8893eac7268bf347b791ad64d290a6b68
   Authorization: "Basic sigstore:"
   ```

   The sigstore token endpoint responds with an identity token, which is a signed JSON web token (JWT) containing several claims about the provider-authenticated user.

1. Retrieve the JSON web keys (JWK) hosted at sigstore’s authorization gateway JWK endpoint. These may be cached by the client until new key ids (kid) are encountered in a JWT. For example:
 
   ```
   GET https://oauth2.sigstore.dev/auth/keys
   ```

   ```jsonc
   {
     "keys": [
       {
         "use": "sig",
         "kty": "RSA",
         "kid": "ff3d847f3c332c9fd856da9391e4f8ae1b82f66c",
         "alg": "RS256",
         "n": "xYQv4S7lc1hOo[...]1pcVjr6zJBhck5w",
         "e": "AQAB"
       },
       // …
     ]
   }
   ```

1. Validate the token signature using the appropriate JWK, then validate the claims (e.g. issuer, expiry, etc).

   The end result is a valid OIDC id token. sigstore's certificate authority, Fulcio, requires this token before it issues a code signing certificate chain. 

   This concludes the OIDC portion of the flow. The remaining steps cover the main act: signing the gem.

1. Read the verified email address from the token. Verify that the email address is listed in the gemspec's `email` field. If the signer is not listed as a maintainer, fail the operation, as there will be nothing linking this individual to the gem version they are signing.

1. Create a key pair: a private key for signing and a public key for adding to a public certificate.

1. Sign the email address' SHA256 digest with the private key to prove possession of the private key. We will refer to this as the `proof` in the next step.

1. Request a code signing certificate from Fulcio. Use the id token as the bearer token. The following example uses some pseudocode for brevity:

   ```
   POST https://fulcio.sigstore.dev/api/v1/signingCert
   Authorization: "Bearer <the JWT>"
   {
       publicKey: {
         content: Base64.encode64(pub_key.to_der),
         algorithm: "ecdsa",
       },
       signedEmailAddress: Base64.encode64(proof),
   }
   ```

   The response is a PEM-encoded pair of X.509 certificates: the code signing certificate with the user's email address as the subject alternative name, and Fulcio's own issuing certificate:
   
   ```
   -----BEGIN CERTIFICATE-----
   MIIDdjCCAvygAwIBAgITRT47a4MaF80DGP/Rcbt5eis4RjAKBggqhkjOPQQDAzAq
   MRUwEwYDVQQKEwxzaWdzdG9yZS5kZXYxETAPBgNVBAMTCHNpZ3N0b3JlMB4XDTIy
   [rest of signing certificate]
   -----END CERTIFICATE-----
   
   -----BEGIN CERTIFICATE-----
   MIIB+DCCAX6gAwIBAgITNVkDZoCiofPDsy7dfm6geLbuhzAKBggqhkjOPQQDAzAq
   MRUwEwYDVQQKEwxzaWdzdG9yZS5kZXYxETAPBgNVBAMTCHNpZ3N0b3JlMB4XDTIx
   [rest of issuing (root) certificate]
   -----END CERTIFICATE-----
   ```

1. Compute a SHA256 digest of the entire .gem file's contents, then sign the digest using the private signing key.

1. Submit the signature and public signing certificate to Rekor before the signing certificate expires (example uses pseudocode):

   ```
   POST https://rekor.sigstore.dev/api/v1/log/entries
   {
    kind: "hashedrekord",
    apiVersion: "0.0.1",
    spec: {
      signature: {
        format: "x509",
        content: Base64.encode64(signature),
        publicKey: {
          content: Base64.encode64(cert_chain_pem),
        },
      },
      data: {
        hash: {
          algorithm: "sha256",
          value: gem_digest,
        },
      },
    },
   }
   ```

The gem signature and certificate chain are stored together in an immutable Rekor log entry, such as https://rekor.sigstore.dev/api/v1/log/entries/a9bd97ee5453ce44525069e8ff8703555bc28d9f1553b9745bd6cdff3732bb87. In the signature verification flow, we will be able to locate this entry by searching Rekor using the .gem file's digest.

### The verifying flow

The signature verification flow occurs at gem installation time. In our proposed Phase 1, verification is opt-in, and the signature entry is only hosted on Rekor itself. By Phase 2, we anticipate that rubygems.org will keep a copy of the signature as well, and that clients will download it alongside the gem itself, decoupling the install experience from any Rekor queries on the happy path.

The following examples illustrate Phase 1.

<img src="https://user-images.githubusercontent.com/33674553/151441714-5b69f814-39b4-41e1-9dbc-b8241dfb83b6.png" width="45%"/>

(Click on image for full-sized version).

1. Query Rekor for a list of log entry UUIDs for a single gem file digest. 
   
   ```
   POST https://rekor.sigstore.dev/api/v1/index/retrieve
   {
     hash: "sha256:342024b59f3b8fe1a37efce6167023bc368f71ca01779ae81e78b2f4aca376be"
   }
   ```
   
   The response is an array of entry UUIDs for the log entries matching the digest. (Note: Rekor uses its own format for UUIDs, not IETF RFC4122 UUIDs.)
   
   ```json
   ["a9bd97ee5453ce44525069e8ff8703555bc28d9f1553b9745bd6cdff3732bb87"]
   ```

1. Retrieve the log entries by their UUIDs.

   ```
   POST https://rekor.sigstore.dev/api/v1/log/entries/retrieve
   {
     entryUUIDs: ["a9bd97ee5453ce44525069e8ff8703555bc28d9f1553b9745bd6cdff3732bb87"]
   }
   ```

   The response is an array of log entries (abbreviated), keyed by the log entry UUIDs we submitted:
   
   ```jsonc
   [
    {
      "a9bd97ee5453ce44525069e8ff8703555bc28d9f1553b9745bd6cdff3732bb87": {
        "attestation": {},
        "body": "eyJhcGlWZXJza[...]FMwdExRbz0ifX19fQ==",
        "integratedTime": 1644951590,
        "logID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d",
        "logIndex": 1408321,
        "verification": {
          "inclusionProof": {
               //…
        	},
          "signedEntryTimestamp": "MEUCIQC0pBuO1sywbAsf1O96vpv[...]k2V2camfU/Zwqb8FoFUsJLFSV0="
        }
      }
    }
   ]
   ```

   The log entry's `integratedTime` field records the time of the signature's upload into Rekor. It is the verifiable point in time at which the certificate chain was required to be valid.
   
   The base64-encoded `body` field is a [_hashed rekord_](https://github.com/sigstore/rekor/blob/main/pkg/types/hashedrekord/v0.0.1/hashedrekord_v0_0_1_schema.json), a base type in Rekor. Decode it to reveal the signature and public signing certificate:
   
   ```json
   {
     "apiVersion": "0.0.1",
     "kind": "hashedrekord",
     "spec": {
       "data": {
         "hash": {
           "algorithm": "sha256",
           "value": "342024b59f[...]1779ae81e78b2f4aca376be"
         }
       },
       "signature": {
         "content": "gcshnwE7S9tF9sa8lQkTFS[...]x1ePN9ww7Tiuiko1rv4/P2Q==",
         "format": "x509",
         "publicKey": {
           "content": "LS0tLS1CRUdJTiBDRVJUSUZ[...]RVJUSUZJQ0FURS0tLS0tCg=="
         }
       }
     }
   }
   ```

   Then for each log entry:

1. Note that the `spec.signature.publicKey.content` field is in fact the X.509 public signing certificate, encoded in base64.

   The signer's email address is recorded as a subject alternative name in the signing certificate. If the email address does not appear in the gemspec's `email` field, i.e. its list of maintainers, then reject the log entry from further consideration. It is possible for a sufficiently capable person to manually submit a signature of their own without using the gem client, as the transparency log does not control who submits an entry for a given gem.  

   The issuing certificate is not included in the log entry. Retrieve it from the location referenced in the signing certificate's authority information access (AIA) extension:

   ```
   Authority Info Access: CA Issuers - URI:http://privateca-content-603fe7e7-0000-2227-bf75-f4f5e80d2954.storage.googleapis.com/   ca36a1e96242b9fcb146/ca.crt
   ```

   An issuing certificate does not change after creation.  It may therefore be cached on the gem client. Fulcio's root certificate will infrequently change.

1. Validate the certificate chain in a manner similar to how [`Gem::Security::Policy`](https://github.com/rubygems/rubygems/blob/master/lib/rubygems/security/policy.rb) does it today. However, use the Rekor log entry's `integratedTime` field (seen in step 2) to determine whether the signing certificate chain was valid at the time of signing. Recall how the signing certificate is only valid for a 20 minute period.

1. Extract the signature (`spec.signature.content`) from the hashed rekord, decoding the value from base64. Using the public key included in the signing certificate, verify the signature against the gem's digest, which we compute locally.

1. Rejoice!

## Drawbacks

We believe that the proposed approach is a sound and effective way to improve gem signing security and ease-of-use. However, the proposed approach relies on the introduction of a new service that is not controlled or operated by RubyGems (as well as relying on identity providers). This means that signing gems may be slowed if sigstore is slow, or prevented if sigstore or identity providers are unavailable. It also means that RubyGems relies on sigstore remaining financially and operationally viable in the long term.

A potential alternative is for RubyGems to fully operate its own sigstore infrastructure, independently of the public instances operated by the sigstore project. We don’t think this is advisable, as it will add to the operational and financial burden on RubyGems. Additionally, there is no guarantee that a self-operated sigstore infrastructure would be any more reliable.

While today sigstore is at an early stage of production readiness, we have confidence that in the long term sigstore’s infrastructure will be well funded and operated. It has funding support from the [OpenSSF](https://openssf.org/), a foundation which has [already collected 10 million dollars of funding from industry partners](https://openssf.org/press-release/2021/10/13/open-source-security-foundation-raises-10-million-in-new-commitments-to-secure-software-supply-chains/). We expect this funding to remain steady for the foreseeable future.

We are less concerned about unavailability risks at verification time. This is because we believe it will be possible to cache Rekor log entries into the RubyGems bucket. Essentially, a file containing the signature and signing certificate for a .gem would live side-by-side with the .gem. Clients could retrieve the signature and certificate from the RubyGems bucket, without needing to connect to sigstore (though retaining the option to do so if sufficiently paranoid). Since installation operations greatly outnumber builds, the impact of any sigstore service problems would be contained to gem authors.

## Rationale and Alternatives

Our overriding goal is to improve end-to-end software supply chain security. We see ubiquitous signing of gems as a critical element of achieving that goal. In turn we believe that signing requires improved ease-of-use and security guarantees in order to achieve ubiquitous signing.

The experience we outlined in the guide shows a greatly simplified user experience compared to the existing signing mechanism. Certificates do not need to be manually generated or manually added to trust stores. Keys don’t need to be stored, protected and rotated. Signatures don’t “expire”. Build and installation become automatic points for signing and verification. Overall, we believe this will make signed gems the normal case for almost all maintainers and users.

We see two major alternatives to our proposal: doing nothing, or implementing The Update Framework.

### Doing nothing

This includes possibly removing the existing gem signing system, as it is little-used and largely ineffective.

Doing nothing is not a credible alternative. Supply chain attacks are growing more numerous and they are likely to worsen indefinitely.

### Implementing The Update Framework

[The Update Framework](https://theupdateframework.io/) (TUF) is a robust design for securing software repositories against a variety of attacks. There have been previous efforts to implement TUF for RubyGems in the past, particularly in the 2013-2014 timeframe, but these were not completed.

We do not think that our approach and TUF are mutually exclusive. Instead they are complementary. Gem signing gives a guarantee of authenticity that TUF does not, and TUF can give guarantees of whole-repository integrity that gem signing cannot. We believe that implementing TUF is still desirable.

However we have focused on gem signing for three reasons. Firstly, we feel more confident in implementing sigstore-based gem signing than TUF. Signing isn’t simple, but it seems to us to be simpler as a first step. Secondly, we are most concerned about end-to-end authenticity and integrity, which TUF does not solve. Thirdly, we are familiar with, and active in, the fast-growing, highly active sigstore community. We know where to get help, guidance and expert review.

We believe that TUF should be revisited in the future, but doing so is outside the scope of this RFC.

## Unresolved questions

The proposal in this RFC is novel and moderately complex. We do not pretend that this RFC can completely describe every aspect of the work to come – and neither should it. Therefore we have left some questions open, expecting that answers will be found during development of the new signing mechanism. We may open further follow-up RFCs in the future.

### Private gem servers

Some organizations operate private gem servers. These can be fully internal caches of rubygems.org, they can be gem servers for serving commercially-licensed gems, or they can be private SaaS offerings for private gems. In all these scenarios, rubygems.org is the source of neither truth nor blobs. Instead APIs and buckets are independent from rubygems.org.

Any design we develop will need to take these users into account. Similarly, we will need to be cognizant that some organizations will choose to operate independent, private instances of sigstore infrastructure.

While we can probably ignore these considerations for Phase 1, by Phase 3 they will need to be resolved.

### New dependencies vs hand-rolling

OIDC and OAuth2 are non-trivial standards to implement, because each supports a variety of usecases and flows. In the development of our proof of concept, we have relied on gems which provide OIDC and OAuth2 capabilities, even though we use a fraction of those capabilities. We also leverage gems to work with JSON web tokens and JSON web keys. As a consequence, the proof of concept requires several dozen direct and transitive dependencies for a RubyGems client plugin.

If transplanted into the main client, this would be a substantial step increase over the current number of dependencies. However, the client, as a design principle, eschews dependencies as much as possible. This leaves us with a difficult balancing act. On the one hand, we want to maintain the principle of minimal dependencies. On the other hand, hand-rolling narrowly-targeted security code will be slow, complex and can introduce a heightened risk of security bugs.

### Other kinds of log entries

Build signatures are a form of attestation that assert authorship or co-authorship of a gem. But there is no need to limit RubyGems attestations to signatures alone. We foresee that all security-critical events could be stored in sigstore to ensure an immutable log independent of RubyGems infrastructure. These could be rubygems.org events such as Push and Yank. Daily TUF repository signatures could provide [snapshots](https://www.youtube.com/watch?v=PJ6b2eoq0NY&list=PLj6h78yzYM2NOCoaYcYbiAf4KPIF36T8t&index=12) of all gems. Gems could be endorsed by individuals who would attest what kind of inspection they had performed. Some work on the latter has been [contemplated by the in-toto community](https://github.com/in-toto/attestation/issues/77).

Ideally, these other log entry types would be developed in partnership with other ecosystems, to ensure monitoring consistency for application and infrastructure security teams.

### Alternative flows

The design outline above requires interaction with a browser. In some scenarios this will be undesirable or unworkable. We believe that alternative flows are technically feasible, but we have not prototyped such flows. Ideally such a design would be settled by the start of Phase 2. As this work develops, we expect to open a follow up RFC.

### Dealing with older gems and older clients

Current and historical gem versions will not be signed. Older clients will not be able to sign. Older clients will not be able to verify signatures.

By Phase 3, this means that gems built with older clients will be rejected by newer clients by default. This is one reason why we have suggested that the rollout be phased.

Some gems may never be updated, but will still be installed by a newer client. We will need to define a cutoff date, before which gem versions are subject to a lenient verification policy, after which stricter policies apply.

### Bundler

We foresee that Bundler will need to pass through arguments to RubyGems, for example `--verify-signature`. We have not prototyped this capability.

### Independence from large identity providers

There is concern that as it stands, the signing flow works by federating identity with large providers (currently Github, Google and Microsoft). Some members of the community would prefer not to authenticate with such providers.

An alternative flow using email verification links provided by a sigstore-operated service seems technically feasible, but has not been prototyped. We believe that this option would be suitable for maintainers unwilling to rely on major identity providers.

### Key trust and management

Key management and the selection of a key trust policy is notoriously frustrating for implementers. In this case, the question is how to distribute and how to derive trust for the root certificates of several sigstore components. We have not addressed this question in the RFC, but it would need to be settled by no later than the start of Phase 2.
