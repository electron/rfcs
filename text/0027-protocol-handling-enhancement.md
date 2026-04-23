# Protocol Handling Enhancement

- Start Date: 2026-04-09
- RFC PR: [electron/rfcs#29](https://github.com/electron/rfcs/pull/29)
- Electron Issues: [electron/electron#15434](https://github.com/electron/electron/issues/15434), [electron/electron#49358](https://github.com/electron/electron/issues/49358), [electron/electron#39709](https://github.com/electron/electron/issues/39709), [electron/electron#39525](https://github.com/electron/electron/issues/39525)
- Reference Implementation: [electron/electron#49671](https://github.com/electron/electron/pull/49671)
- Status: **Proposed**

## Summary

This RFC outlines the need for a better approach to request interception / introspection / modification API's.

## Motivation

Right now, the ability to intercept, introspect, and/or modify HTTP requests is split across the `protocol.handle` API and the `webRequest` handlers. This has lead to some users attempting to use `protocol.handle` as a sort of middleware, using `bypassCustomProtocolHandlers` to forward the intercepted request to the built-in handler. This is the way that option is currently documented. However, it is documented it with the use of `net.fetch`, which can have different semantics than the Chromium browser (especially around redirects). A more-aligned "fallback to Chromium" path should be added.

The simplfied `Request`->`Response` flow has also lead to some users attempting to set cookies merely by returning a `Response` with their desired `Set-Cookie` header. Things like this may, currently, be better handled with `webRequest` handlers. However, that splits up behavior when a user may want to be able to do all of these things in a single API.

## Guide-level explanation

The future API design for depends on answers to the unresolved questions below.

## Reference-level explanation

The future API design for depends on answers to the unresolved questions below.

## Drawbacks

The work required to balance all the nuances here is likely non-trivial.

## Rationale and alternatives

N/A

## Prior art

To be filled out as alignment happens on a more clear API design.

## Unresolved questions

- Is `protocol.handle` meant, by us, to be an middleware-like interception API or a provider API for the `https` scheme?
- Should `protocol.handle` be split into two different API's (e.g. `protocol.handle` and `protocol.intercept`)?
- In the case that a more middleware-like interception API is desired:
  - Should we allow for the intercepted request to be modified before a potential fallback?
  - Should we allow for the returned response to be modified before passing it along?
  - Should the "fallback to Chromium" option use the in-flight request or dispatch a new one?
  - Should setting the `Set-Cookie` header in a returned response set the cookies?
  - Should layered handlers (middleware-like) be supported?
- What should happen to the `webRequest` handlers API?

## Future possibilities

N/A