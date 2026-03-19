# Explainer for Service Worker Synthetic Response

This proposal is an early design sketch by Google Chrome Loading team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Authors

- [Shunya Shishido](https://github.com/sisidovski)
- [Yoshisato Yanagisawa](https://github.com/yoshisatoyanagisawa)

## Proponents

- Google Chrome Loading team

## Participate
- [Discussion forum](https://github.com/explainers-by-googlers/service-worker-synthetic-response/issues)

## Table of Contents 

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
- [Synthetic Response for navigations via the Static Routing API](#synthetic-response-for-navigations-via-the-static-routing-api)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Example: Cache for App Shell](#example-cache-for-app-shell)
    - [Example: Cache for HTTP Headers only](#example-cache-for-http-headers-only)
- [Detailed design discussion](#detailed-design-discussion)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

The Service Worker Static Routing API (https://github.com/WICG/service-worker-static-routing-api) provides a mechanism to bypass Service Worker bootstrap by declaratively routing requests. While this solves the "Service Worker Startup" tax, navigation performance is still limited by network latency. 

Currently, even with a static route, the browser process must wait for the first bytes of a network response before it can create/find a renderer process and commit the navigation. This "Network-to-Commit" gap can often take hundreds of milliseconds.

Service Worker Synthetic Response extends the Static Routing API to eliminate this gap. It allows the browser to immediately serve a cached "synthetic" response (like an App Shell) to trigger the renderer commit phase instantly, while the dynamic content is fetched from the network in the background.


## Goals

- Early Navigation Commit: Trigger the renderer's commit phase immediately using cached data, without waiting for a network round-trip.
- Parallelize Renderer Initialization and Network Fetch: Overlap the time spent setting up the renderer process with the time spent waiting for the server.
- Improved Loading Metrics: By committing the static part of the response immediately, the browser can start initializing the renderer process, which includes the document creation or JavaScript engine startup. This results in parsing resources (CSS/JS) and painting the UI sooner.


## Non-goals

- Handling Non-GET Requests: The synthetic response mechanism is intended for navigation requests (requestMode: "navigate"), which are primarily GET requests. Handling POST or other non-idempotent methods is not a goal.
- Full Body Merging Logic: The synthetic response proposes a solution to merge responses. But particularly how to merge two response bodies might be handled by other proposals e.g. [Declarative partial updates](https://github.com/WICG/declarative-partial-updates).
- Offline Solution: While it utilizes the cache, it is primarily a performance optimization for connected navigations. It is not intended to replace full offline-first PWA architectures.

## User research

There are many websites using Service Worker App Shell architecture. We can frame the synthetic response as a native support of App Shell.

## Use cases

The Problem: The Network-to-Commit Gap
Even with Static Routing API, standard navigation follows these steps.

- Browser initiates a network request.
- Idle Time: Browser waits for the server to respond (TTFB).
- Browser receives the response.
- Renderer Setup: Browser finds/creates a renderer process and starts the Commit phase.
- Renderer begins loading and parsing.

The renderer is idle during step 2. For users on slow or high-latency networks, this gap is the primary source of perceived slowness.

## Synthetic Response for navigations via the Static Routing API

```js
self.addEventListener("install", e => {
  e.addRoutes({
    condition: { requestMode: "navigate" },
    source: {
      syntheticResponse: {
        cacheName: "static-resources",
        contentPath: "/app-shell.html",
        fallbackPath: “/error-page.html”,
        timeout: 3000
      }
    }
  });
});
```

### How this solution would solve the use cases

Key Performance Benefits:
- Unblocking the Renderer: The renderer's life cycle (Find -> Commit -> Load) is moved much earlier in the timeline, often occurring before the server has even finished processing the request.
- Decoupling from TTFB: Navigation commitment becomes a local operation (cache hit) rather than a network-bound operation.
- Optimized Resource Discovery: By committing the shell early, the renderer can discover and start fetching static sub-resources (like site-wide CSS or fonts) defined in the <head> of the shell while waiting for the primary body content.

#### Example: Cache for App Shell

This allows the browser to commit a full static UI shell while waiting for the body content.

```js
self.addEventListener("install", e => {
  e.addRoutes({
    condition: { requestMode: "navigate" },
    source: {
      syntheticResponse: {
        cacheName: "static-resources",
        contentPath: "/app-shell.html", // Served immediately to trigger commit
        fallbackPath: “/error-page.html”,
        timeout: 3000
      }
    }
  });
});
```

#### Example: Cache for HTTP Headers only

The simplest optimization: commit the renderer with just the cached headers. This allows the renderer to begin pre-initializing (e.g., preparing for the specific Content-Type) while the entire body comes from the network.

```js
self.addEventListener("install", e => {
  e.addRoutes({
    condition: { requestMode: "navigate" },
    source: {
      syntheticResponse: {
        cacheName: "v1",
        httpHeaderPath: "/_header" // Minimal headers to unblock commit
      }
    }
  });
});
```

## Detailed design discussion

- Body Merging Strategy: Defining exactly where the network body starts and the cached shell ends. Declarative partial updates proposal might be a solution.
- CSP Nonce Consistency: Handling Content Security Policy nonces that are typically unique per-request but are now partially served from a static cache. Using <meta> tag is one workaround, but it may be better to provide a way to inform browser generated nonce strings for CSP to servers, and encourage servers to use them.
- Server Signaling: Providing a way for the server to know it only needs to send the "body" part (e.g., via a specialized request header).


<!--
### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

-->

## Security and Privacy Considerations

The synthetic response has the same level of capability that the current Service Worker and Service Worker Static Routing API can do. Please refer the Security and Privacy section of the [Service Worker Static Routing API proposal](https://github.com/WICG/service-worker-static-routing-api/blob/main/security-privacy-questionnaire.md).

<!--
[Describe any interesting answers you give to the [Security and Privacy Self-Review
Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/) and any interesting ways that
your feature interacts with [Chromium's Web Platform Security
Guidelines](https://chromium.googlesource.com/chromium/src/+/master/docs/security/web-platform-security-guidelines.md).]

-->

## Stakeholder Feedback / Opposition

This proposal was presented at Web Applications WG TPAC 2025. There were no clear objections. More detailed design and proposal were requested to evaluate that this is really the case that existing Service Worker APIs can't achieve. 
- [minutes](https://docs.google.com/document/d/1fOjXiGUFf_krkfYaWf5drbqzY7CTxn7EcDYr5ZWjzFg/edit?tab=t.0#heading=h.czu3oufq4anf).
- [slides](https://docs.google.com/presentation/d/1J-XVrSxSRpAp3nANZq7KoSMYYM1ZQOTvLKuv51SNJ3Q/edit?slide=id.p#slide=id.p)

<!--

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

-->

## References & acknowledgements

<!--
Many thanks for valuable feedback and advice from:

- [Person 1]
-->
