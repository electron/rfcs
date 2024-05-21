

- Start Date: 2024-05-12
- RFC PR: [electron/rfcs#0005](https://github.com/electron/rfcs/pull/0005)
- Addresses Electron Issues:  [issue #1](https://github.com/electron/electron/issues/40182);  [issue #2](https://github.com/electron/electron/issues/25667).
- Status: **Proposed**

# Two-Way IPC Invocation from Main to Renderer Process

## Summary

This proposal aims to introduce a new two-way IPC pattern that allows calling a renderer process from main process and waiting for the result. This will be done by adding `webContents.invoke` paired with `ipcRenderer.handle`. This feature will streamline bidirectional communication between the main and renderer processes, eliminating the need for workarounds and third-party libraries.

## Motivation

The current absence of an `invoke/handle` pattern from the main to renderer processes in Electron limits the efficiency and simplicity of inter-process communication. While `ipcRenderer.invoke()` allows for asynchronous messaging from renderer to main, a symmetric functionality is missing for the opposite direction. 

Introducing this feature will streamline communication and create cleaner, more maintainable codebases by eliminating the need for convoluted workarounds or reliance on external libraries. The popularity of 3rd party libraries like [electron-promise-ipc](https://www.npmjs.com/package/electron-promise-ipc), which has peaked at almost 1500 weekly downloads, underscores the real demand in the developer community. However, these external solutions often lack robust safeguards against edge cases and may become outdated or incompatible with newer Electron versions, potentially exposing developers to vulnerabilities. Developers have also been asking for this feature as documented in this [issue](https://github.com/electron/electron/issues/40182) and this [issue](https://github.com/electron/electron/issues/25667).  

This rfc hopes to implement a more efficient and straightforward communication mechanism between the main and renderer processes, reducing boilerplate code and providing a consistent API across Electron.

## Guide-level explanation 

### `webContents.invoke(options, channel, ...args)`
* `options` Object (optional)
  * `maxTimeoutMs` number
- `channel` string
- `...args` any[]

Returns `Promise<any>` - Resolves with the response from the renderer process.

Send a message to the renderer process via channel and expect a result asynchronously. Arguments will be serialized with the Structured Clone Algorithm, so prototype chains will not be included. Sending Functions, Promises, Symbols, WeakMaps, or WeakSets will throw an exception.

`options` is an optional parameter that specifies the maximum amount of time (in milliseconds)  to wait for a response from the renderer process before rejecting the promise. This is useful in scenarios where the function might take a long time to complete, or where it might not complete at all due renderer process termination.

If `maxTimeoutMs` is not provided, a default value of 5000 milliseconds (5 seconds) is used. This means
that if the function doesn't receive a response within 5 seconds, it will time out and throw an error.

The renderer process should listen for channel with `ipcRenderer.handle()`.

For example:

```
// main process
const { BrowserWindow } = require('electron');

const rendererWindow = new BrowserWindow({ ... });

rendererWindow.webContents.on('did-finish-load', async () => {
  const result = await rendererWindow.webContents.invoke('my-channel', arg1, arg2);
  console.log(result); // Output from the renderer process

  // adding custom timeout
  const result = await rendererWindow.webContents.invoke({maxTimeoutMs: 1000}, 'my-channel', arg1, arg2);
});
```

### `ipcRenderer.handle(channel, ...args)`
- `channel` string
- `listener` Function<Promise<any\> | any\>
- `event` IpcRendererInvokeEvent
- `...args` any[]

Handles a single `invoke`able IPC message, then removes the listener. See `ipcRenderer.handle(channel, listener)`.

For example:

```
// renderer process
const { ipcRenderer } = require('electron');

ipcRenderer.handle('my-channel', async (event, ...args) => {
  const [arg1, arg2] = args;
  console.log(`Received from main process: ${arg1} ${arg2}`);
  return 'Response from renderer process';
});
```

This feature will impact existing use cases by providing a more streamlined approach to communication between the main and renderer processes. Developers can migrate directly to utilize the new APIs. Additionally, the existing IPC messaging methods will remain available for backward compatibility.
The proposed feature will make Electron code more maintainable by providing a consistent API for communication between the main and renderer processes. The promise-based pattern and the ability to handle events directly in the main process will reduce boilerplate code and improve code readability.

## Reference-level explanation

The `webContents.invoke()` method will be implemented as a wrapper around a method on `mainFrame: WebFrameMain` where it will parse/ set default timeout.

```
// lib/browser/api/web-contents.ts

WebContents.prototype.invoke = async function (...args: any[]) {
  // Check if the first argument is an options object
  const isOptionsProvided = typeof args[0] === 'object';
  const { maxTimeoutMs = 5000 } = isOptionsProvided ? args.shift() : {};
  const channel = args.shift();

  return this.mainFrame.invoke(maxTimeoutMs, channel, ...args);
};

```
The javascript wrapper for `WebFrameMain` will handle the timeout and reject the promise if timeout is exceeded.
```
// lib/browser/api/web-frame-main.ts

WebFrameMain.prototype.invoke = async function (maxTimeoutMs, channel, ...args) {
  if (typeof channel !== 'string') {
    throw new TypeError('Missing required channel argument');
  }

  if (typeof maxTimeoutMs !== 'number') {
    throw new TypeError('Invalid timeout argument');
  }

  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      reject(new Error(`Timeout after ${maxTimeoutMs}ms waiting for IPC response on channel '${channel}'`));
    }, maxTimeoutMs);

    this._invoke(channel, args).then(({ error, result }) => {
      clearTimeout(timeoutId);
      if (error) {
        reject(new Error(`Error invoking remote method '${channel}': ${error}`));
      } else {
        resolve(result);
      }
    });
  });
};

```

New invoke method on WebFrameMain that serializes arguments, pass the its own routingId and a callback to resolve the promise.

```
// shell/browser/api/electron_api_web_frame_main.cc

v8::Local<v8::Promise> WebFrameMain::Invoke(v8::Isolate* isolate,
                                            const std::string& channel,
                                            v8::Local<v8::Value> args) {

  if (!CheckRenderFrame()) {
    return v8::Local<v8::Promise>();
  }

  blink::CloneableMessage message;
  if (!gin::ConvertFromV8(isolate, args, &message)) {
    isolate->ThrowException(v8::Exception::Error(
        gin::StringToV8(isolate, "Failed to serialize arguments")));
    return v8::Local<v8::Promise>();
  }

  gin_helper::Promise<blink::CloneableMessage> promise(isolate);
  auto handle = promise.GetHandle();

  GetRendererApi()->Invoke(
      channel, RoutingID(), std::move(message),
      base::BindOnce(
          [](gin_helper::Promise<blink::CloneableMessage> promise,
             blink::CloneableMessage result) {  promise.Resolve(result); },
          std::move(promise)));

  return handle;
}
```

Update `api.mojom` to add new cross process method for Invoke on interface ElectronRenderer
```
// shell/common/api/api.mojom

  // Emits an event on |channel| from the ipcRenderer JavaScript object in the renderer
  // process, and returns the response.
  Invoke(
      string channel,
      int32 routingId,
      blink.mojom.CloneableMessage arguments) => (blink.mojom.CloneableMessage result);
```

Implement new invoke method on ElectronRenderer by creating a new event with a `ReplyChannel` (same interface as the existing invoke function), check that routingIds are the same (this may be redundant check because the ElectronApiServiceImpl is 1:1 with webFrame, but check to be safe )

```
void ElectronApiServiceImpl::Invoke(const std::string& channel,
                                    int routingId,
                                    blink::CloneableMessage arguments,
                                    InvokeCallback callback) {
  blink::WebLocalFrame* frame = render_frame()->GetWebFrame();

  if (!frame) {
    v8::Isolate* isolate = v8::Isolate::GetCurrent();
    gin_helper::ReplyChannel<InvokeCallback>::Create(isolate,
                                                     std::move(callback))
        ->SendError("webFrame does not exist");
    return;
  }

  v8::Isolate* isolate = frame->GetAgentGroupScheduler()->Isolate();
  v8::HandleScope handle_scope(isolate);

  v8::Local<v8::Context> context = renderer_client_->GetContext(frame, isolate);
  v8::Context::Scope context_scope(context);

  if (render_frame()->GetRoutingID() != routingId) {
    gin_helper::ReplyChannel<InvokeCallback>::Create(isolate,
                                                     std::move(callback))
        ->SendError("Routing ID does not match the current webFrame");
    return;
  }

  v8::MicrotasksScope script_scope(isolate, context->GetMicrotaskQueue(),
                                   v8::MicrotasksScope::kRunMicrotasks);
  // Make an event object for the IPC message.
  gin::Handle<gin_helper::internal::Event> event =
      gin_helper::internal::Event::New(isolate);

  gin_helper::Dictionary dict(isolate, event.ToV8().As<v8::Object>());

  // add callback to the event object
  dict.Set("_replyChannel",  gin_helper::ReplyChannel<InvokeCallback>::Create(isolate, std::move(callback)));

  v8::Local<v8::Value> args = gin::ConvertToV8(isolate, arguments);

  EmitInvokeEvent(context, channel, event, args);
}

```

InvokeIpcCallback on `-ipc-invoke` with event with reply channel and serialized args.
```

void EmitInvokeEvent(v8::Local<v8::Context> context,
                     const std::string& channel,
                     gin::Handle<gin_helper::internal::Event> event,
                     v8::Local<v8::Value> args) {
  auto* isolate = context->GetIsolate();

  v8::HandleScope handle_scope(isolate);
  v8::Context::Scope context_scope(context);
  v8::MicrotasksScope script_scope(isolate, context->GetMicrotaskQueue(),
                                   v8::MicrotasksScope::kRunMicrotasks);

  std::vector<v8::Local<v8::Value>> argv = {
      event.ToV8().As<v8::Object>(), gin::ConvertToV8(isolate, channel), args};

  InvokeIpcCallback(context, "-ipc-invoke", argv);
}
```

Return response in common-init where `ElectronApiServiceImpl` will look for the "ipcNative" hidden object when
invoking the '-ipc-invoke' callback. Only handlers currently registered with ipcRenderer will reply with resolved value. Otherwise, an error will be returned for unregistered handlers.

```
// common-init.ts

v8Util.setHiddenValue(global, 'ipcNative', {
  onMessage (internal: boolean, channel: string, ports: MessagePort[], args: any[]) {
    const sender = internal ? ipcRendererInternal : ipcRenderer;
    sender.emit(channel, { sender, ports }, ...args);
  },

  '-ipc-invoke': async function (event: Electron.IpcRendererInvokeEvent, channel: string, args: any[]) {
    const replyWithResult = (result: any) => event._replyChannel.sendReply({ result });
    const replyWithError = (error: Error) => {
      console.error(`Error occurred in handler for '${channel}':`, error);
      event._replyChannel.sendReply({ error: error.toString() });
    };
    const handler = (ipcRenderer as any)._invokeHandlers.get(channel);

    if (!handler) {
      replyWithError(new Error(`No handler registered for '${channel}'`));
      return;
    }

    try {
      replyWithResult(await Promise.resolve(handler(event, ...args)));
    } catch (err) {
      replyWithError(err as Error);
    }
  }
});
```

In the renderer process, the `ipcRenderer.handle()` method will be responsible for responding to the `invoke-method` message sent by the main process. This allows a unique set of handlers for each webFrame. If frame navigates, the `_invokeHandlers` will reset.

```
class IpcRenderer extends EventEmitter implements Electron.IpcRenderer {
  private _invokeHandlers: Map<string, (e: IpcRendererEvent, ...args: any[]) => void> = new Map();
  ....

  handle: Electron.IpcRenderer['handle'] = (method, fn) => {
    if (this._invokeHandlers.has(method)) {
      throw new Error(`Attempted to register a second handler for '${method}'`);
    }
    if (typeof fn !== 'function') {
      throw new TypeError(`Expected handler to be a function, but found type '${typeof fn}'`);
    }

    this._invokeHandlers.set(method, fn);
  };

  handleOnce: Electron.IpcRenderer['handleOnce'] = (method, fn) => {
    this.handle(method, (e, ...args) => {
      this.removeHandler(method);
      return fn(e, ...args);
    });
  };

  removeHandler (method: string) {
    this._invokeHandlers.delete(method);
  }
}
```
### Edge Cases Addressed
The implementation handles the following edge cases:

1. Renderer Process Termination: If the renderer process containing the target frame is terminated before responding to the IPC message, the promise should be rejected.
     > Current Solution: The timeout option on webContents.invoke will make sure the user gets a response rather than hanging.

2. Frame Navigation: If the target frame navigates before responding to the IPC message, the promise should be rejected.

   a. cross-site navigation: if a render frame navigates to a different site (not just a different page), it will be swapped out and a new render frame host will be created (process isolation). 

      > Current Solution: The implementation checks that the routing id of the mainFrame (RenderFrameHost) is the same as the routing id of webFrame to make sure main process is not invoking a response from a different webFrame than intended. (Maybe this is a redundant check because the ElectronRenderer/ RendererAPI always corresponds to the calling WebFrameMain/RenderFrameHost so the routingIds will never be different)

   b. same-site navigation: Navigating the same sime, the same render frame host will be used.
      > Current Solution: Renderer will respond with a resolved value from registered handler. Ideally we want to reject the response if it is not the same page we intended to invoke from the main process, but there is no way to control that same site navigation re-uses the same frame with the same context. 

3. No Registered Handler: If the renderer process does not have a registered handler for the specified channel, the promise should be rejected.

   > Current Solution: When page changes, whether same or cross site navigation, the renderer process updates and new handlers are registered specific to that renderer process. So if a call from main is made before a handler registers or to a handler that does not exist, the promise will be rejected.

### Edge Cases Not Addressed

4. Multiple Frames: The `webContents.invoke()` method supports invoking from the mainFrame WebFrameMain. Future area of development is being able to invoke from a specific RenderFrameHost/ WebFrameMain instance by providing the routingId/ frameId option. This allows for communication with individual frames within a single webContents instance.


## Drawbacks

- Introducing new APIs may increase the complexity of the Electron IPC and documentation. This can potentially introduce new bugs or edge cases for the team to support.
- Implementing timeout functionality and handling edge cases may introduce additional overhead and potential performance implications.
- The timeout mechanism may not be suitable for all use cases, as some operations in the renderer process may take longer than the configured timeout period.

## Rationale and alternatives

The proposed design follows the existing pattern established by `ipcRenderer.invoke()`. Alternative designs, such as relying solely on third-party libraries or manual workarounds to send multiple messages were considered but deemed less desirable due to potential compatibility issues, dependency concerns, and inconsistent API patterns.

Not implementing this feature could lead to continued reliance on workaround solutions, increased code complexity, and fragmentation within the Electron ecosystem as developers adopt different approaches to address the lack of bidirectional IPC.

## Prior art

Similar bidirectional IPC patterns exist in other frameworks and libraries, such as Node.js's worker_threads and various IPC modules in the Node.js ecosystem. The introduction of `webContents.invoke()` and `ipcRenderer.handle()` brings Electron closer to parity with these patterns and simplifies cross-process communication for Electron developers.

## Unresolved questions

- I have a working implementation in a draft PR. The exact implementation details regarding timeout management and handling of edge cases need further discussion and refinement.
- Consideration should be given to documentation so developers are well educated and guard for potential failure cases. 

## Future possibilities

- Support multiple frames where webContents can take a routingId to invoke from different WebFrameMain instances
- Add internal version for `ipcRendererInternal`
- Be able to add a more robust implementation by removing the timeout and detect in the C++ code when the frame has navigated or destroyed and be able to reject the promise. 

  - i.e when webContents gets destroyed in the middle of an invoke, I've tested that `AbortCallback` gets called because an attempt is made to execute avaScript in a context that should not be executing JavaScript. i.e context that has been destroyed. But have not figured out a way to pass the error back to the javascript/ main world


  ``` 
  // src/electron/shell/renderer/electron_api_service_impl.cc

  void AbortCallback(v8::Isolate* isolate, v8::Local<v8::Context> context) {
   std::cout << "Attempted to execute JavaScript in an invalid context" << std::endl;
  }


  context->SetAbortScriptExecution(AbortCallback);
  ```