# Restricting Document Access of Same Origin Documents
An explainer to define ability to restrict access to frames of same origin documents.

## Motivation

Allowing cross-document DOM access has made the web very complicated. Especially since
it can go across origins due to [document.domain](https://developer.mozilla.org/en-US/docs/Web/API/Document/domain).
We have to support this forever to not break the web, but wouldn't it be nice if
individual pages could opt themselves or their frames into a simplier mode which
didn't allow cross-document access? Then everything would be simpler, more secure.
The well-defined [postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
and friends could only be used for cooperating between two frames that opt into this policy.

## Goals

- Be able to embed iframes that have same-origin as other frames in the frame tree but
  not be able to directly script them.
- Have a same-origin iframe with other iframes be in a separate event loop.

## Non-goals

- Be able to isolate cross-origin frames in different processes (ie. Not a solution for
https://github.com/whatwg/html/issues/4175).

## Proposal

Sandbox flags work in dropping certain features that an iframe has (eg. scripting,
fullscreen, top level navigation or opaque origins). A sandboxed frame by default
is a opaque origin but there are cases where you really want to preserve the origin
of the original document. For example sandboxed opaque origins won't send CORS
Origin headers. Access to shared cookies and local storage are sometimes
necessary inside an iframe as well.

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

The proposal is to support a Feature Policy `document-access` that prevents
a frame from reaching into the DOM of any other frame. The frame will fail
same origin checks on the JS bindings security perspective.

To implement this the [isPlatformObjectSameOrigin(0)](https://html.spec.whatwg.org/#isplatformobjectsameorigin-(-o-))
needs to be change so that if the current settings object contains a feature
policy that restricts access, then return false.

## Defined Cross Origin properties
The following properties which are cross origin properties would be allowed on same
origin documents that had the `document-access` policy restricted.

- [postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
- [location](https://developer.mozilla.org/en-US/docs/Web/API/Location)
- [blur](https://developer.mozilla.org/en-US/docs/Web/API/Window/blur)
- [focus](https://developer.mozilla.org/en-US/docs/Web/API/Window/focus)
- [close](https://developer.mozilla.org/en-US/docs/Web/API/Window/close)
- [parent](https://developer.mozilla.org/en-US/docs/Web/API/Window/parent)
- [opener](https://developer.mozilla.org/en-US/docs/Web/API/Window/opener)
- [top](https://developer.mozilla.org/en-US/docs/Web/API/Window/top)
- [length](https://developer.mozilla.org/en-US/docs/Web/API/Window/length)
- [frames](https://developer.mozilla.org/en-US/docs/Web/API/Window/frames)
- [closed](https://developer.mozilla.org/en-US/docs/Web/API/Window/closed)
- [self](https://developer.mozilla.org/en-US/docs/Web/API/Window/self)
- [window](https://developer.mozilla.org/en-US/docs/Web/API/Window/window)

## Why can't you just use different origins?

While it is possible to get the same behavior using different subdomains
for each iframe it is not enforcable from a framework since individual
iframes could navigate themselves.

Consider [AMP iframe](https://www.ampproject.org/docs/reference/components/amp-iframe)
which requires the iframe (embedee) to be an another domain than the
container (embedder). This is a [policy](https://github.com/ampproject/amphtml/blob/master/spec/amp-iframe-origin-policy.md)
so that the embedee does not become dependent on the embedeer and can
be served from anywhere. ie. If the embedee was placed inside another
embedder it would function exactly the same if embedder and the second
embedder were different top level domains.

With the adoption of [signed exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)
it is believed that enforcing this different origin policy on the
[AMP cache](https://github.com/ampproject/amphtml/issues/20848) will no longer
be enforcable.

## Example

```html

<iframe allow="document-access 'none'" src="iframe.html"></iframe>

```

Alternatively it can be combined with sandbox flags to drop sandbox flags:

```html

<iframe sandbox="allow-scripts allow-same-origin" allow="document-access 'none'" src="iframe.html"></iframe>

```

## Security Considerations

This specifically fixes cases where a web developer might create a sandboxed
iframe with `sandbox="allow-same-origin allow-scripts` expecting that it provides
some type of security but it doesn't if it is the same origin. Since the
same origin document can reach into the iframe and adjust the sandbox flags
and reload itself. As once the iframe is navigated to a new location the new
sandbox properties are be applied.

## Proposed HTML spec changes

These changes will hopefully be formally specified into the HTML spec via
[PR 4606](https://github.com/whatwg/html/pull/4606) which also incorporates
[PR 4617](https://github.com/whatwg/html/pull/4617).

The changes specifically involve the lookup of the Agent Cluster and the
determination if two platform objects are the same origin.

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

