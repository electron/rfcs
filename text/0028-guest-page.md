# GuestPage: main-process-driven embedding of web content into a host page

- Start Date: 2026-05-26
- RFC PR: [electron/rfcs#0000](https://github.com/electron/rfcs/pull/0000)
- Electron Issues: N/A
- Reference Implementation: N/A
- Status: **Proposed**

## Summary

`GuestPage` is a new main-process API for embedding a separately-loaded web page into a
placeholder element of another `WebContents`. The embedded page is composited as part of the host
page, so its bounds, z-order, clipping, transforms, and scrolling follow the host element through
normal layout — like `<webview>` — but the guest is created, configured, navigated, and observed
**exclusively from the main process**. The host renderer gains no new capabilities: its only role
is to provide a placeholder `<iframe>` and to lay it out.

The API is deliberately specified so that it can be implemented on top of today's
inner-`WebContents` attachment mechanism (`WebContents::AttachInnerWebContents`, the same primitive
`<webview>` uses) *and* on Chromium's in-progress MPArch GuestView architecture
(`content::GuestPageHolder` + `WebContents::AttachGuestPage`), in which guests no longer have a
`WebContents` at all. For that reason `GuestPage` never exposes a `webContents` property and is not
related to `WebContentsView`.

`GuestPage` is intended to become the long-term successor to the `<webview>` tag.

## Motivation

### The placement problem

A recurring request from app developers is: *"I want a piece of embedded web content whose bounds
and stacking order are controlled by an element in my app's UI."* Today there are two ways to do
this, and both have significant problems:

1. **`<webview>`** gives exactly the right visual behavior — the guest is composited inside the
   host page, so bounds, z-index, clipping, and scrolling all follow the element — but:
   - it is renderer-driven: the embedder renderer chooses the guest's URL, partition, preload
     script, and (via the unfiltered `webpreferences` attribute string) effectively any
     `WebPreferences` key, constrained only by a small inheritance allowlist and the opt-in
     `will-attach-webview` event;
   - the embedder renderer retains a privileged control channel over the guest after creation
     (`executeJavaScript`, `loadURL`, `sendInputEvent`, `openDevTools`, `downloadURL`, …);
   - enabling it requires the all-or-nothing `webviewTag` web preference for the whole host
     `WebContents`;
   - the documentation has recommended against using it for years due to the instability of the
     underlying Chromium guest architecture, which is now actively being rebuilt (MPArch).

2. **`WebContentsView` + manual bounds syncing** keeps the security model right (everything is
   owned by the main process) but the visual integration wrong:
   - the view is a native child view that is always stacked above the window's web contents; it
     cannot be interleaved with the host page's own painted content, clipped by the host's
     `overflow`, or affected by host transforms;
   - keeping its bounds in sync with a DOM element requires `ResizeObserver`/scroll listeners,
     IPC, and `setBounds()` calls, which visibly lag during scrolling, animation, and window
     resizing.

`GuestPage` combines the two: the compositing model of `<webview>` with the ownership and security
model of a main-process-created `WebContents`.

### The `<webview>` succession problem

Chromium is migrating GuestViews to MPArch ("multiple page architecture",
[crbug.com/40202416](https://crbug.com/40202416)). Under MPArch a guest is no longer a separate
`content::WebContents` attached into the embedder; it is a guest `FrameTree` owned by a
`content::GuestPageHolder` and hosted *inside* the embedder's `WebContents`. Once that ships,
`WebContents::AttachInnerWebContents` — the primitive Electron's `<webview>` is built on — is
slated for removal.

This has two consequences:

- Electron's `<webview>` tag will need substantial rework regardless of this RFC, and some of that
  rework (per-guest preferences and preload delivery, guest session handling) is shared with this
  proposal.
- Any *new* embedding API must be specified so that it does not promise things only the old
  architecture can deliver. In particular, it must not promise that the embedded content *is* a
  `WebContents`.

`GuestPage` is specified against the intersection of both architectures so that the API contract
survives the migration.

### Expected outcome

- Apps can place main-process-controlled web content using ordinary HTML/CSS layout in their UI,
  with correct stacking, clipping, and scroll behavior.
- Apps that today enable `webviewTag` only because they need embedded content get a path to turn
  it off, removing a renderer-accessible privilege-escalation surface.
- Electron gets a credible long-term replacement story for `<webview>` that does not depend on
  deprecated Chromium machinery.

## Guide-level explanation

### Concepts

- **GuestPage** — a main-process object representing an embedded web page. Conceptually similar to
  a `webContents` you create yourself, except that it renders inside another page's element
  instead of inside a window, and its API surface is intentionally smaller.
- **Slot** — an ordinary `<iframe>` element in the host page (typically `about:blank`, typically
  wrapped in whatever app component makes sense). The slot has no special powers and requires no
  Electron API in the renderer; it is simply where the guest appears. The host page controls the
  slot's size, position, stacking, clipping, and visibility through normal CSS — that is the
  feature.
- **Attaching** — the main-process operation that connects a `GuestPage` to a slot, identified by
  the slot frame's `WebFrameMain`.

### Basic usage

Host page (renderer — no Electron APIs involved):

```html
<div class="sidebar">…</div>
<div class="content">
  <!-- The guest renders here. Style/position/stack this like any other element. -->
  <iframe name="docs-pane" class="docs-pane"></iframe>
</div>
```

Main process:

```js
const { GuestPage } = require('electron')

const docs = new GuestPage({
  // Defaults to the host's session once attached; set explicitly if you need otherwise.
  session: session.defaultSession,
})

await docs.loadURL('https://docs.example.com')

// Find the slot frame in the host WebContents and attach.
const slot = hostContents.mainFrame.frames.find((f) => f.name === 'docs-pane')
await docs.attach(slot)

// Control and observe the guest from the main process, as usual.
docs.on('did-navigate', (event, url) => updateAddressBar(url))
docs.mainFrame.send('set-theme', 'dark')
docs.setWindowOpenHandler(({ url }) => {
  shell.openExternal(url)
  return { action: 'deny' }
})

docs.on('detached', ({ reason }) => {
  // 'slot-removed' | 'host-navigated' | 'host-destroyed'
})
```

The host page never sees a new API. If the host page is compromised, the only things it can do to
the guest are things it can already do to its own DOM: move, resize, occlude, hide, or remove the
slot, focus it, or `postMessage` to it (subject to the usual origin checks). It cannot create
guests, choose what they load, navigate them, read their content, or inject input into them.

### Mental model versus `<webview>`

| | `<webview>` | `GuestPage` |
| --- | --- | --- |
| Who creates the guest | host renderer (tag insertion) | main process (`new GuestPage()`) |
| Who chooses URL, session, preload, prefs | host renderer attributes (+ `will-attach-webview` veto) | main process constructor/methods |
| Who controls the guest afterwards | host renderer (element methods) and main | main process only |
| Where does placement come from | element layout | element layout (the slot) |
| Renderer capability required | `webviewTag: true` for the whole WebContents | none |
| Guest object in main process | full `webContents` | `GuestPage` (reduced, migration-safe surface) |

### Popups

If the guest calls `window.open()`, the app decides what happens via `setWindowOpenHandler`:

```js
checkout.setWindowOpenHandler((details) => {
  if (details.url.startsWith('https://pay.example.com/3ds')) {
    // Create the popup as another guest page; the opener relationship is preserved.
    return { action: 'guest' }
  }
  return { action: 'deny' }
})

checkout.on('guest-created', async (event, popup, details) => {
  // Ask the host page (via your own IPC) to add another slot, then attach.
  const slotName = await requestPopupSlot(hostContents, details)
  const slot = hostContents.mainFrame.frames.find((f) => f.name === slotName)
  await popup.attach(slot)
})
```

With `{ action: 'guest' }`, `window.opener`, named targeting, and `postMessage` between the opener
guest and the popup guest work as they do for normal pages, **provided the popup is attached to a
slot in the same host `WebContents`**. This mirrors how Chromium's own `<webview>` "newwindow"
flow works, on both the current and the MPArch guest architectures. Opening the popup as an
independent `BrowserWindow` is possible (`{ action: 'deny' }` plus opening the URL yourself), but
in that case there is no `window.opener` relationship — that combination is not supported by
either architecture.

### Sessions and storage

- By default a `GuestPage` uses the **host's session** once attached. This is the guaranteed,
  forever-supported configuration.
- A different `session` may be passed at construction. This works on the initial implementation
  (see Reference-level explanation), and is the same capability `<webview partition="…">` has
  today. Its long-term shape under MPArch depends on an upstream question about guest
  `BrowserContext`s that this RFC calls out explicitly in *Unresolved questions*; apps should not
  assume that arbitrary cross-session guests will remain expressible in exactly this form.
- Either way, the cookie jar and storage the guest uses are decided by the main process, never by
  the host renderer (unlike `<webview>`'s renderer-chosen `partition` attribute).

### Migrating from `<webview>`

| `<webview>` feature | `GuestPage` equivalent |
| --- | --- |
| `src` attribute | `guestPage.loadURL()` |
| `partition` attribute | `session` constructor option (main-chosen) |
| `preload` attribute | `preload` constructor option (main-chosen; see provisional tier) |
| `nodeintegration`, `webpreferences`, `disablewebsecurity`, … | not available to the host page at all; set explicitly in the constructor where supported |
| element methods (`loadURL`, `executeJavaScript`, `openDevTools`, …) | main-process methods on `GuestPage` / `guestPage.mainFrame`; the host page has no methods |
| `ipc-message` event / `webview.send()` | ordinary IPC with the guest's frames (`guestPage.mainFrame.send()`, `ipcMain` with `event.senderFrame`) |
| `new-window` / `setWindowOpenHandler` on the guest contents | `guestPage.setWindowOpenHandler()` |
| `will-attach-webview` sanitization in main | unnecessary — the main process is the creator |

Apps embedding content they fully control can migrate mechanically. Apps that intentionally let
remote content in the host page spawn its own webviews cannot be expressed with `GuestPage` — that
is by design.

### Platform considerations

The guest is composited content, not a native view, so behavior is identical across Windows,
macOS, and Linux. No per-platform API differences are proposed.

## Reference-level explanation

### Object model

`GuestPage` is an `EventEmitter` created and used only in the main process. It deliberately does
**not** expose a `webContents`, is not a `View`, and has no relationship to `WebContentsView` or
window APIs. This is what makes the contract implementable on the MPArch guest architecture, where
the embedded page has no `WebContents` of its own.

The API surface is split into two tiers:

- **Guaranteed** — implementable on both the current architecture and MPArch using public
  `content/` APIs that exist today. These ship with the first version.
- **Provisional** — desirable, implementable today, but dependent on parts of the MPArch guest
  implementation that are currently incomplete upstream, or on Electron-side plumbing that is
  shared with the future `<webview>` migration. These ship behind clearly-documented experimental
  status (or in follow-ups) so that the guaranteed tier never has to regress.

#### Constructor

```ts
new GuestPage(options?: {
  session?: Session              // default: the host's session at attach time
  transparent?: boolean          // default false
  backgroundColor?: string
  // Provisional tier:
  preload?: string               // absolute path, main-chosen
  partition?: string             // alternative to `session`
})
```

#### Guaranteed methods, properties, and events

| Member | Notes |
| --- | --- |
| `loadURL(url, options)` / `reload()` / `stop()` | Navigation may be initiated before attach; rendering begins at attach. |
| `url`, `getTitle()` | |
| `navigationHistory` | Same `NavigationHistory` class used by `webContents`. |
| `mainFrame: WebFrameMain` | Per-frame surface: `executeJavaScript`, `send()`/`postMessage()`, `frames`, `origin`, process info. Null before first attach/navigation as appropriate. |
| `session: Session` | Read-only. |
| `attach(frame: WebFrameMain): Promise<void>` | See *Attachment semantics*. |
| `isAttached()`, `isDestroyed()`, `destroy()` | |
| `setAudioMuted()` / `isAudioMuted()` | |
| `setWindowOpenHandler(handler)` | Actions: `'deny'`, `'guest'` (see *Popups*). |
| Events: `did-start-loading`, `did-stop-loading`, `did-finish-load`, `did-fail-load`, `did-navigate`, `did-navigate-in-page`, `dom-ready`, `page-title-updated`, `render-process-gone`, `attached`, `detached`, `destroyed`, `guest-created` | |

#### Provisional members

`preload`, `setZoomFactor()`/`getZoomFactor()`, `page-favicon-updated`, `openDevTools()`,
`capturePage()`, `findInPage()`, `printToPDF()`, `sendInputEvent()`, fullscreen-related events.

Each of these has a workable implementation today (the guest is a real `WebContents` internally),
but their MPArch equivalents are either per-frame rather than per-page (zoom, devtools), rely on
delegate plumbing that upstream has not finished (fullscreen, pointer lock, downloads,
find-in-page for unattached guests), or require Electron-side per-guest preference delivery that
does not exist yet (preload, sandbox/contextIsolation overrides). Promising them as guaranteed now
risks an API break when the backend changes, so they are explicitly marked.

#### Deliberately excluded

- `guestPage.webContents` — never. This is the single most important exclusion; everything else
  follows from it.
- Any renderer-facing API: no tag, no creation IPC, no method-forwarding channel, no
  `webviewTag`-style preference. (An optional, capability-free custom element that wraps a slot
  `<iframe>` for ergonomics may be published as a userland package or documentation recipe; it is
  not part of this proposal.)
- Adoption of an *existing* `webContents` or `WebContentsView` into a slot
  (`webContents.attachToFrame(...)`). The current architecture could support it; MPArch cannot.
  Out of scope permanently unless upstream direction changes.

### Attachment semantics

`guestPage.attach(frame)`:

- `frame` must be a `WebFrameMain` that is a **subframe** (never a main frame) of some
  `WebContents`, must be current and live, and should be an `about:blank` document (matching the
  guidance in the Chromium attach APIs). The slot's parent frame must not be another guest.
- Validation happens entirely in the browser process against a main-process-supplied frame handle.
  There are no renderer-supplied identifiers in the attach path (unlike `<webview>`, whose attach
  IPC carries a renderer-supplied frame token whose ownership is only `DCHECK`ed).
- One guest per slot, one slot per guest. Attaching to an occupied slot, or attaching an
  already-attached guest, throws.
- There is **no `detach()`**. Neither architecture can preserve a guest across detachment from its
  outer frame: on the current architecture the inner `WebContents` is owned and destroyed by the
  outer one when the outer frame goes away; on MPArch, ownership of the `GuestPageHolder`
  transfers permanently to the embedder frame at attach. The API mirrors reality instead of
  pretending otherwise.

Lifetime:

- Removing the slot element, navigating the host frame, or destroying the host `WebContents`
  destroys the guest page. `detached` (with `reason`) fires, followed by `destroyed`. The
  `GuestPage` object remains as an inert handle (`isDestroyed() === true`), like a destroyed
  `webContents`.
- `destroy()` may be called by the app at any time; if attached, the slot's iframe simply becomes
  an empty `about:blank` frame again.
- This means a renderer action (removing a DOM node) can end the life of a main-process object.
  This is intrinsic to both underlying architectures and is documented prominently. Apps that need
  content to survive the host UI being torn down should use `WebContentsView`/`BrowserWindow`
  instead — the two APIs are complementary, not interchangeable.

### Host ↔ guest boundary

What the host page **can** do (all of it through the web platform, none of it Electron-specific):

- Lay out, clip, transform, occlude, hide, or remove the slot — full visual authority.
- `postMessage` to the guest's main frame (and receive messages back), with normal origin
  semantics. Chromium explicitly supports embedder ↔ guest-main-frame messaging across the
  embedding boundary.
- Focus the guest.

What the host page **cannot** do (enforced by Chromium, not by Electron-side checks):

- Navigate the guest (setting `iframe.src` or using the window proxy is dropped browser-side
  because the guest is in an unrelated browsing instance), close it, read its DOM, or synchronously
  script it.
- Read input delivered to the guest. Input events are routed by the browser's hit-testing directly
  to the guest's renderer, exactly as for cross-origin OOPIFs.
- Create, reconfigure, or destroy guests other than by removing slot elements that the main
  process chose to attach into.

The visual-authority point deserves emphasis as the one *intrinsic* risk this API does not remove:
embedding trusted, privileged UI inside an untrusted host page is a clickjacking-shaped mistake,
exactly as it is with iframes. Documentation will state plainly: do not embed content that is more
privileged than the page hosting it. (A native `WebContentsView` remains the right tool when the
embedded UI must not be occluded or restyled by other web content.)

### Popups and `window.open`

`guestPage.setWindowOpenHandler(handler)` is consulted for `window.open` and target=_blank
navigations originating in the guest:

- `{ action: 'deny' }` — the open is blocked (the app may open the URL itself by other means; no
  opener relationship exists in that case).
- `{ action: 'guest' }` — Electron creates a new, unattached `GuestPage` for the popup and emits
  `guest-created` on the opener with the new object and the open details. The app attaches it to
  another slot **in the same host WebContents**. The opener relationship (`window.opener`, named
  targeting, `postMessage`, `window.close()`) is preserved; the popup shares the opener's
  session/storage by construction. If the app never attaches it, it stays unattached (its
  navigation deferred) until it, the opener, or the host is destroyed.

Not supported: a popup that is simultaneously an independent OS window *and* retains a live
`window.opener` to the guest. Neither the current nor the MPArch guest architecture supports that
combination (Chromium's own `<webview>` has the same restriction), and `noopener`/
`opener_suppressed` opens always produce an unrelated context.

### Sessions and storage

Two configurations, in decreasing order of guarantee:

1. **Host's session (default).** Always supported. The guest shares the host's cookie jar and
   storage; isolation, if desired, comes from the web platform (different origins) rather than
   from Electron.
2. **A different Electron `session`.** Supported by the initial implementation (the guest is
   internally a `WebContents` created on that session's `BrowserContext`, exactly as `<webview
   partition="…">` works today). Under MPArch this configuration currently cannot work as-is:
   `GuestPageHolderImpl` constructs the guest frame tree with the *owner* WebContents'
   `BrowserContext`, and navigation-time consistency checks (and storage-partition resolution)
   assume the guest's `SiteInstance` belongs to that same `BrowserContext`. Continuing to support
   arbitrary cross-session guests therefore requires an upstream change (have `GuestPageHolder`
   honor the `BrowserContext` of the `SiteInstance` it is given). This is called out in
   *Unresolved questions*; the API shape (`session` option) does not change either way, but its
   documented support level might.

In both configurations the decision is made by the main process. The host renderer can no
longer choose which cookie jar embedded content runs against, which closes the `<webview>`
`partition`-attribute footgun (a compromised embedder minting arbitrary in-memory sessions).

### Implementation

#### Initial backend (current Chromium guest architecture)

- `new GuestPage()` creates an internal guest-typed `content::WebContents` (with a
  `BrowserPluginGuestDelegate`, as `<webview>` guests have today). It is **not** exposed as an
  `api::WebContents` / JS `webContents`; the `GuestPage` wrapper holds it directly.
- `attach(frame)` resolves the `WebFrameMain`'s `RenderFrameHost` in the browser process, verifies
  it is a live subframe of the host, and calls `AttachInnerWebContents` on the host `WebContents`
  — the same call `<webview>` uses today. No renderer-supplied tokens; no use of the
  `guest-view-manager` IPC surface; no `webviewTag` requirement on the host.
- `setWindowOpenHandler({ action: 'guest' })` creates the popup guest with the opener's
  `SiteInstance` (preserving the browsing instance), parallel to Chromium's pre-MPArch webview
  behavior.
- Events map to a `WebContentsObserver` on the internal guest; `mainFrame` is the existing
  `WebFrameMain` wrapper over the guest's main frame.
- No new Chromium patches are anticipated: nothing in Electron's current patch set touches
  `AttachInnerWebContents`, and the existing webview-adjacent patches (fullscreen propagation,
  cross-guest drag-and-drop, the two SiteInstance/StoragePartition consistency-check suppressions)
  already cover the behaviors this backend exercises.

#### MPArch backend (when `features::kGuestViewMPArch` ships)

- `new GuestPage()` creates a `content::GuestPageHolder` (`GuestPageHolder::Create`) with a
  `GuestPageHolder::Delegate` implemented by Electron; `attach()` uses
  `RenderFrameHost::PrepareForInnerWebContentsAttach` + `WebContents::AttachGuestPage`. All of
  this is public `content/` API and does not require the `components/guest_view` extensions layer.
- Guaranteed-tier members map as follows: navigation → the holder's `NavigationController`;
  `mainFrame` → the holder's guest main `RenderFrameHost`; audio mute → the holder; events → the
  delegate callbacks plus the host-WebContents observer filtered on guest main-frame navigations;
  popups → the `GuestCreateNewWindow` delegate method (which both architectures route through).
- Electron-side work this backend requires (shared with the eventual `<webview>` migration, not
  specific to `GuestPage`):
  - **Per-guest preferences and preload delivery.** Electron currently keys preload scripts,
    sandbox/context-isolation decisions, and `OverrideWebkitPrefs` off the `WebContents`; for an
    MPArch guest those lookups would resolve to the *host*. They need to be keyed off the guest
    page (Chromium already provides per-guest renderer/web preferences hooks on the holder).
  - **Permission attribution.** Permission requests from guest frames arrive on the host
    `WebContents` with the guest frame as the requesting frame. Electron's permission-handler
    details will identify the `GuestPage` explicitly so apps key decisions off the guest's origin
    rather than the host's `webContents`.
  - **Session/storage decision** per the previous section.

#### What stays out of Electron's hands

Bounds, z-order, clipping, scrolling, occlusion, and input routing are all handled by Blink layout
and Chromium's cross-process frame embedding in both backends. Electron ships no geometry code,
no coordinate IPC, and no compositor integration for this feature.

## Drawbacks

- **Another embedding primitive.** Electron would document `<iframe>`, `<webview>` (legacy),
  `WebContentsView`, and `GuestPage`. The mitigation is that `GuestPage` has a crisp answer to
  "when should I use which": layout-integrated, host-page-embedded content → `GuestPage`;
  window-level, native-stacked content → `WebContentsView`; and `<webview>` becomes "legacy, use
  GuestPage".
- **It builds on guest infrastructure that upstream is actively rebuilding.** The API is specified
  to survive the MPArch migration, but the implementation will need a second backend, and some
  provisional features depend on upstream finishing pieces that are currently `NOTIMPLEMENTED()`.
  There is schedule risk outside Electron's control.
- **Host visual authority is a foot-gun in the opposite direction from `<webview>`'s.** The classic
  webview risks were "the renderer can escalate"; the GuestPage risk is "developers embed trusted
  UI inside untrusted hosts and get clickjacked". This is inherent to the feature (it is the
  feature) and must be addressed with documentation and security-checklist guidance.
- **Renderer-influenced lifetime.** A host page removing a DOM node destroys a main-process
  object. Apps with main-process state hanging off a `GuestPage` must handle `detached`/`destroyed`
  robustly.
- **Per-guest preference plumbing under MPArch is real work.** It is, however, work Electron must
  do anyway to keep `<webview>` alive past the migration; this RFC just makes it load-bearing for
  a supported API rather than a deprecated one.

## Rationale and alternatives

### Alternative: anchor-element bounds syncing for `WebContentsView`

A built-in mechanism could track a host DOM element's rectangle (via the layout pipeline, similar
to how draggable regions are collected, or an autofill-style renderer agent) and apply it to a
`WebContentsView`'s `setBounds()`. This keeps the embedded content a native view and is a useful
feature in its own right, but it cannot satisfy the core requests this RFC targets:

- the native view is always stacked above the host page's content — z-index against host content
  is impossible by construction;
- it cannot be clipped by host overflow or transformed with host CSS;
- compositor-thread scrolling in the host does not run main-thread layout, so the view visibly
  lags any anchor inside a scroller;
- resize of web content is asynchronous, so continuous resizes show gutters.

It also leaves the `<webview>` succession problem unsolved. It may still be worth pursuing
separately for the "panes/sidebars laid out by CSS" use case; it is not a substitute.

### Alternative: keep improving `<webview>`

The tag's problems are structural: creation and control are renderer-initiated, the configuration
surface is a string attribute parsed without an allowlist, and the safety story depends on every
app implementing `will-attach-webview` correctly. Hardening it further still leaves a
renderer-reachable capability that most apps should not grant, and the tag still has to be
re-implemented for MPArch. Building the main-process-driven API first, then deprecating and
removing the tag, is the more honest sequencing.

### Alternative: userland (placeholder element + IPC + `setBounds`)

Works today, requires no Electron changes, and is the right call for simple cases. Its limitations
(latency, always-on-top, no clipping) are inherent and are precisely the gap this RFC fills.

### Alternative: do nothing

Apps keep choosing between a deprecated-in-practice tag and a positioning hack. The MPArch
migration eventually forces a webview reckoning anyway, but without a designed successor.

### Why not a JavaScript module / native addon?

The feature requires attaching content at the `content::WebContents`/frame-tree level and
participating in Chromium's guest plumbing; it cannot be built outside Electron core.

## Prior art

- **Electron `<webview>`** — the existing DOM-integrated embedding mechanism; this proposal keeps
  its compositing model and discards its renderer-driven control model. Years of security guidance
  (`will-attach-webview`, "verify webview options", "do not use allowpopups") document the cost of
  that model.
- **Chrome Apps `<webview>` and the WICG Controlled Frame proposal** — the same embedder-attaches-
  guest architecture, including the "newwindow"/pending-guest flow this RFC adopts for popups.
  Controlled Frame is also the upstream driver for keeping the guest concept alive post-MPArch.
- **Chromium MPArch GuestView migration** (`features::kGuestViewMPArch`, `content::GuestPageHolder`,
  `WebContents::AttachGuestPage`) — the architectural target this API is specified against.
- **Electron `BrowserView` → `WebContentsView`** — precedent for replacing an API whose underlying
  model had decayed with one aligned to the current Chromium architecture, while keeping a
  compatibility shim.
- **Userland bounds-syncing** — many Electron apps ship a "place a BrowserView/WebContentsView over
  a placeholder div" layer; their common pain points (scroll lag, z-order, clipping) shaped the
  motivation above.

## Unresolved questions

To resolve during the RFC process:

1. **Cross-session guests under MPArch.** Should Electron (a) pursue an upstream change so
   `GuestPageHolder` honors the `BrowserContext` of the `SiteInstance` it is given, or (b)
   restrict the `session` option to the host's session when the MPArch backend is active? This
   affects documentation and the `<webview>` migration more than it affects this API's shape.
2. **Popup ergonomics.** Is `{ action: 'guest' }` + `guest-created` the right surface, or should
   the handler return the new `GuestPage` directly? How long may a pending popup stay unattached
   before Electron warns?
3. **Naming.** `GuestPage` vs alternatives.

To resolve during implementation, before stabilization:

4. **Per-guest preference/preload delivery** keyed off the guest page rather than the host
   `WebContents` (shared with the `<webview>` MPArch migration).
5. **Permission attribution details** — the exact shape of handler details identifying the
   `GuestPage`, and verification that guests remain their own permission "top level" under MPArch.
6. **Promotion of provisional members** (devtools, capture, find, print, zoom, input injection)
   as upstream MPArch support and Electron plumbing land.

Out of scope for this RFC:

- Deprecating and removing the `<webview>` tag (future RFC once `GuestPage` is stable).
- Attaching existing `webContents`/`WebContentsView` objects into host pages.
- An anchor-element bounds-syncing mechanism for native `WebContentsView`s.

## Future possibilities

- **`<webview>` deprecation and removal.** Once `GuestPage` is stable and migration guidance
  exists, deprecate the tag and remove it in a later major version (a separate RFC).
- **Controlled-Frame-style capabilities** (request interception scoped to the guest, content
  scripts) built on the per-guest session story.
- **Per-guest devtools, capture, and printing** as the MPArch delegate surface fills out.
- **Slot helpers** — an official, capability-free custom element (`<guest-slot>`) published as a
  package for ergonomics and accessibility defaults around the placeholder iframe.
