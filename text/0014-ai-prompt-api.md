# RFC Template

- Start Date: 2025-04-09
- RFC PR: [electron/rfcs#14](https://github.com/electron/rfcs/pull/14)
- Electron Issues:
- Reference Implementation:
- Status: **Proposed**

# Prompt API support

## Summary

The Prompt API is part of [Built-in AI](https://developer.chrome.com/docs/ai/built-in)
capabilities in Chromium based browsers. With the Prompt API developers can send
natural language requests from the renderer for local processing.  This proposal adds
support for this API in Electron.

## Motivation

Built in AI features are being implemented in Chromium based browsers.  Electron should
also support these features.

## Guide-level explanation

The Prompt API can be used by Electron app developers by defining a local AI handler
script via `Session.registerLocalAIHandlerScript` and a new module `localAIHandler`.
The handler script will run in a [utilityProcess](https://www.electronjs.org/docs/latest/api/utility-process)
and will allow app developers to proxy Built-in API calls to a local LLM implementation of
their choice.

For the initial implementation, the local AI handler will only support the prompt API,
but the long term intention is that this handler would also support other Built-in AI APIs.

### Session.registerLocalAIHandlerScript

`ses.registerLocalAIHandlerScript(options)`
* `options` Object
  * `filePath` string - Path of the script file. Must be an absolute path.

Returns `string` - The ID of the registered local AI handler script.

`ses.unregisterPreloadScript(id)`

* `id` string - local AI handler script ID

Unregisters the specified local AI handler script and terminates the associated
utility process if it is running.

### localAIHandler

> Proxy Built-in AI APIs to a local LLM implementation

Process: [utility](https://www.electronjs.org/docs/latest/api/utility-process)

This module is intended to be used by a script registered to a session via
`ses.registerLocalAIHandlerScript(options)`

#### Methods

The `localAIHandler` module has the following methods:

##### `localAIHandler.setPromptAPIHandler(handler)`
* `handler` Function\<[LanguageModel](https://github.com/webmachinelearning/prompt-api?tab=readme-ov-file#full-api-surface-in-web-idl)\> | null  
  * `webContents` ([WebContents](web-contents.md) | null) - WebContents calling the Prompt API.
  The webContents may be null if the Prompt API is called from a service worker or shared worker.

> Main process

```js
const { session } = require('electron');

app.whenReady().then(() => {
  session.defaultSession.registerLocalAIHandlerScript({
    filePath: 'PATH TO HANDLER SCRIPT'
  })
})
```

> Utility script

```js
const { localAIHandler } = require('electron/utility');

class ElectronAI {
  #webContentsId;
  #languageModel;

  constructor(webContents) {
    this.#webContentsId = webContents.id;
  }

  async createInternal(options) {
    this.#languageModel = //Initialize local LLM.
  }  

  async prompt(input = '', options) {
    if (!this.#languageModel) {
      throw new Error('AI model process not started');
    }
    return #languageModel.prompt(input, options);
  }

  async destroy() {
    if (this.#languageModel) {
      this.#languageModel.destroy();
    }
  }

  async promptStreaming(input = '', options) {
    if (!this.#languageModel) {
      throw new Error('AI model process not started');
    }
    return #languageModel.promptStreaming(input, options);
  }

  static async create(options) {
    const electronAI = new ElectronAI();
    electronAI.createInternal(options);
    return electronAI;
  }
}

localAIHandler.setPromptAPIHandler((webContents) => new ElectronAI(webContents));
```

## Reference-level explanation

1. Create an implementation of [blink::mojom::AILanguageModel](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/public/mojom/ai/ai_language_model.mojom) that can be associated to a utility process. `content/browser/ai/echo_ai_language_model.cc` can be used as a basis for this implementation.  The blink::mojom::AILanguageModel functions will use the specified utility process to proxy their calls to that handler.

2. Create an implementation of [blink::mojom::AIManager](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/public/mojom/ai/ai_manager.mojom) that can be associated to a `electron::api::Session` and
`electron::api::WebContents`. . `content/browser/ai/echo_ai_manager_impl.cc` can be used as a basis for this implementation. `CanCreateLanguageModel` should check if there is a Local AI Handler Script assigned to the associated session. `CreateLanguageModel` should create a new utility process with the specified script, invoke the prompt API handler
in the script and then invoke `create` on the object returned from the prompt API handler.  Additionally, `CreateLanguageModel` should instantiate the implementation of `blink::mojom::AILanguageModel` mentioned above with the current utility process.  The rest of the functions in this class will either be unimplemented for now or alternatively they can use the currently defined echo implementation in `content/browser/ai/`.

3. Implement [BindAIManager](https://source.chromium.org/chromium/chromium/src/+/main:content/public/browser/content_browser_client.h;l=3151) in ElectronBrowserClient.  The BrowserContext passed in can be used to get the
associated `electron::api::Session` and the RenderFrameHost passed in can be used to get the associated
`electron::api::WebContents`.

## Drawbacks

The browser implementations at this time are experimental and subject to change.

## Rationale and alternatives

The approach taken in this RFC is to provide a low level interface to the Prompt API that
can then be wired to a locally running LLM.  Alternatively, Electron could provide a default LLM
like Chromium browsers provide but that is a much larger effort.

Additionally, [node-llama-cpp](https://github.com/withcatai/node-llama-cpp/blob/master/docs/guide/electron.md) provides similar functionality.  For apps that are Electron only this may be a viable solution, but for apps that also provide a browser version of their app there would not be shared functionality available.

## Unresolved questions

- Are there parameters that should be passed to the handler?  [BindAIManager](https://source.chromium.org/chromium/chromium/src/+/main:content/public/browser/content_browser_client.h;l=3151) gets a `BrowserContext` and `base::SupportsUserData`, so I'm not sure what else we could pass along.

## Future possibilities

- Support for other built in AI APIs
- Default implementation of the Prompt API so that Electron app developers do not need to create their own.
