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

The Prompt API can be used by Electron app developers by defining a Prompt API handler
via `Session.setPromptAPIHandler`.  This handler will allow app developers to proxy
Prompt API calls to a local LLM implementation of their choice.

`ses.setPromptAPIHandler(handler)`
* `handler` Function\<[LanguageModel](https://github.com/webmachinelearning/prompt-api?tab=readme-ov-file#full-api-surface-in-web-idl)\> | null
  
This handler will be called when web content calls the [Prompt API](https://github.com/webmachinelearning/prompt-api).

The handler should return a [Language Model Object](https://github.com/webmachinelearning/prompt-api?tab=readme-ov-file#full-api-surface-in-web-idl).

Given the demands of running LLM locally, a utility process should be used to offload processing.  For example:

```js
const { session, utilityProcess } = require('electron/main');

class ElectronAI {
  #aiProcess; //utilityProcess to handle LLM processing.  

  async createInternal(options) {
    this.#aiProcess = utilityProcess.fork(utilityScriptPath, [], {
        stdio: ['ignore', 'pipe', 'pipe', 'pipe'],
    });
  }

  async prompt(input = '', options) {
    if (!this.#aiProcess) {
      throw new Error('AI model process not started');
    }
    this.#aiProcess.postMessage({
      type: UTILITY_MESSAGE_TYPES.SEND_PROMPT,
      data: { input, stream: false, options },
    });    
  }

  async destroy() {
    if (this.#aiProcess) {
      this.#aiProcess.postMessage({ type: UTILITY_MESSAGE_TYPES.STOP });    
    }
    this.#aiProcess = null;
  }

  async promptStreaming(input = '', options) {
    //Use aiProcess to handle prompt streaming
  }

  static async create(options) {
    const electronAI = new ElectronAI();
    electronAI.createInternal(options);
    return electronAI;
  }
}

session.defaultSession.setPromptAPIHandler(() => { ElectronAI });
```

## Reference-level explanation

1. Create an implementation of [blink::mojom::AILanguageModel](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/public/mojom/ai/ai_language_model.mojom) that can be associated to a `electron::api::Session`. `content/browser/ai/echo_ai_language_model.cc` can be used as a basis for this implementation.  If a handler is defined for the associated session then the blink::mojom::AILanguageModel functions should proxy their calls to that handler.

2. Create an implmentation of [blink::mojom::AIManager](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/public/mojom/ai/ai_manager.mojom) that can be associated to a `electron::api::Session`. `content/browser/ai/echo_ai_manager_impl.cc` can be used as a basis for this implementation. `CanCreateLanguageModel` should check if there is an prompt API handler assigned to the associated session. `CreateLanguageModel` should invoke `create` on the object returned from the prompt API handler.  Additionally, `CreateLanguageModel` should instantiate the implementation of `blink::mojom::AILanguageModel` mentioned above with the current `electron::api::Session`.  The rest of the functions in this class will either be unimplemented for now or alternatively they can use the currently defined echo implementation in `content/browser/ai/`.

3. Implement [BindAIManager](https://source.chromium.org/chromium/chromium/src/+/main:content/public/browser/content_browser_client.h;l=3151) in ElectronBrowserClient.  The BrowserContext passed in can be used to get the
associated `electron::api::Session`.

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

- Support for other built in AI APIS
- Default implementation of the Prompt API so that Electron app developers do not need to create their own.
