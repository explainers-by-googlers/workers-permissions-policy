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
- [Example use cases](#example-use-cases)
  - [Connected devices APIs](#connected-devices-apis)
  - [SharedArrayBuffer and cross-origin isolation](#sharedarraybuffer-and-cross-origin-isolation)
  - [Local network access](#local-network-access)
- [Proposal](#proposal)
  - [Permissions-Policy headers](#permissions-policy-headers)
  - [Dedicated workers](#dedicated-workers)
  - [Shared workers](#shared-workers)
  - [Service workers](#service-workers)
- [Considered alternatives](#considered-alternatives)
  - [Policies controlled by the `Permissions-Policy` header in shared workers](#policies-controlled-by-the-permissions-policy-header-in-shared-workers)
  - [Disable non-delegated policies instead of throwing for shared workers](#disable-non-delegated-policies-instead-of-throwing-for-shared-workers)
  - [Key shared workers by their effective permission policy](#key-shared-workers-by-their-effective-permission-policy)
  - [Other alternatives](#other-alternatives)
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

## Example use cases

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

## Proposal

While extending Permissions Policy to dedicated workers in the same way as it
works for iframes is straightforward, supporting shared and service workers
requires more care since shared and service workers can be connected to several
contexts with potentially different effective policies. For service workers,
their unique model with their complex lifecycle (including the fact that they
are implicitly started whenever a matching context is loaded and are
automatically upgraded) makes delegation of capabilities even harder.

Hence we propose different models for the three types of workers:

- Dedicated workers will behave like iframes, inheriting their policies from
  their creator context.

- For shared workers will have their policies set at creation time, and will
  allow connections only if the connecting context has laxer policies than the
  ones allowed in the shared worker.

- Service workers, given their unique model, will be treated as top-level
  contexts.

### Permissions-Policy headers

We propose to support the `Permissions-Policy` and
`Permissions-Policy-Report-Only` header in dedicated, shared and service
workers, with the same syntax and meaning as for documents. In particular, the
headers allow to disable some policy-controlled features in the worker context
and to control reporting.

Since workers cannot start or embed nested cross-origin contexts, in practice
specifying allowlists of origins in the `Permissions-Policy` header of workers
will be useless. The only exception might be the case of a dedicated worker
starting another dedicated worker with an opaque origin.

### Dedicated workers

Even if this is not covered yet in the specification, some vendors already
implement Permissions Policy inheritance for dedicated workers, a behavior that
is also covered by WPTs, see [issue
5](https://github.com/explainers-by-googlers/workers-permissions-policy/issues/5).

We propose to specify this model for dedicated workers, adding support for the
`Permissions-Policy` and `Permissions-Policy-Report-Only` headers to further
restrict the allowed policy-controlled features and for customizing reporting.

### Shared workers

We propose a model that satisfied the following key properties:

- If a document is not allowed to use a certain policy-controlled feature, then
  any worker it can connect to is allowed to use that feature, either.

- Only one shared worker can run at any given time for a given URL.

- The model is backwards compatible.

The concrete mechanism that we propose is similar to how Permissions Policy
works for cross-origin iframes (even if shared workers are same-origin!). In
particular, when starting a shared worker, an `allow` parameter can be used to
delegate capabilities to the shared worker (with no capabilities delegated by
default, as for cross-origin iframes).

```js
const sharedWorker = new SharedWorker("./shared-worker.js",
                                      {allow: "some-capability"});
```


If another context later tries to connect to the same shared worker, it must
specify the same `allow` parameter, otherwise the connection will fail. This is
similar to how `extendedLifetime` works.

```js
const sharedWorker = new SharedWorker("./shared-worker.js",
                                      {allow: "some-capability"});

// this will result in sharedWorker2.onerror:
const sharedWorker2 = new SharedWorker("./shared-worker.js");
```

Differently than for iframes, however, allowlisting in the `allow` header a
policy-controlled feature which is not enabled in the worker's parent context
will result in an error:

```js
// index.html (main document), served with no Permissions-Policy header
// Embeds:
// <iframe src="https://cross-origin.test/iframe.html">
// "some-capability" is disabled in the cross-origin iframe.

// https://cross-origin.test/iframe.html
// The shared worker will not start. It will trigger sharedWorker.onerror.
const sharedWorker = new SharedWorker("./shared-worker.js",
                                      {allow: "some-capability"});
```


This seems necessary in order to avoid shared workers
to be created by mistake with less policies than they actually need. (As an
example, suppose that `http://a.test` contains code to start a shared worker,
enabling a policy-controlled feature which is essential for the site to work
properly. If, by mistake, `https://a.test` should end up embedded as a
cross-origin iframe, that could start a shared worker without the needed
policy-controlled feature, which in turn could break the functionality of a
top-level `http://a.test` page opened in another tab.)

The `Permissions-Policy` and `Permissions-Policy-Report-Only` headers can be
used to additionally restrict the set of enabled policy-controlled features and
to control reporting settings.

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

### Policies controlled by the `Permissions-Policy` header in shared workers

An earlier version of this document was proposing, for shared workers, to
enforce, as effective policies, the policies delivered in the
`Permissions-Policy` header, and to refuse connections if the creator/connecting
context could not delegate enough capabilities to satisfy the ones required in
the `Permissions-Policy` header.

However, it was [pointed
out](https://github.com/explainers-by-googlers/workers-permissions-policy/issues/3)
that using the `allow` parameter in `SharedWorkerOptions` better adheres to the
existing behavior for other properties, like `extendedLifetime`. Moreover, the
model proposed with the `allow` parameter seems easier to implement and to
follow, since it replicates the current model for cross-origin iframes.

### Disable non-delegated policies instead of throwing for shared workers

Instead of triggering an error, we could just disable the non-delegated
policy-controlled features in the shared worker (following the same model as for
iframes). However, this does not work well with multiple inheritance. For
example, a shared worker with multiple contexts connecting to it could have
inherited different policies depending on which context started it first,
resulting in non-predictable behavior and possibly privilege escalation (if the
shared worker has broader access than any of its connecting contexts).

### Key shared workers by their effective permission policy

We could key shared workers by their delegated permissions policy (i.e. the
policies listed in the `allow` parameter when starting the worker, assuming that
they are all allowed in the creator context itself) and allow multiple shared
workers with the same URL and different policies. That would break the promise
that there is only one shared worker for a given URL though.

### Other alternatives

* We could default the `allow` parameter to delegate everything, similar to what
  happens for same-origin child iframes.

* We could have a different, simpler syntax for the Permissions-Policy header in
  workers (and for the `allow` parameter), that just allows enabling/disabling
  features without control on the list of origins, since there is no need at the
  moment to support other origins anyway. However, it seems better to keep the
  same syntax as for documents. Moreover, this is future-proof in case the
  platform ever allowed cross-origin workers in the future.

## Security and Privacy Considerations

This proposal in itself does not seem to have any security or privacy impact,
since it does not expose anything new by itself. Exposing to workers single
powerful capabilities which are gated behind Permissions Policy should be
evaluated separately.
