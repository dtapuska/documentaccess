# Restricting Document Access of Same Origin Documents
An explainer to define ability to restrict access to frames of same origin documents.

## Motivation

The use of [AMP iframes](https://github.com/ampproject/amphtml/blob/master/spec/amp-iframe-origin-policy.md)
currently enforce that iframes be hosted on different origins. The main goal of this is to ensure
that pages behave the same when served via the AMP cache vs directly from the origin.

Sandbox flags work in dropping certain features that an iframe has (eg. scripting,
fullscreen, top level navigation or opaque origins). A sandboxed frame by default
is a opaque origin but there are cases where you really want to preserve the origin
of the original document. For example sandboxed opaque origins won't send CORS
Origin headers. Same origin documents are also useful for cookies and local storage
to be shared.

It is desirable that a document be neutered from the rest of the frame tree and
treated as cross origin only for the JS bindings. The access through a
[window proxy](https://html.spec.whatwg.org/#windowproxy) should match the
cross origin rules even though the documents have the same origin.

For example:

A embeds two frames from B, both of which include content that's not as
trustworthy as it ought to be. One of the frames from B turns out to be malicious.
Today, it can takeover any other frames from B by walking the tree exposed
through window.top.frames. It is desirable that B could make that more difficult
by forcing the same origin-domain checks that enable DOM access to fail by setting
some policy.

## Proposal

The proposal is to support an Feature Policy `document-access` that prevents
a frame from reaching into the DOM of any other frame. The frame will fail
same origin checks on the JS bindings security perspective.

To implement this the [isPlatformObjectSameOrigin(0)](https://html.spec.whatwg.org/#isplatformobjectsameorigin-(-o-))
needs to be change so that if the current settings object contains a feature
policy that restricts access, then return false.

## Why can't you just use different origins?

TODO: Expand this section

There already is a way to get around the enforcement of the [same origin policy in AMP](https://github.com/ampproject/amphtml/blob/master/spec/amp-iframe-origin-policy.md#security-impact). With the adoption of
[signed exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)
it is more likely the different origins isn't enforced for [AMP](https://github.com/ampproject/amphtml/issues/20848).

## Example

```html

<iframe allow="document-access 'none'" src="iframe.html"></iframe>

```

Alternatively it can be combined with sandbox flags to drop sandbox flags:

```html

<iframe sandbox="allow-scripts allow-same-origin" allow="document-access 'none'" src="iframe.html"></iframe>

```

## Security Considerations

This specifically fixes cases where a sandbox iframe may allow same origin
access and allow scripting, allowing the iframe to adjust its own sandbox
properties. Once the iframe is navigated to a new location the new sandbox
properties will be applied.

## Alternatives Explored

Allow [document.domain = null](https://github.com/whatwg/html/issues/2757). Subtle
and is not delegated by parent frame but the embedee needs to assign it. So it is
subject to timing of the assignment of the `document.domain = null` value. This
assignment also had non-definitive rules around whether `same-origin` or
`same origin-domain` rules passed. Using the feature policy and adjusting
the `isPlatformObjectSameOrigin` algorithm clarifies this.

Implement another sandbox policy like same-origin but call it
same-origin-without-document-access. This itself is not useful for pages
that don't want to use sandboxes.

