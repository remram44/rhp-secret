A Proposal for Reportable Secrets
=================================

Introduction
------------

This documents proposes a new format for secrets, for example API key, to address the shortcomings of current systems.

Use of "secret tokens" is common across systems. Historically, each system has used their own format, but it is common to make them recognizable in some way. For example, Amazon Web Services uses an `AKIA` prefix for access keys, SendGrid uses an `SG.` prefix on API keys, PyPI uses a `pypi-` prefix for their tokens, GitHub a `ghp_` prefix, etc.

To prevent accounts being hacked, code platforms have been automatically detecting tokens and reporting them. For example, [GitHub's "secret scanning"](https://docs.github.com/en/code-security/secret-security/about-secret-scanning) checks uploaded code for tokens in a variety of formats and reports them to both the code's author and the API service, which can revoke the leaked token. However because each token and service is different, this is done each service giving GitHub a regular expression to identify their token, and an API endpoint they can use to report and revoke found tokens.

The problems with that approach are:

* While the API specified by GitHub is implemented by multiple services, it is useless to everyone except GitHub in practice
* Building the list of token formats and reporting endpoints is an unnecessarily manual process, and currently only GitHub possess the list
* Platforms (in particular code hosting platforms) not being able to implement scanning increases risk for users

In this document, I propose a way to fix those issues via a new unified secret format. The format would make it clear that the token is a secret, eliminating the need for multiple regular expressions, and would contain enough information to enable automated reporting, eliminating the need for a registry.

Alternatives
------------

Building an open registry of regular expressions and reporting endpoints, available to everyone and not only GitHub, would also work. It would allow a new code hosting platform to scan for all those services without having to be contacted by each one.

[RFC 8959](https://www.rfc-editor.org/rfc/rfc8959.html) introduced a `secret-token:` URI scheme to make tokens recognizable, however it does not help with reporting in any way. We could extend this format, however there is a risk of collision with existing tokens, and this specification would be hard to discover.

Using a URL is another option. For example Sentry uses the format `https://123456789@sentry.io/12345` (Sentry is self-hostable, and they chose to include the user ID, domain, and key in the same string they call DSN).

Design
------

The proposed format is:

```
rhp-secret:example.org/mysecretcontent
```

* The string `secret` makes this easily recognizable as a secret. Simply using `secret` might be too ambiguous, so I propose `rhp-secret` for "reporting host prefixed secret". This can be changed/improved. Using a unique string has the advantage of making this specification more discoverable.
* The next part is a domain name. This allows for easy discovery of the related service and its leak reporting endpoint.
* The rest of the string is service-specific, allowing each service to put whatever data they need there, for example a JWT token, a random string, a private key, etc.

The reporting protocol is close to [the GitHub one](https://docs.github.com/en/developers/overview/secret-scanning-partner-program#create-a-secret-alert-service), to avoid services having to implement a brand new endpoint. The endpoint is built from the included domain name by using the path `.well-known/report-leaked-secrets`, e.g. `https://example.org/.well-known/report-leaked-secrets`.

```
POST /.well-known/report-leaked-secrets HTTP/1.1
Host: example.org
Content-Type: application/json

[
  {
    "token": "rhp-secret:example.org/mysecretcontent",
    "type": "rhp-secret",
    "url": "https://github.com/remram44/rhp-secret/blob/master/README.md"
  }
]
```

Limitations and drawbacks
-------------------------

* This needs services to change their token format
* There is now a competing IETF scheme (though it hasn't seen much adoption)
* The format is kind of verbose (`rhp-secret:sendgrid.com/91on9SIkbUfSsfvfznkrVx.vAqI9VLmlANMdA7I5dc2TEwla_bkPCIK_6tFaq7FsTV` instead of `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`)
* You need to allow leaked tokens to be submitted from everywhere, not just GitHub. The security implications of this are not clear (e.g. potential for denial of service); this is not the way GitHub designed their protocol, as they recommend checking the included signature, but no motivation was provided
* The reporting mechanism needs a service to have a domain name and to deploy the endpoint at the root, so it might not work for a service used from an IP or deployed in a subdirectory of a shared host.
* GitHub provides a mechanism to report false-positives which is not part of this specification, though it could be added without breaking compatibility (as HTTP request header?).

Discussions
-----------

* https://news.ycombinator.com/item?id=25979908 2021-01-31
* https://news.ycombinator.com/item?id=26568651 2021-03-24
  * Asked PyPI developer why their reporting endpoint needed to be protected by a GitHub signature
    * He mentioned it might be used as an Oracle to check tokens, as that can be done without knowing the username (API use needs username + token)
    * Someone mentioned denial-of-service related to revoking someone's token, however possession of the token leads to worse attacks, and you can always commit it to GitHub to get it revoked
