# Table Of Contents
1. [Motivation](#motivation)
1. [Goals](#goals)
1. [Out of scope](#out-of-scope)
1. [Use Cases](#usecases)
   1. [News Aggregator](#newsaggregator)
   1. [Safe Frame Ads](#safeframeads)
   1. [Performance Guarantees](#slowiframes)
1. [Proposal](#proposal)
   1. [By Example](#byexample)
   1. [The Details](#details)
   1. [Pull Request](#pr)
   1. [Cross Origin Properties](#prop)
1. [Common Questions](#questions)
   1. [Just use origins?](#origins)
   1. [Use another sandbox flag?](#sandbox)
   1. [Comaptability Risks?](#risks)
   1. [What about optin in?](#optin)
1. [Security](#security)
   1. [Threat Model 1](#threat1)
   1. [Threat Model 2](#threat2)
1. [Alternatives](#alternatives)

# Restricting Document Access of Same Origin Documents
An explainer to define ability to restrict access to frames of same origin documents.

## Motivation  <a name="motivation"></a>

Allowing cross-document DOM access has made the web very complicated. Especially since
it can go across origins due to [document.domain](https://developer.mozilla.org/en-US/docs/Web/API/Document/domain).
We have to support this forever to not break the web, but wouldn't it be nice if
individual pages could opt themselves or their frames into a simplier mode which
didn't allow cross-document access? Then everything would be simpler.
The well-defined [postMessage()](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)
and friends could only be used for cooperating between two frames that opt into this policy.

## Goals  <a name="goals"></a>

- Be able to embed iframes that have same-origin as other frames in the frame tree but
  not be able to directly script them.
- Separate event loop per iframe for advanced scheduling.
- CORS Origin headers aren't serialized on sandboxed origins.
- Prevent `sandbox="allow-scripts allow-same-origin"` sandbox escaping.

## Out of scope  <a name="out-of-scope"></a>

- Be able to isolate cross-origin frames in different processes (i.e., not a solution for
https://github.com/whatwg/html/issues/4175).
- Separate cookie, local storage.

# Use Cases <a name="usecases"></a>

## News Aggregator <a name="newsaggregator"></a>

Consider an example of a news aggregator that embeds a series of iframes. It may
embed two documents from the same origin but it really doesn't want those two pages
to communicate via direct scripting with each other. With window.top those frames can
communicate with each other calling diretly into another frame and manipulating the
DOM (possibly within the the context of a user gesture).

## Safe Frame Ads <a name="safeframeads"></a>

[Safe Frame Ads](https://www.iab.com/guidelines/safeframe/) may place two ads from
different syndicate networks on the same origin. An ad should not be able to read
content from another ad (via window.name) thereby leaking some private information
about the user. Although there are solutions to prevent this it is a largely
adopted pattern we could help mitigate.

## Performance Guarantees <a name="perf"></a>

Being that the attribute isolates the iframe and the subtree in another Agent Cluster
(and thereby another event loop) performance guarantees come with that. AMP supports
`<amp-iframe>`, but the iframe has to be forbidden from scripting the
top-level page in order to ensure AMP's performance guarantees. This works fine when
AMP is served from a different origin than the origin of the iframe. However with
signed exchanges it is possible to serve both the main document and the iframe
document from the same origin thereby preventing the performance guarantees.
See details in the [amphtml issue](https://github.com/ampproject/amphtml/issues/20848).

# Proposal  <a name="proposal"></a>

## Example  <a name="byexample"></a>

```html

<iframe disallowdocumentaccess src="iframe.html"></iframe>

```

Alternatively it can be combined with sandbox flags to drop sandbox flags:

```html

<iframe sandbox="allow-scripts allow-same-origin" disallowdocumentaccess src="iframe.html"></iframe>

```

## The Details  <a name="details"></a>

Sandbox flags work bu dropping certain features that an iframe has (eg. scripting,
fullscreen, top level navigation or opaque origins). A sandboxed frame by default
is a opaque origin but there are cases where you really want to preserve the origin
of the original document. For example, sandboxed opaque origins won't send CORS
Origin headers. Access to shared cookies and local storage are sometimes
necessary inside an iframe as well.

It is desirable that a document be neutered from the rest of the frame tree and
treated as cross origin only for the JS bindings. The access through a
[window proxy](https://html.spec.whatwg.org/#windowproxy) should match the
cross origin rules even though the documents have the same origin.

For example:

A embeds two frames from B, both of which include content that's not as
trustworthy as it ought to be. One of the frames from B turns out to be malicious.
Today, it can take over any other frames from B by walking the tree exposed
through window.top.frames. It is desirable that B could make that more difficult
by forcing the same origin-domain checks that enable DOM access to fail by setting
some policy.

The proposal is to support an iframe attribute `disallowdocumentaccess` that prevents
a frame from reaching across the frame boundary. The frame will fail same-origin checks
from the JS bindings security perspective.

A new [agent cluster map](https://html.spec.whatwg.org/multipage/browsers.html#agent-cluster-map)
should be allocated when an iframe encounters the `disallowdocumentaccess` attribute instead
of using the one from the [browsing context group](https://html.spec.whatwg.org/multipage/browsers.html#browsing-context-group).
This will cause the [agents](https://html.spec.whatwg.org/multipage/webappapis.html#obtain-similar-origin-window-agent)
allocated to each execution context to be different. Since new agents will be allocated
these frames will have a separate event loop.

An additional change to [isPlatformObjectSameOrigin(0)](https://html.spec.whatwg.org/#isplatformobjectsameorigin-(-o-))
needs to be completed. The algorithm should check that the [Agent](https://tc39.es/ecma262/#sec-agents) is the same on 
the objects. If it is different that it restricts access, then return false.

## Pull Request against HTML Spec <a name="pr"></a>

A [pull request](https://github.com/whatwg/html/pull/4606) has been made against the HTML spec.


## Defined Cross Origin properties <a name="prop"></a>
The following properties which are cross origin properties would be allowed on same
origin documents that had the `disallowdocumentaccess` attribute applied.

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

# Common Questions  <a name="questions"></a>

## Why can't you just use different origins?  <a name="origins"></a>

While it is possible to get the same behavior using different subdomains
for each iframe it is not enforcable from a framework since individual
iframes could navigate themselves.

Consider [AMP iframe](https://www.ampproject.org/docs/reference/components/amp-iframe),
which requires the iframe (embedee) to be an another domain than the
container (embedder). This is a [policy](https://github.com/ampproject/amphtml/blob/master/spec/amp-iframe-origin-policy.md)
so that the embedee does not become dependent on the embedeer and can
be served from anywhere, i.e., if the embedee was placed inside another
embedder it would function exactly the same if embedder and the second
embedder were different top level domains.

With the adoption of [signed exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)
it is believed that enforcing this different origin policy on the
[AMP cache](https://github.com/ampproject/amphtml/issues/20848) will no longer
be enforcable.

## Why isn't this a sandbox feature?  <a name="sandbox"></a>

Sandbox flags have issues that they are all set by default so it is a very
restrictive policy. For example setting `sandbox` specifies that all
sandbox flags are set even new ones added in the future. 

Hypothetically if a sandbox flag was added for document access, a page that
only wants to restrict that would have to re-enable every sandbox flag now
and in the future. Since it isn't possible to determine what the future sandbox
flags would be it would mean that the author would have to update the sites
everytime new sandbox flags are added. This causes sandbox flags not to be the
ideal choice for this feature.

## Compatibility Risks  <a name="risks"></a>

Consider the case where A sets the `disallowdocumentaccess` attribute on B1.

```
      A
    /   \
   B1   B2
  /  \    \
 C1  C2    C3

```

B1 and B2 cannot directly manipulate one another. 
C1 and C2 can directly manipulate one another.
C1/C2 and C3 cannot directly manipulate one another.

B2's access to B1 is essentially limited to `postMessage`, but this is very
similar to A setting the `sandbox` attribute on B1 with the exception of
storage access (B1 and B2 do not derive a new opaque origin when
`disallowdocumentaccess` is set).

So while it is possible to write a web observable test to see this behavior
(try to access `window.location.href` but see that `postMessage` is successful),
the situations it creates web authors have already had to deal with because
A is controlling what is being embedded.

## What about specifically having each iframe acknowledge it allows this mode?  <a name="optin"></a>

A given iframe has no control over whether other iframes in the same frame tree are
loaded or not so requiring it to opt-in on the enhanced agent policy seems like
an additional barrier for adoption.

This functionality would like to be enabled in popular publishing libraries
such as [amp](https://amp.dev) in the definition of the
[amp-iframe](https://amp.dev/documentation/components/amp-iframe/). Since that
embeds an arbitrary number of origins, the impact of making such a change
wouldn't be possible if the iframes failed to load due to not "opting in".

# Security Considerations  <a name="security"></a>

This specifically fixes cases where a web developer might create a sandboxed
iframe with `sandbox="allow-same-origin allow-scripts` expecting that it provides
some type of security. While it does give some guarantee when an iframe
and the parent frame are on the different origins it behaves poorly when
they are the same origin. This is complicated for the web developer to understand
and cause some subtle security issues. i.e. The same origin document can reach into
the iframe and adjust the sandbox flags and reload itself. As once the iframe is
navigated to a new location the new sandbox properties are be applied.

## Threat model #1  <a name="threat1"></a>

### The actors
- **The user**. This is the human who relies on the user agent to deliver a good experience for them and protect them.
- **The user agent**. This is the web browser that tries protect privacy. 
- **Safe frame**. Old technology that tries to bring safe ads to users.

### The threat
1. Arbirtary content in [safe frames](https://www.iab.com/guidelines/safeframe/) can 
read location and other attributes on the document from the same origin.

### The attack
1. Create a malicious script that is embedded in the script of the safe iframe.
1. Walk frame tree trying to extract information from other ads in the same syndicate network.

## Threat model #2  <a name="threat2"></a>

### The actors
- **The user**. This is the human who relies on the user agent to deliver a good experience for them and protect them.
- **The user agent**. This is the web browser that tries protect privacy. 
- **Web developer**. Tries to use "allow-scripts allow-same-origin" sandbox attribute.

### The threat
1. A web developer assumes that placing content in a sandbox gives they some assurances.
1. The iframe might require allow-scripts and allow-same-origin to run correctly.
1. The web developer chooses to host the iframe and parent document on the same domain.
1. The iframe gets adjusted to host a 3rd party hosted script.
1. The 3rd party hosted script gets adjusted to try to escape sandboxes.
1. Sandbox is easily escaped when 'allow-scripts allow-same-origin' is used on an iframe that has
the same origin as the parent, such as downloads.

### The attack
1. Adjust frame owner element's attributes directly.
1. Reload iframe.
1. Have more privileges.


# Alternatives Explored  <a name="alternatives"></a>

Allow [document.domain = null](https://github.com/whatwg/html/issues/2757). Subtle
and is not delegated by parent frame but the embedee needs to assign it. So it is
subject to timing of the assignment of the `document.domain = null` value. This
assignment also had non-definitive rules around whether `same-origin` or
`same origin-domain` rules passed. Using the feature policy and adjusting
the `isPlatformObjectSameOrigin` algorithm clarifies this.

Implement another sandbox policy like same-origin but call it
same-origin-without-document-access. This itself is not useful for pages
that don't want to use sandboxes. And since sandboxes get all bits
and then opt out of specific things going forward it is difficult to
force web developers to using sandboxes just for this feature.

Implement `document-access` via a feature policy instead of an iframe
attribute. The problem with feature policy is that it inherits across
the frames that set it. So while a embeeder may wish to restrict a frame
access to its ancestors and siblings it may not wish to restrict access
inside that frame itself.
For example:

```
Frame A0
 Frame A1
  Frame B1
  Frame B2
 Frame A2
```

If Frame A1 has a feature policy set on it, it could not directly access
A2 or A0 but since feature policy inherits B1 could not access B2 either.
We wish to allow B1 & B2 to maintain accessing each other.
