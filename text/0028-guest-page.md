# GuestPage: main-process-driven embedding of web content into a host page

- Start Date: 2026-05-26
- RFC PR: [electron/rfcs#0030](https://github.com/electron/rfcs/pull/0030)
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

A `GuestPage` has no public constructor. It is created by attaching into a placeholder frame —
`webFrameMain.attachGuestPage(options)` — which is also the moment the guest's host, session, and
lifetime become known. The API is deliberately specified so that it can be implemented on top of
today's inner-`WebContents` attachment mechanism (`WebContents::AttachInnerWebContents`, the same
primitive `<webview>` uses) *and* on Chromium's in-progress MPArch GuestView architecture
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

- **Slot** — an ordinary `<iframe>` element in the host page (typically `about:blank`, typically
  wrapped in whatever app component makes sense). The slot has no special powers and requires no
  Electron API in the renderer; it is simply where the guest appears. The host page controls the
  slot's size, position, stacking, clipping, and visibility through normal CSS — that is the
  feature.
- **GuestPage** — a main-process object representing an embedded web page. Conceptually similar to
  a `webContents` you control yourself, except that it renders inside a host page's slot instead
  of inside a window, and its API surface is intentionally smaller. There is **no public
  constructor**: a `GuestPage` is created by attaching into a slot
  (`webFrameMain.attachGuestPage(options)`), or handed to you when a guest opens a popup. This is
  deliberate — a guest only ever exists in the context of a host, and the API shape says so.

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
// Find the slot frame in the host WebContents and attach a guest into it.
const slot = hostContents.mainFrame.frames.find((f) => f.name === 'docs-pane')

const docs = await slot.attachGuestPage({
  // Defaults to the host's session; set explicitly if you need otherwise.
  session: session.fromPartition('persist:docs'),
})

await docs.loadURL('https://docs.example.com')

// Control and observe the guest from the main process, as usual.
docs.on('did-navigate', (event, url) => updateAddressBar(url))
docs.mainFrame.send('set-theme', 'dark')
docs.setWindowOpenHandler(({ url }) => {
  shell.openExternal(url)
  return { action: 'deny' }
})

docs.on('destroyed', ({ reason }) => {
  // 'slot-removed' | 'host-navigated' | 'host-destroyed' | 'app-destroyed'
})
```

The host page never sees a new API. If the host page is compromised, the only things it can do to
the guest are things it can already do to its own DOM: move, resize, occlude, hide, or remove the
slot, focus it, or `postMessage` to it (subject to the usual origin checks). It cannot create
guests, choose what they load, navigate them, read their content, or inject input into them.

### Mental model versus `<webview>`

| | `<webview>` | `GuestPage` |
| --- | --- | --- |
| Who creates the guest | host renderer (tag insertion) | main process (`slotFrame.attachGuestPage()`) |
| Who chooses URL, session, preload, prefs | host renderer attributes (+ `will-attach-webview` veto) | main process |
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
  // The popup GuestPage exists but is not yet placed. Ask the host page (via your
  // own IPC) to add another slot, then adopt the popup into it.
  const slotName = await requestPopupSlot(hostContents, details)
  const slot = hostContents.mainFrame.frames.find((f) => f.name === slotName)
  await slot.attachGuestPage({ guest: popup })
})
```

With `{ action: 'guest' }`, `window.opener`, named targeting, and `postMessage` between the opener
guest and the popup guest work as they do for normal pages, **provided the popup is attached to a
slot in the same host `WebContents`**. This mirrors how Chromium's own `<webview>` "newwindow"
flow works on both the current and the MPArch guest architectures.

`GuestPage` does not offer "the popup becomes an independent `BrowserWindow` and keeps a live
`window.opener` back to the guest." The current architecture can technically do this (it is what
`<webview allowpopups>` + `window.open` produces today), but the MPArch architecture cannot — a
popup with a live opener can only ever be another guest page in the same host — so offering it
would bake in a promise the API could not keep across the migration. Apps that want an external
window can return `{ action: 'deny' }` and open the URL themselves (without an opener
relationship).

### Sessions and storage

- By default a `GuestPage` uses the **host's session**. This is the guaranteed, forever-supported
  configuration.
- A different `session` may be passed to `attachGuestPage()`. This works on the initial
  implementation (see Reference-level explanation), and is the same capability
  `<webview partition="…">` has today. Its long-term shape under MPArch depends on an upstream
  question about guest `BrowserContext`s that this RFC calls out explicitly in *Unresolved
  questions*; apps should not assume that arbitrary cross-session guests will remain expressible
  in exactly this form.
