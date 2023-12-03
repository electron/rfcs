- Start Date: 2023-11-12
- RFC PR: [electron/rfcs#0000](https://github.com/electron/rfcs/pull/0000)
- Status: **Proposed**

# Preload Realm for Service Workers

## Summary

Preload realms provide an isolated context to run JavaScript prior to
code evaluating in another context. This expands functionality of 
Electron's preload scripts to provide new context targets. This proposal
intends to target Service Worker contexts with potential plans for new
contexts (web workers, shared workers) in future proposals.

## Motivation

[Electron's Chrome extension support](https://www.electronjs.org/docs/latest/api/extensions) adds a subset of APIs to support the use case of DevTools-related extensions. Extending this to support the full API would not only add a large maintenence burdon, but might impose and restrict design of features such as [extension action buttons.](https://developer.chrome.com/docs/extensions/reference/action/)

Expanding preload scripts to allow targeting of new contexts such as Service Workers would allow the community to fill in the gaps for Chrome extension APIs as well as entirely new ideas. Preload "realms" would provide a basis for this functionality.

In addition, others have expressed desire for preload support in worker contexts:
- https://github.com/electron/electron/issues/28620
  > there is no way to fix issues like Electron's broken atob/btoa implementation, or hiding APIs such as battery/keyboard/gpuinfo from being accessed [...] in workers
- https://github.com/electron/electron/issues/39848
  > `nodeIntegrationInWorker: true` does not cause loading of preload script in workers

Although the initial goal of this RFC is to support Service Workers, I anticipate support of web workers can be added with relative ease following its completion.

## Guide-level explanation

A preload realm is an isolated JavaScript context—without access to the DOM—which exists alongside a worker context. It offers a way to safely interact with the worker context in order to add new APIs which can communicate with Electron's main process.

This functionality can be used to transform network requests, prototype new extension APIs, or enable a new method of background data synchronization to name a few use cases.

To demonstrate the API, we'll take a look at how we can use Preload Realms to prototype a new extension API.

### Creating a new extension API

Chrome's Manifest V3 extensions provide event routing and browser APIs through a [background service worker.](https://developer.chrome.com/docs/extensions/mv3/service_workers/) While leveraging the existing APIs provided by Chrome, we can add our own to extend its full set of functionality.

Given the recent rise of generative AI, we'll look at how we might offer extensions the ability to summarize text using one in Electron.

We can start by writing a script to run in a preload realm targeting extension service workers.
```js
// extension-sw-preload.js
const { contextBridge, ipcRenderer } = require('electron');

// Check that our preload script was initiated by an extension service worker.
if (process.type === 'worker' &&
    process.scriptUrl.startsWith('chrome-extension:')) {
  // Expose our API to the service worker!
  exposeApi();
}

function exposeApi() {
  const api = {
    ai: {
      summarize: (text) => {
        // Send an IPC to Electron's main process to summarize the text.
        return ipcRenderer.invoke('SUMMARIZE_TEXT', text);
      }
    }
  };

  // Expose our API in the JS context which initiated the creation of this
  // preload realm.
  contextBridge.exposeInInitiatorWorld('myElectronApi', api);
}
```

With our preload realm script ready, we can write an extension service worker script to use it.
```js
// extension-sw-script.js

// Listen for messages which request summarizing of text.
chrome.runtime.onMessage.addListener(
  async function(request, sender, sendResponse) {
    if (request.type === 'summarize') {
      // Invoke our new API.
      const summary = await myElectronApi.ai.summarize(request.payload);
      sendResponse({ text: summary });
    };
  }
);
```

In the main process, we'll handle the IPC request with some additional checks to validate the sender.
```js
// main.js
const { ipcMain } = require('electron');

// Handle the IPC sent from the service worker preload realm.
ipcMain.handle('SUMMARIZE_TEXT', (event, text) => {
  // We want to validate that our IPC was sent from an extension running in a
  // service worker context and that the text is of the expected type.
  const isValid = (event.senderType === 'service-worker') &&
    event.senderWorker.url.startsWith('chrome-extension:') &&
    typeof text === 'string';

  if (!isValid) {
    throw new Error('Invalid request');
  }

  // Call out to an LLM to summarize the text :)
  return gptSummarize(text);
});

// Add our preload realm script to the session.
session.defaultSession.setPreloadRealmScripts([
  {
    // Our script should only run in service worker preload realms.
    type: 'service-worker',
    // The absolute path to the script.
    script: path.join(__dirname, 'extension-sw-preload.js'),
  }
]);
```

Finally, as an example, we can summarize the text of the web document as soon
as it completes loading.
```js
// content-script.js
window.addEventListener('load', () => {
  const response = await chrome.runtime.sendMessage({
    type: 'summarize',
    payload: document.body.innerText
  });
  console.log('summary', response.text);
});
```

## Reference-level explanation

### Preload realm

Chrome extension content scripts introduced the concept of running code in an isolated world where V8 globals are not shared between contexts. Electron's [preload scripts](https://www.electronjs.org/docs/latest/tutorial/tutorial-preload) build on the same technology to isolate Electron code from the main document's context. This explanation will expand on the ideas behind these technologies.

> [!NOTE]
> For a primer on V8 isolates, contexts, and worlds, refer to the [Design of V8 bindings](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/bindings/core/v8/V8BindingDesign.md;drc=eb90eb0f510ccf69c0ac8043e8cfe608fea8f41b) guide in Chromium.

A **preload realm** is an isolated `v8::Context` which is created in response to another context being created in a renderer. A `content::ContentRendererClient` exposes hooks for when a new context is created and ready for evaluation.

[`content::ContentRendererClient::WillEvaluateServiceWorkerOnWorkerThread`](https://source.chromium.org/chromium/chromium/src/+/main:content/public/renderer/content_renderer_client.h;l=349-358;drc=1d4f50793406146e0d40df7063e01787cf5e7426) provides a hook for when a service worker context will be evaluated. This will be used to create a preload realm for service worker contexts.

#### ShadowRealm proposal

[ShadowRealms](https://github.com/tc39/proposal-shadowrealm/blob/main/explainer.md) are an ECMAScript TC39 Stage 3 proposal which introduces the ability to create new isolated contexts from any existing JS context. Although not yet finalized, there's an existing implementation available in V8 and Chromium which can be used as a reference.

> [!NOTE]
> It's possible to test ShadowRealms in Electron by adding a V8 flag:
> ```js
> app.commandLine.appendSwitch('js-flags', '--harmony-shadow-realm');
> ```

In Chromium, an [implementation for a V8 hook named `OnCreateShadowRealmV8Context` is provided.](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/bindings/core/v8/shadow_realm_context.cc;l=63;drc=f092aac6779095fcb906f43fdcd223cd7148483f) In this hook, a new `v8::Context` is created with a `ShadowRealmGlobalScope` blink execution context. Using blink APIs would allow the context to interact with debugger agents to provide a DevTools debugging experience-although not currently exposed in Chrome.

The implementation of ShadowRealms provides a basis for the design of Preload Realms.

### Preload realm scripts

TODO
- adding to session
- synchronous IPC to gather scripts

### `contextBridge`

A preload realm interacts between an initiator context and the preload realm context. The goal of bridging objects between two contexts remains the same, but may require new terminologies. A "main world" might not make sense for workers so I've elected to use "initiator world" instead.

In the case of render frames, `webFrame.executeJavaScript` exists as a way to evaluate JS in the main world. Workers will require the same functionality which could be added without introducing another top-level module.

```ts
interface ContextBridge {
  exposeInInitiatorWorld(key: string, api: any);
  evaluateInInitiatorWorld(code: string);
}
```

To avoid hard crashes while invoking existing methods (ie. `exposeInMainWorld`), they can be modified to throw if the current V8 context isn't associated with a render frame.

### `ipcRenderer` <=> `ipcMain`

IPCs in Electron have historically only been transmitted between the main process and render frames in renderer processes.

While web workers can exist in the same process as render frames, service workers live in their own process on a worker thread. IPC handlers in the main process will need to be aware of multiple types of senders for messages received.

#### ServiceWorkerMain

`WebContents`-and recently `WebFrameMain`-has served as the primary class for transmitting IPC messages. Service Workers will require a new class to facilitate messages to SWs in the renderer process.

Chrome's extension system uses [`ServiceWorkerHost`](https://source.chromium.org/chromium/chromium/src/+/main:extensions/browser/service_worker/service_worker_host.h;drc=c59bc6c0e4ae58869eae5f4fa911840a56b647ff) which will serve as inspiration.

```ts
/** JS wrapper */
class ServiceWorkerMain {
  /** Script URL running in the SW */
  url: string;
  /** Unique version ID */
  versionId: number;
  /** IPC interface */
  ipc: IpcMainImpl;

  /**
   * Get an existing instance from its unique version ID.
   * Similar to Session.serviceWorkers#getFromVersionID
   */
  static fromVersionID(versionId: number): ServiceWorkerMain | null;
}

interface IpcServiceWorkerEvent {
  senderType: 'service-worker';
  senderWorker: ServiceWorkerMain;
}

ipcMain.on('ping', (event) => {
  if (event.senderType === 'service-worker') {
    event.senderWorker.ipc.send('pong');
  }
});
```

ServiceWorkerMain instances would be created when a remote associated interface is binded for a service worker in the renderer. This can be handled using `ContentBrowserClient::RegisterAssociatedInterfaceBindersForServiceWorker`. When its associated renderer process terminates, it will be destroyed.

- new `v8::Context`
- use of `ShadowRealmGlobalScope`
  - reference to drawbacks section on using proposed ShadowRealm API
- add new `lib/` init scripts
- extending contextBridge to allow render frame and SW usage
  - should "main world" still be used?
- extending IpcMainImpl to listen for SW broker interface
- refactor: ipc messages to Session instead of WebContents


## Drawbacks

### Increased IPC complexity

Electron users will now need to be aware of multiple types of senders. The approach we decide ipcMain's TypeScript types will determine whether it requires breaking changes.

```diff
interface IpcMainEvent {
  // Breaks existing code
-  sender: WebContents;
+  sender: WebContents | ServiceWorkerMain;

  // Introduces properties while keeping the old intact
+  senderType: 'frame' | 'service-worker';
+  senderWorker: ServiceWorkerMain;
}
```

Internal changes will also need to abandon assuming a render frame is always the sender. `ElectronApiIPCHandlerImpl` assumes render frames are used which will require changes.

### Blink internals usage

Some parts of Electron's code are already aware of Blink's ExecutionContexts.

```cpp
// shell/renderer/electron_renderer_client.cc
auto* ec = blink::ExecutionContext::From(context);
if (ec->IsServiceWorkerGlobalScope() || ec->IsSharedWorkerGlobalScope() ||
    ec->IsMainThreadWorkletGlobalScope())
  return;
```

By introducing Preload Realms, we may need to add additional checks for `IsShadowRealmGlobalScope()` or `IsPreloadRealmGlobalScope()`-the latter depending on how much we patch Blink.

Additionally, the setup code ShadowRealm's V8 Context relies on Blink's Oilpan GC ([see implementation](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/bindings/core/v8/shadow_realm_context.cc;l=63;drc=f092aac6779095fcb906f43fdcd223cd7148483f)). Whether we can switch to using V8/Gin's GC instead is yet to be seen.

### Lack of DOM APIs

The ShadowRealm proposal is currently stalled due to lack of consensus on which DOM APIs should be made available in the ShadowRealmGlobalScope (https://github.com/tc39/proposal-shadowrealm/issues/284). This might be a new concept for consumers of preload realms. Common APIs we typically think of as JavaScript-native are missing such as `setTimeout` and `setInterval`.

This could be addressed if necessary. Introducing our own `PreloadRealmGlobalScope` and exposing APIs via WebIDL's [`[Exposed]`](https://chromium.googlesource.com/chromium/src/+/master/third_party/blink/renderer/bindings/IDLExtendedAttributes.md#Exposed) attribute is possible via patching.

It's worth noting that there's already an effort to expose some DOM APIs to all contexts via [`[Exposed=*]`](https://github.com/whatwg/webidl/pull/526).

## Rationale and alternatives

- Provides a new `v8::Context` which can be adapted to new contexts
- Could adopt more `ShadowRealm` features when standardized
  - Reduced maintenence burdon
- Impact of not doing this
  - Reduced abilities for Electron projects to innovate
  - Less competition to competing frameworks


### Rationale
- Provides security and isolation over previous designs.
- Aligns with developing JS standards, namely ShadowRealms.
- Maintenance debt and risk of using Blink internals.

### Alternatives

#### `RendererPreloadWorklet`

[Worklets](https://developer.mozilla.org/en-US/docs/Web/API/Worklet) are a relatively new component of the web. Now used for single-purpose processes such as painting, layout, and audio computation.

Electron could provide its own worklet for deciding when and what to inject as a preload script.

```js
// Allow multiple registrations for extensibility.
registerPreload(class {
  /**
   * Target one or more contexts.
   * Possible values could be:
   *    main-frame, sub-frame, web-worker, service-worker
   */
  static get targetContexts() { return ['service-worker']; }

  /**
   * Filter based on URL of the context. 
   */
  static get urlPatterns() { return ['chrome-extension://*']; }

  /**
   * One or more relative path scripts to be injected.
   */
  static get scripts() {
    return [
      'setup-script.js',
      'relative-script.js'
    ];
  }

  // OR could allow more flexibility and provide APIs similar to ShadowRealm
  // proposal.
  handleContext(context) {
    context.importScript('setup-script.js');
    context.importScript('relative-script.js');
  }
});
```

A approach like above would allow us to eliminate all `nodeIntegrationIn*` APIs
which have been a source of confusion for some users (see [issues with nodeIntegrationInSubFrames](https://github.com/electron/electron/issues/22582)).

## Prior art

### Preloads for Web Workers

Previously existed in Electron, however, never was introduced with context isolation support and thus never landed.

[A PR was created](https://github.com/electron/electron/pull/28923) without support for context isolation which ultimately led to its rejection.

### ShadowRealms

See above in the reference-level guide.

## Unresolved questions

- Design of IPC handlers
- Migration paths
- Preloads in web workers, shared workers
- Design of ServiceWorkerMain API
- Should we use ShadowRealmGlobalScope or introduce our own
  PreloadRealmGlobalScope?
- Maybe we stick with "preload scripts" instead of "preload realms"
- Service Worker wake/sleep
- Only supported in sandboxed browsers?

## Future possibilities

- Preload realms for web workers and shared workers
  - After an initial implementation of support for service workers,
    this should be better understood.