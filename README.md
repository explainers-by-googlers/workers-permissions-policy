# Explainer for Permissions Policy for workers

This proposal is an early design sketch by the Chrome Permissions team to
describe the problem below and solicit feedback on the proposed solution. It has
not been approved to ship in Chrome.

## Proponents

- Chrome Permissions Team

## Participate

- https://github.com/explainers-by-googlers/workers-permissions-policy/issues

## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Example use case](#example-use-case)
  - [SharedArrayBuffer and cross-origin isolation](#sharedarraybuffer-and-cross-origin-isolation)
  - [Local network access](#local-network-access)
  - [Connected devices APIs](#connected-devices-apis)
- [Proposal](#proposal)
  - [Dedicated and shared workers](#dedicated-and-shared-workers)
  - [Service workers](#service-workers)
- [Considered alternatives](#considered-alternatives)
- [Security and Privacy Considerations](#security-and-privacy-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Permissions Policy allows embedders to control delegation of powerful web
capabilities to nested navigables. By default, Permissions Policy disables most
capabilities in cross-origin nested navigables, and it provides explicit ways to
enable them in specific contexts.

Conceptually, there is no real reason why a mechanism like Permissions Policy
shouldn’t exist for workers. However, on the one hand there has been no strong
need to enable powerful capabilities in workers, and on the other hand finding a
meaningful design that supports shared and service workers requires a slightly
different model than the existing one, which is based on the single inheritance
assumption, see also previous discussion on an
[issue](https://github.com/w3c/webappsec-permissions-policy/issues/207) and on a
[PR](https://github.com/w3c/webappsec-permissions-policy/pull/174) on the
Permissions Policy specification.

## Goals

The goal of this proposal is to extend Permissions Policy to workers, in order
to enable shipping Permissions Policy gated capabilities to those contexts.

## Non-goals

This document does not propose to expose any powerful capability to worker
contexts directly. That should be evaluated on an API by API basis. We just
propose to prepare the ground for it by supporting Permissions Policy in
workers.

## Example use case

### SharedArrayBuffer and cross-origin isolation

SharedArrayBuffers and other APIs that require cross-origin isolation can be
controlled, in window contexts, by the cross-origin-isolated policy-controlled
feature. SharedArrayBuffers are exposed to workers, and currently the [html
standard](https://html.spec.whatwg.org/#worker-processing-model) manually checks
the creator’s Permissions Policy for dedicated workers, but not for shared
workers, because of the multiple inheritance issue that this proposal aims to
solve. This proposal would allow to simplify that by reusing the Permissions
Policy framework directly also for workers.

### Local network access

The [Local Network Access](https://wicg.github.io/local-network-access/)
specification [gates](https://wicg.github.io/local-network-access/#fetching)
local network requests behind Permissions Policy. Given the absence of
Permissions Policy checks for workers, it currently forbids local network access
from workers.

This proposal would more easily allow to extend that logic to workers.

### Connected devices APIs

Most APIs which allow connecting to external peripherals are currently only
exposed to documents. However, there have been requests and attempts to extend
them to workers, in particular:

- [Gamepad API](https://github.com/w3c/gamepad/issues/37)
- [WebUSB](https://github.com/WICG/webusb/issues/73)
- [WebHID](https://github.com/WICG/webhid/issues/120)
- [Web MIDI](https://github.com/WebAudio/web-midi-api/issues/99)
- [Web Serial](https://github.com/WICG/serial/issues/116)

As those APIs are normally gated behind Permissions Policy, extending
Permissions Policy to workers seems a prerequisite for shipping those APIs to
workers in a consistent way.

## Proposal

While extending Permissions Policy to dedicated workers in the same way as it
works for iframes could be straightforward, supporting shared and service
workers requires more care since shared and service workers can be connected to
several contexts with potentially different effective policies.

We actually propose different models for dedicated and shared workers on the one
hand, and for service workers on the other hand. For dedicated and shared
workers we propose a model that takes into account delegation of capabilities
from their creator contexts, similar to child documents. On the other hand,
given their unique model, we propose to handle service workers as top-level
contexts, that control their policy controlled features uniquely through a
response header.

### Dedicated and shared workers

The mechanism we propose for dedicated and shared workers entails two parts,
similar to how Permissions Policy works for iframes. On the one hand, worker
scripts can specify in a `Permissions-Policy` header the policies which they
need in order to run (the *declared policies*). On the other hand, the worker’s
client can enforce through delegation some policies, which we call the
*delegated policies*, on the worker (depending on which policies are delegated
by its ancestors and by using a new `allow` parameter when creating the worker).

When starting a worker (or connecting to a shared worker), the user agent will
compare the two sets (or, to be more specific, dictionaries) of *declared
policies* and *delegated policies*. Unless the *delegated policies* are laxer
than the *declared policies*, starting the worker will result in an error.

For example, suppose that two workers scripts are served with the following
Permissions-Policy header:

```
// worker.js, shared-worker.js are served with
Permissions-Policy: some-capability=(self)
```

In order to be started successfully, the workers need to be delegated the
policy-controlled feature `some-capability`. In particular, starting those
workers from the top-level document with default Permissions-Policy would
succeed:

```js
// index.html (main document), served with no Permissions-Policy header.
// These would all succeed:
const worker = new Worker("./worker.js");
const sharedWorker = new SharedWorker("./shared-worker.js");
```

However, if the Permissions-Policy disables the needed feature, they would fail.
In particular,

```js
// index.html (main document), served with
// Permissions-Policy: some-capability=().

// The worker will not start. It will trigger worker.onerror.
const worker = new Worker("./worker.js");

// The shared worker will not start. It will trigger sharedWorker.onerror.
const sharedWorker = new SharedWorker("./shared-worker.js");
```

For workers started from cross-origin nested contexts, the Permissions Policy
inherited by the nested context matters. If the context is not allowed to use
the feature, then it cannot start a worker which needs that feature:

```js
// index.html (main document), served with no Permissions-Policy header
// Embeds:
// <iframe src="https://cross-origin.test/iframe.html">

// https://cross-origin.test/iframe.html
// The worker will not start. It will trigger worker.onerror.
const worker = new Worker("./worker.js");
// The shared worker will not start. It will trigger sharedWorker.onerror.
const sharedWorker = new SharedWorker("./shared-worker.js");
```

For that to work, the nested context needs to be allowed to use the delegated
capability:

```js
// index.html (main document), served with no Permissions-Policy header
// Embeds:
// <iframe src="https://cross-origin.test/iframe.html" allow="some-capability">

// https://cross-origin.test/iframe.html
// These would all succeed:
const worker = new Worker("./worker.js");
const sharedWorker = new SharedWorker("./shared-worker.js");
```

For enabling more granular control when delegating capabilities to workers, we
also propose to introduce an `allow` parameter to the worker constructor (to be
passed when creating/connecting to workers) which works analogously as the
`allow` attribute on iframes:

```js
// index.html (main document), served with no Permissions-Policy header

// The worker will not start. It will trigger worker.onerror.
const worker = new Worker("./worker.js", {
  allow: "some-capability 'none'"
});
// The shared worker will not start. It will trigger sharedWorker.onerror.
const sharedWorker = new SharedWorker("./shared-worker.js", {
  allow: "some-capability 'none'"
});
```

For backwards compatibility, the *declared policies* will be empty if no
Permissions-Policy header is specified (hence no Permissions Policy error will
be triggered when the Permissions-Policy header is absent, and the worker will
just not be allowed to use any policy-controlled feature by default).

We propose to keep the usual syntax for the Permissions-Policy header for
workers. In particular, this allows to allowlist more origins than just `self`.
This is allowed but useless, since workers can start other workers, but only of
the same-origin.

Reporting will work for workers in the same way as it works for documents.
Potential violations will be reported when starting a worker if there is a
permissions policy mismatch. Moreover, workers can add `report-to` to their
`Permissions-Policy` header or deliver a `Permissions-Policy-Report-Only` header
in order to get Permissions Policy violation reports sent to the relative
reporting endpoint.

### Service workers

For service workers, we propose to acknowledge their unique model (which for
example allows them to run in background, with no clients attached, when
processing a push message, or lets the user agent start them before a matching
document in order to intercept the corresponding network request) and treat them
as top-level contexts. However, given the fact that they effectively run in
background, we propose to enable policy-controlled features in service workers
only case-by-case.

In particular, we propose to augment every [policy-controlled
feature](https://w3c.github.io/webappsec-permissions-policy/#features) with a
boolean *can be enabled in service workers*. Unless stated otherwise, and in
particular for all currently defined policy-controlled features, that boolean is
*false*.

If a policy-controlled feature *cannot be enabled in service workers* (that is,
the corresponding boolean is *false*), then no service worker can use that
feature.

We still propose to extend the `Permissions-Policy` header to service workers.
Through that header, service workers can enable policy-controlled features which
*can be enabled in service workers*.

For example, given two policy-controlled features `feature-a`, which *can be
enabled in service workers*, and `feature-b`, which *cannot be enabled in
service workers*, we would have the following:

- A service worker served with no `Permissions-Policy` header would not be
  allowed to use `feature-a` and not be allowed to use `feature-b`.

- A service worker served with `Permissions-Policy: feature-a=(self)
  feature-b=(self)` would be allowed to use `feature-a` but would still not be
  allowed to use `feature-b`.

More background on what motivated this proposal for service workers is contained
in [issue 4](https://w3c.github.io/webappsec-permissions-policy/#features).

## Considered alternatives

A few alternatives were considered for this design:

* Instead of throwing an error, we could just disable the non-delegated
  policy-controlled feature in the worker (following the same model as for
  iframes). However, this does not work well with multiple inheritance (shared
  workers). For example, a shared worker with multiple contexts connecting to it
  could have inherited different policies depending on which context started it
  first, resulting in non-predictable behavior and possibly privilege escalation
  (if the shared worker has broader access than any of its connecting contexts).

* We could default the delivered Permissions-Policy of workers as for documents,
  instead of disabling everything by default. However, this would break
  backwards compatibility.

* We could have a different, simpler syntax for the Permissions-Policy header in
  workers (and for the `allow` parameter), that just allows enabling/disabling
  features without control on the list of origins, since there is no need at the
  moment to support other origins anyway. However, it seems better to keep the
  same syntax as for documents. Moreover, this is future-proof in case the
  platform ever allowed cross-origin workers in the future.

* The complication of triggering an error if there is a policy mismatch is
  really only needed for shared workers, to ensure that a shared worker is
  started with a consistent and predictable set of effective policies
  independently of which context connects to it first. Dedicated workers could
  follow a simpler model, analogous to iframes, in which the effective policies
  are just the intersection of the declared policies and the delegated policies.
  We propose here to accept the additional complication also for dedicated
  workers in exchange for coherence in the behavior of shared and dedicated
  workers, although we acknowledge that the alternative is also a viable option.

## Security and Privacy Considerations

This proposal in itself does not seem to have any security or privacy impact,
since it does not expose anything new by itself. Exposing to workers single
powerful capabilities which are gated behind Permissions Policy should be
evaluated separately.