- Either way, the cookie jar and storage the guest uses are decided by the main process, never by
  the host renderer (unlike `<webview>`'s renderer-chosen `partition` attribute).

### Migrating from `<webview>`

| `<webview>` feature | `GuestPage` equivalent |
| --- | --- |
| `src` attribute | `guestPage.loadURL()` |
| `partition` attribute | `session` option of `attachGuestPage()` (main-chosen) |
| `preload` attribute | `preload` option of `attachGuestPage()` (main-chosen; provisional tier) |
| `nodeintegration`, `webpreferences`, `disablewebsecurity`, … | not available to the host page at all, and not part of GuestPage v1 — guests always run with Electron's default, sandboxed renderer configuration; per-guest `preload` is the only planned override |
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

There is no public constructor. A `GuestPage` comes into existence in exactly two ways:

1. `webFrameMain.attachGuestPage(options)` — creates a guest and attaches it into that frame
   (the "slot"). This is the only way for an app to create one, and it is also the moment the
   host, the session, and the guest's lifetime scope are determined. Both underlying
   architectures require the owner/host to be known when a guest is created, so the API shape
   simply reflects reality instead of simulating a free-standing object.
2. The `guest-created` event — when a guest's `window.open` is answered with
   `{ action: 'guest' }`, Electron creates a *pending* (not yet placed) `GuestPage` for the popup
   and hands it to the app, which places it with `slotFrame.attachGuestPage({ guest: popup })`.

The API surface is split into two tiers:

- **Guaranteed** — implementable on both the current architecture and MPArch using public
  `content/` APIs that exist today (in some cases with non-trivial Electron-side plumbing, which
  the Implementation section describes). These ship with the first version.
- **Provisional** — desirable, and mostly implementable on the current architecture, but dependent
  on parts of the MPArch guest implementation that are incomplete upstream, on upstream API that
  does not exist yet, or on Electron-side per-guest plumbing shared with the future `<webview>`
  migration. These ship behind clearly-documented experimental status (or in follow-ups) so that
  the guaranteed tier never has to regress.

#### `webFrameMain.attachGuestPage(options)`

```ts
frame.attachGuestPage(options?: {
  session?: Session              // default: the host's session
  guest?: GuestPage              // adopt a pending popup GuestPage instead of creating a new one
  // Provisional tier:
  preload?: string               // absolute path, main-chosen
  transparent?: boolean
  backgroundColor?: string
}): Promise<GuestPage>
```

- Returns once the guest is created and attached. With `guest`, no other creation options may be
  passed (the popup's session and configuration were fixed when it was created from its opener).
- The frame it is called on becomes the slot. After attachment the slot frame's original document
  is gone (the placeholder frame is swapped for the embedding boundary); the `WebFrameMain` handle
  the method was called on should be considered consumed — use the returned `guestPage.mainFrame`
  to reach the embedded content.

#### Guaranteed methods, properties, and events

| Member | Notes |
| --- | --- |
| `loadURL(url, options)` / `reload()` | |
| `url`, `getTitle()` | |
| `navigationHistory` | Exposes the same documented `NavigationHistory` shape as `webContents.navigationHistory` (an object over the guest's NavigationController; not a shared class instance). |
| `mainFrame: WebFrameMain` | Per-frame surface: `executeJavaScript`, `send()`/`postMessage()`, `frames`, `origin`, process info. |
| `session: Session` | Read-only. |
| `isAttached()`, `isDestroyed()`, `destroy()` | `isAttached()` is false only for a pending popup guest that has not been adopted yet. See *Lifetime* for `destroy()` semantics. |
| `setAudioMuted()` / `isAudioMuted()` | |
| `setWindowOpenHandler(handler)` | Actions: `'deny'`, `'guest'` (see *Popups*). |
| Events: `did-start-loading`, `did-stop-loading`, `did-finish-load`, `did-fail-load`, `did-navigate`, `did-navigate-in-page`, `dom-ready`, `page-title-updated`, `render-process-gone`, `destroyed`, `guest-created` | |

#### Provisional members

`preload`, `transparent`, `backgroundColor`, `stop()`, `setZoomFactor()`/`getZoomFactor()`,
`page-favicon-updated`, `openDevTools()`, `capturePage()`, `findInPage()`, `printToPDF()`,
`sendInputEvent()`, fullscreen-related events.

Each of these has a workable implementation today (the guest is a real `WebContents` internally),
but does not yet clear the bar for the guaranteed tier on MPArch, for one of these reasons:

- **No public per-guest equivalent exists yet upstream**: `stop()` (there is no per-guest stop —
  `NavigationController` has no `Stop()` and `WebContents::Stop()` would also stop the host),
  `transparent`/`backgroundColor` (the `GuestPageHolder` interface exposes no per-guest visual
  configuration today).
- **The MPArch equivalent is per-frame or page/widget-level plumbing that is not finished
  upstream**: zoom and devtools (per-frame), fullscreen, pointer lock, downloads, find-in-page for
  unattached guests, and the page/widget-level operations `capturePage()`, `printToPDF()`,
  `sendInputEvent()`, and favicon delivery.
- **It requires Electron-side per-guest preference/preload delivery that does not exist yet**
  (shared with the `<webview>` MPArch migration): `preload`.

Promising these as guaranteed now risks an API break when the backend changes, so they are
explicitly marked.

#### Deliberately excluded

- `guestPage.webContents` — never. This is the single most important exclusion; everything else
  follows from it.
- A public constructor, or any way to create a `GuestPage` without a host. Both architectures
  require the owner at guest-creation time, and a host-less guest would reintroduce exactly the
  lifetime ambiguity this API is trying to avoid.
- Any renderer-facing API: no tag, no creation IPC, no method-forwarding channel, no
  `webviewTag`-style preference. (An optional, capability-free custom element that wraps a slot
  `<iframe>` for ergonomics may be published as a separate package or documentation recipe; it is
  not part of this proposal.)
- Adoption of an *existing* `webContents` or `WebContentsView` into a slot. Neither architecture
  supports this cleanly: the current one requires guests to be created as guests up front, and
  MPArch has no `WebContents` to adopt. Out of scope permanently unless upstream direction
  changes.

### Attachment semantics

`webFrameMain.attachGuestPage(options)`:

- The frame must be a **subframe** (never a main frame) of some `WebContents`, must be current and
  live, should be an `about:blank` document (matching the guidance in the Chromium attach APIs),
  and must not itself be inside a guest page.
- Validation happens entirely in the browser process against a main-process-supplied frame handle,
  *before* calling into the content attach APIs (which enforce their own invariants with
  process-level CHECKs). There are no renderer-supplied identifiers anywhere in the path — unlike
  `<webview>`, whose attach IPC carries a renderer-supplied frame token whose ownership is only
  `DCHECK`ed.
- One guest per slot, one slot per guest. Calling `attachGuestPage()` on an occupied slot, or
  adopting an already-placed guest, rejects.
- There is **no `detach()`** and no re-parenting. Neither architecture can preserve a guest across
  detachment from its outer frame: on the current architecture the inner `WebContents` is owned
  and destroyed by the outer one when the outer frame goes away; on MPArch, ownership of the
  `GuestPageHolder` transfers permanently to the embedding frame at attach. The API mirrors
  reality instead of pretending otherwise.

### Lifetime

- Removing the slot element, navigating the host frame, or destroying the host `WebContents`
  destroys the guest page. `destroyed` fires with a `reason` of `'slot-removed'`,
  `'host-navigated'`, or `'host-destroyed'`. The `GuestPage` object remains as an inert handle
  (`isDestroyed() === true`), like a destroyed `webContents`.
- `destroy()` may be called by the app at any time and fires `destroyed` with reason
  `'app-destroyed'`. Destroying an attached guest tears down the embedding: the slot `<iframe>` is
  left without a content document, and the host page must recreate (or re-navigate) the iframe
  element if it wants a usable slot again — the slot is not silently restored to `about:blank`.
  (Implementation note: neither architecture has a direct "destroy the attached guest" API;
  teardown goes through removal of the embedding frame node, which is also how `<webview>`
  replacement works today. Electron's existing helper for this, `WebContents::DetachFromOuterFrame`,
  has a broken guard — inverted during the Chromium 131 `FrameTreeNodeId` migration — and needs
  repair as part of this work.)
- A pending popup guest that is never adopted is destroyed when the handler denies it, when its
  opener guest is destroyed, or when the host goes away.
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
- Read input delivered to the guest. Pointer input is routed by the browser's hit-testing and
  keyboard input by browser-side focus tracking, directly to the guest's renderer — exactly as for
  cross-origin OOPIFs; the host renderer never sees it.
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
- `{ action: 'guest' }` — Electron creates a new, pending `GuestPage` for the popup and emits
  `guest-created` on the opener with the new object and the open details. The app places it with
  `slotFrame.attachGuestPage({ guest: popup })`, where the slot must be **in the same host
  WebContents**. The opener relationship (`window.opener`, named targeting, `postMessage`,
  `window.close()`) is preserved; the popup shares the opener's session/storage by construction.
  If the app never adopts it, it stays pending (its navigation deferred) until it, the opener, or
  the host is destroyed.

Not offered: a popup that becomes an independent OS window *while retaining* a live
`window.opener` back to the guest. The current architecture can produce that combination (it is
what `<webview allowpopups>` + `window.open` does today), but the MPArch architecture cannot — a
popup that keeps its opener can only ever be another guest page in the same host (Chromium's own
`<webview>` has the same restriction) — so `GuestPage` does not offer it on either backend rather
than ship a behavior that the migration would have to break. `noopener`/`opener_suppressed` opens
always produce an unrelated context.

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

- `attachGuestPage()` resolves the slot `WebFrameMain`'s `RenderFrameHost` in the browser process,
  verifies it is a live subframe of the host (and not inside another guest), creates an internal
  guest-typed `content::WebContents` (with a `BrowserPluginGuestDelegate`, as `<webview>` guests
  have today) on the chosen session, and calls `AttachInnerWebContents` on the host `WebContents`
  — the same call `<webview>` uses today. Creating and attaching in one step matters: the current
  architecture requires the owner WebContents at guest-creation time (`BrowserPluginGuest::Init`
  runs during `WebContents::Create` and dereferences the owner), which is exactly why the API has
  no host-less constructor. Resolving the frame from `WebFrameMain` (rather than reusing the
  `<webview>` frame-token helper) also removes that helper's assumption that the slot frame lives
  in the host's main-frame process.
- The internal guest WebContents is **not** exposed as an `api::WebContents` / JS `webContents`.
  This keeps the surface migration-safe, but it means GuestPage needs its own browser-side
  plumbing for things Electron currently hangs off `api::WebContents`: a `WebContentsObserver` to
  drive the event list, delivery of frame-scoped IPC from guest frames to `ipcMain` (today the IPC
  handler drops messages for contents with no `api::WebContents` wrapper), the window-open gate
  that currently consults the opener's `WebContentsPreferences`, and registration with Electron's
  `WebViewManager`/fullscreen propagation paths where guests are expected. None of this requires
  Chromium changes, but it is the bulk of the Electron-side work.
- `setWindowOpenHandler({ action: 'guest' })` creates the popup guest with the opener's
  `SiteInstance` (preserving the browsing instance) via the existing
  `BrowserPluginGuestDelegate::CreateNewGuestWindow` hook, defers its navigation until adoption
  (`ResumeLoadingCreatedWebContents`), and hands it to the app as a pending `GuestPage` — the same
  flow Chromium's pre-MPArch `<webview>` "newwindow" implementation uses. This is new Electron
  code (today's webview popups become top-level windows instead), and the guest `window.open` path
  relies on the two SiteInstance/StoragePartition consistency-check suppressions Electron already
  carries for `<webview>`.
- Teardown of an attached guest goes through removal of the embedding frame node;
  `WebContents::DetachFromOuterFrame` is repaired (its guard was inverted in the Chromium 131
  bump) and reused for this.
- No new Chromium patches are anticipated: nothing in Electron's current patch set touches
  `AttachInnerWebContents`, and the existing webview-adjacent patches (fullscreen propagation,
  cross-guest drag-and-drop, the two consistency-check suppressions) are generic content-level
  changes that cover the behaviors this backend exercises.

#### MPArch backend (when `features::kGuestViewMPArch` ships)

- `attachGuestPage()` creates a `content::GuestPageHolder` (`GuestPageHolder::Create`, which takes
  the owner WebContents — again matching the factory shape) with a `GuestPageHolder::Delegate`
  implemented by Electron, then uses `RenderFrameHost::PrepareForInnerWebContentsAttach` +
  `WebContents::AttachGuestPage`. All of this is public `content/` API and does not require the
  `components/guest_view` extensions layer.
- Guaranteed-tier members map as follows: navigation → the holder's `NavigationController`;
  `mainFrame` → the holder's guest main `RenderFrameHost`; audio mute → the holder; popups → the
  `GuestCreateNewWindow` delegate method (the legacy backend's equivalent hook is
  `BrowserPluginGuestDelegate::CreateNewGuestWindow`); events → `GuestPageHolder::Delegate`
  callbacks plus a `WebContentsObserver` on the host filtered to guest-main-frame navigations,
  with three known pieces of non-obvious plumbing: `did-start-loading` is synthesized from
  `DidStartNavigation` (the holder's loading-state callback is not implemented upstream yet),
  `page-title-updated` comes from `TitleWasSetForMainFrame` (the primary-frame `TitleWasSet`
  observer method does not fire for guests), and `render-process-gone`'s exit code comes from a
  `RenderProcessHostObserver` on the guest's process (the delegate callback only carries the
  termination status).
- Because guest frames live inside the host's `WebContents`, Electron must also filter its
  host-facing APIs so guest frames do not leak into them (the host's `dom-ready`/`frame-created`
  events, `mainFrame.framesInSubtree`, etc.) — the inverse of the plumbing that makes
  `guestPage.mainFrame` work.
- Electron-side work this backend requires (shared with the eventual `<webview>` migration, not
  specific to `GuestPage`):
  - **Per-guest preferences and preload delivery.** Electron currently keys preload scripts,
    sandbox/context-isolation decisions, and `OverrideWebkitPrefs` off the `WebContents`; for an
    MPArch guest those lookups would resolve to the *host*. They need to be keyed off the guest
    page (Chromium already provides per-guest renderer/web preferences hooks on the holder).
  - **Permission attribution.** Electron's permission-handler details will identify the
    `GuestPage` and the requesting guest frame explicitly, so apps key decisions off the guest's
    origin rather than the host's `webContents`; the exact routing of permission requests from
    guest frames is one of the things to verify during implementation (Unresolved question 5).
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
- **The guest is deliberately not an `api::WebContents`, so Electron must duplicate plumbing.**
  Event emission, frame-scoped IPC delivery, window-open gating, and guest-manager registration
  all currently assume an `api::WebContents` wrapper exists; GuestPage needs parallel paths. This
  is the main implementation cost of the migration-safe surface.
- **Host visual authority is a foot-gun in the opposite direction from `<webview>`'s.** The classic
  webview risks were "the renderer can escalate"; the GuestPage risk is "developers embed trusted
  UI inside untrusted hosts and get clickjacked". This is inherent to the feature (it is the
  feature) and must be addressed with documentation and security-checklist guidance.
- **Renderer-influenced lifetime.** A host page removing a DOM node destroys a main-process
  object. Apps with main-process state hanging off a `GuestPage` must handle `destroyed` robustly.
- **Per-guest preference plumbing under MPArch is real work.** It is, however, work Electron must
  do anyway to keep `<webview>` alive past the migration; this RFC just makes it load-bearing for
  a supported API rather than a deprecated one.

## Rationale and alternatives

### Why a factory on `WebFrameMain` instead of a constructor

Both guest architectures require the owner/host to be known at the moment a guest is created (the
current one initializes the guest against its owner during `WebContents::Create`; MPArch's
`GuestPageHolder::Create` takes the owner WebContents). A free-standing `new GuestPage()` would
therefore have to either secretly defer creation until attach — making pre-attach `loadURL()` and
the "default to the host's session" option polite fictions — or demand a host argument anyway.
`frame.attachGuestPage()` makes the real lifecycle the visible one: a guest exists only in the
context of a host, its session default is resolvable at the call site, and the returned object is
live from the moment the app gets it. The cost is that content cannot be "pre-warmed" before a
slot exists; neither backend reliably supports that anyway (MPArch defers a guest's first
navigation until attach), so the API declines to promise it.

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
2. **Popup adoption ergonomics.** Is `guest-created` + `slotFrame.attachGuestPage({ guest })` the
   right surface, or should the handler be able to name a slot directly? Should Electron warn if a
   pending popup is never adopted?
3. **Naming.** `GuestPage` / `attachGuestPage` vs alternatives.

To resolve during implementation, before stabilization:

4. **Per-guest preference/preload delivery** keyed off the guest page rather than the host
   `WebContents` (shared with the `<webview>` MPArch migration).
5. **Permission attribution** — verify how permission requests from guest frames are routed on
   each backend, and shape the handler details so apps can identify the `GuestPage` and the
   requesting origin (and so guests remain their own permission "top level" under MPArch).
6. **Promotion of provisional members** (stop, transparency/background, devtools, capture, find,
   print, zoom, input injection) as upstream MPArch support and Electron plumbing land — including
   whether a per-guest `Stop()` is worth proposing upstream.

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
- **Detached creation (`GuestPage.createDetached()`)** for pre-warming content before a slot
  exists, if a real need emerges and the backends grow support for unattached guests with live
  navigation.
- **Slot helpers** — a capability-free custom element (`<guest-slot>`) published as a separate
  package for ergonomics and accessibility defaults around the placeholder iframe.
