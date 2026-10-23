# Independent WebContents lifetime for DOM embedding

- Start Date: 2026-10-23
- RFC PR: [electron/rfcs#34](https://github.com/electron/rfcs/pull/34)
- Electron Issues: [electron/electron#55248](https://github.com/electron/electron/issues/55248)
- Reference Implementation: [electron/electron#55135](https://github.com/electron/electron/pull/55135) (closed experimental prototype)
- Status: **Proposed**

## Summary

Separate the lifetime of a DOM-embedded WebContents from the lifetime of the DOM element used to display it. The existing webview implementation embeds an inner WebContents into an iframe; this proposal asks for a supported detach-and-reattach mechanism that leaves the WebContents alive when its display element is removed or its host closes. Applications could move the same live page between DOM slots and windows while continuing to control it through the main-process WebContents API. This RFC proposes the ownership and attachment contract, without prescribing a new tag name or requiring a particular public API shape.

## Motivation

### The existing webview lifecycle is tied to a DOM object

The existing `<webview>` combines two responsibilities: owning a guest page and presenting that page in the host document. Its implementation creates and manages a guest WebContents as part of the element's lifecycle. Element removal/disconnection participates in guest teardown; the public API does not give applications a supported way to remove the display element, retain that same live guest, and attach it to a new element in another window.

Applications often need the page to outlive its current display location. For example, moving a tab into a separate window should preserve its document, JavaScript heap, form values, scroll position and ongoing work. Closing the original workspace should not destroy a page already displayed in the separate window. Closing the separate window may restore the page to its original workspace, or destroy it if no workspace remains. The application should decide when the page is no longer needed.

### Native views have composition and interaction limitations

WebContentsView is Electron's recommended native embedding API and can already be moved between windows. It works well when an application's layout and input requirements fit native views.

The difficulty arises when composing multiple WebContentsViews together with an HTML interface. Each embedded page occupies a native view in a separate hierarchy from the host DOM. Applications must coordinate native view bounds and stacking with CSS layout. Overlapping views can cover another page or a host control in ways that cannot be resolved with the host document's CSS z-index alone.

Input makes this harder. A transparent-looking region does not necessarily allow mouse events to pass through to a view underneath. Applications may need an annotation layer to receive input on its controls while clicks elsewhere reach the embedded page. Multiple native WebContentsViews do not behave as one DOM hit-testing tree, and visual stacking and input routing must be coordinated separately. These are integration constraints, not a claim that every native composition is broken or that a DOM embedding path automatically solves every case.

### Reuse the iframe embedding concept with independent ownership

The existing webview implementation already displays an inner WebContents through an iframe in the host document. If that embedding relationship could be detached without destroying the inner WebContents, the application could keep its page model while replacing the display slot. Reattachment to a new iframe would permit window transfer without reloading the page.

The expected outcome is a narrow separation between the page and its display location. DOM layout can continue to describe where the page belongs, while the main process owns navigation, page events and destruction. The proposal does not promise arbitrary CSS transforms, overlay transparency or event pass-through: those behaviors still require explicit validation against Chromium's embedding and compositor rules.

## Guide-level explanation

### A page and a display slot are separate objects

An application creates an independently owned WebContents in the main process, using an API to be agreed during this RFC. A display slot in a trusted host document refers to that contents through an application-controlled binding. Creating the slot does not create the page; removing the slot does not destroy it.

Conceptually, application code would have these responsibilities. The following is pseudocode, not a proposed shipping API:

```js
const page = createIndependentlyOwnedContents(pagePreferences);
await page.loadURL('https://example.com');

bindPageToDOMSlot(page, workspaceSlot);

// Move the same page to another window.
detachPageFromDOMSlot(page);
bindPageToDOMSlot(page, separateWindowSlot);

// Release the page when the application's model no longer needs it.
page.destroy();
```

The existing main-process WebContents methods and events remain the way to navigate and control the page. The display slot need not proxy WebContents methods or forward its page events. Its minimal lifecycle notifications describe attachment and detachment only.

### Transfer and closure policy

A tab model holds a strong application reference to its WebContents and tracks its current display host. To open a tab in a separate window, the application detaches it, creates a fresh slot in the new window, and attaches the same contents there. The original slot may show a placeholder offering restoration.

Closing a window destroys only the webview element; it does not affect the lifetime of the WebContents object.

## Reference-level explanation

### Ownership and attachment are distinct

The main-process Electron wrapper owns the Chromium WebContents. A separate attachment component maintains the relationship between that inner WebContents and the host's target iframe. Attaching changes the display relationship, not ownership. Detaching clears that relationship while preserving the page object and document.

A page has at most one DOM attachment at a time. A host slot has at most one inner contents. Attachment to a new slot requires prior detachment; a request must not silently replace a page already occupying the destination iframe.

### Chromium implementation direction

Electron's current webview path uses guest management and Chromium's inner/outer WebContents iframe embedding. Chromium also exposes experimental `AttachUnownedInnerWebContents` and `DetachUnownedInnerWebContents` operations. These are a candidate mechanism for separating attachment from ownership, not a guarantee of long-term Chromium support.

The reference prototype uses an Electron bridge authorized through Chromium's PassKey restriction, and adds an observer notification after unowned detachment completes. It validates the target through private Chromium frame and frame-tree state. Any implementation would need to document these dependencies, minimize patches where possible, and continuously verify the lifecycle contract during Chromium upgrades.

The RFC does not require retaining the current guest-view registration machinery. Public API naming, renderer integration and the supported Chromium primitive should be chosen through review. The important requirement is that removing an embedding relationship cannot destroy the application-owned page.

### Fresh iframe targets

The candidate design creates a fresh initial-empty about:blank iframe for each attachment and obtains its current LocalFrameToken. The browser resolves that token within the requesting host frame's process and validates a live active direct child, local/same-SiteInstance state, no pending navigation, and no existing inner contents. Host and page must use the same session/BrowserContext. Self-embedding and cycles are rejected.

This deliberately avoids repurposing an arbitrary navigated iframe, invoking its beforeunload handlers, or replacing an occupied frame. The prototype uses this narrow precondition instead of a general prepare/attach protocol. That choice must be reviewed against Chromium's attachment requirements; it is not a statement that `PrepareForInnerWebContentsAttach` is unnecessary for general embedding.

### Completion and notifications

The native relationship must be fully cleared before detach reports completion. The next main-process attachment request must observe the detached state. This does not require a synchronous renderer IPC API: the public protocol can be asynchronous as long as its completion has an unambiguous meaning.

Each attachment has a unique identifier or equivalent generation. Delayed events and release requests from a removed element must not detach a later attachment of the same page. An attachment notification means the browser relationship has been established; it is separate from the first visible compositor frame.

### Required lifecycle behavior

| Operation or event | WebContents | DOM display slot |
| --- | --- | --- |
| Remove the element or destroy its iframe | Detaches and remains alive | Removed or empty |
| Navigate or destroy the host | Attachment is cleared; page object remains alive | Old slot becomes unavailable |
| Explicit detach | Remains alive and can be attached elsewhere | Becomes empty |
| Explicit WebContents destruction | Clears attachment and destroys the page | Element can remain empty |
| Attach after detach in another window | Same contents and document | New iframe displays it |
| Late release from an old element | Does not affect a newer attachment | Old bookkeeping is released |
| Invalid or occupied target | Request fails without modifying an existing attachment | No implicit replacement |

Independent ownership is not process fault isolation. If Chromium places the host and inner page in one renderer process, a crash may affect both. Cleanup must nevertheless leave no dangling native relationship or stale attachment record.

### Security and resource management

The browser process must validate the IPC sender, host frame, target iframe and page binding. A numeric WebContents ID is an identifier, not authorization. A supported design needs a clear host/page access policy, such as bindings established by the main process or grants scoped to a host and page. It must not enable a compromised host to claim arbitrary pages, cross sessions, or bypass origin isolation.

A closed shadow root is an implementation detail, not a security boundary. This capability must not change the embedded page's Node integration, sandbox or context-isolation policy. Request validation and resource limits must cover malformed, stale and flooded attachment operations.

Explicit ownership creates a cleanup obligation. Application shutdown, page destruction and failed attachment paths must release native state. Host closure releases attachment state, while the application decides whether the surviving detached contents are still needed.

### Interaction and composition validation

DOM placement is the motivation, but supported behavior must be specified rather than inferred from using an iframe. Validation should cover multiple embedded pages, overlap with host DOM controls, clipping, scroll and resize, zoom, pointer hit-testing, keyboard focus, IME, drag/drop and accessibility. Transparent pixels and pointer pass-through must be evaluated separately from visual composition.

The proposal also needs documented behavior for visibility/background throttling, beforeunload, fullscreen, popups, DevTools and cross-origin process swaps. Simultaneous DOM and WebContentsView display is outside the initial scope; future native/DOM transfer would require an explicit competing-owner policy.

### Prototype evidence

The experimental implementation linked above demonstrated independently owned iframe embedding on macOS. Electron 43.8.1 Release arm64 and x64 builds each passed 13 lifecycle tests, including unattached loading, element-first and contents-first destruction, transfer, immediate detachment, raw iframe removal and invalid, occupied or forged targets. The arm64 build also passed nine selected original-webview regressions and nine application transfer/cleanup checks.

Existing WebContents loadURL and executeJavaScript worked without an iframe attachment. The debugger API also worked: CDP Runtime evaluation, mouse input and page navigation were tested, and debugger connectivity survived detach and reattachment. No debugger API on the display element is required.

These results establish feasibility for the tested lifecycle paths. They do not establish Windows/Linux support, complete hostile-renderer security, or the multi-layer composition and input semantics described in the motivation. Those require dedicated validation before stabilization.

## Drawbacks

- Independent ownership moves cleanup responsibility to applications and can cause leaks if abandoned pages are not destroyed.
- An additional lifetime mode increases implementation, documentation and migration complexity.
- Experimental/private Chromium embedding dependencies require maintenance across Chromium upgrades.
- DOM embedding still has compositor and input limitations; it cannot promise every overlay or pass-through behavior applications want.
- Browser-side authorization and race handling add complexity even if the display element is small.
- Cross-platform behavior and interaction coverage remain incomplete in the prototype.

## Rationale and alternatives

**Add independent ownership to an existing DOM embedding API.** This most directly addresses the lifecycle coupling while reusing the concept of iframe embedding. It avoids requiring a second tag name, but needs an explicit opt-in and compatible behavior for existing applications.

**Improve WebContentsView composition and input APIs.** Better native clipping, stacking, geometry synchronization and hit-testing could address some motivating cases without a DOM attachment mechanism. This deserves comparison, especially where native views remain the most supportable architecture. It does not by itself make native views part of the host DOM layout and input tree.

**Offscreen rendering and shared textures.** Page output can be integrated into web rendering systems, but interactive embedding also requires input forwarding, focus, IME, accessibility and resource synchronization. This is a viable architectural alternative whose costs and behavior should be measured.

**Reload or serialize page state on transfer.** Simpler for applications that can tolerate it, but cannot generally preserve arbitrary third-party JavaScript heaps, ongoing work or debugging sessions.

**A JavaScript package or native addon.** A package can implement tab models and host-close delegates. Existing public APIs do not provide the required DOM detach-and-reattach primitive. Private Chromium access in an addon would relocate the compatibility and security burden rather than remove it.

Without this capability, applications can continue using WebContentsView, accept reloads, or build offscreen integration. The proposal is useful when retaining a live page and integrating its display with DOM layout are both required.
