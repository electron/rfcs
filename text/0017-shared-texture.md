- Start Date: 2025-05-16
- RFC PR: [electron/rfcs#17](https://github.com/electron/rfcs/pull/17)
- Electron Issues: [electron/electron#46779](https://github.com/electron/electron/issues/46779)
- Reference Implementation: [electron/electron#46811](https://github.com/electron/electron/pull/46811)
- Status: **Proposed**

# Import Shared Texture

## Summary

This feature introduces a way to import a native shared texture handle into Electron, specifically as a [`VideoFrame`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame). `VideoFrame` natively supports several web rendering systems, including `WebGPU` and `WebGL`. This enables developers to integrate arbitrary native-rendered content with their web applications.

https://github.com/user-attachments/assets/b09bde3a-d2c5-46c3-b3ef-52dda994eb28

## Motivation

While `Offscreen Rendering` allows developers to export a shared texture for embedding the browser's output into their own rendering programs efficiently, there are cases where developers may want the reverse: to import images into the browser. For example, broadcasting apps may use this feature to embed various input sources with modern frontend technologies, providing a contemporary user experience. By using shared textures, we can maximize performance and provide a standardized way to bridge images between Electron and native rendering programs.

## API Design

### sharedTexture

A new `sharedTexture` module will be added, available in both the sandboxed renderer process and the main process. This module provides the primary interface for importing external shared texture handles. It includes the following methods:

#### `sharedTexture.importSharedTexture(options)` _Experimental_

This method takes an object containing the necessary information about the shared texture to import and returns a holder object for the imported shared texture.

* `options` Object - Information about the shared texture to import.
  * `pixelFormat` string - The pixel format of the texture. Can be `rgba` or `bgra`.
  * `colorSpace` [ColorSpace](https://github.com/electron/electron/tree/main/docs/api/structures/color-space.md) (optional) - The color space of the texture.
  * `codedSize` [Size](https://github.com/electron/electron/tree/main/docs/api/structures/size.md) - The full dimensions of the shared texture.
  * `visibleRect` [Rectangle](https://github.com/electron/electron/tree/main/docs/api/structures/rectangle.md) (optional) - A subsection of [0, 0, codedSize.width, codedSize.height]. In most cases, this is the full area.
  * `timestamp` number (optional) - A timestamp in microseconds that will be reflected in the `VideoFrame`.
  * `handle` [SharedTextureHandle](#sharedtexturehandle-object) - The shared texture data.

Returns [`SharedTextureImported`](#sharedtextureimported-object) - The imported shared texture.

#### `sharedTexture.finishTransferSharedTexture(transfer)` _Experimental_

This method takes a transfer object that may have been passed from another process and returns a holder object for the source shared texture, increasing a cross-process reference count to the underlying resource.

* `transfer` [SharedTextureTransfer](#sharedtexturetransfer-object) - The transfer object of the shared texture.

Returns [`SharedTextureImported`](#sharedtextureimported-object) - The imported shared texture from the transfer object.

### SharedTextureHandle Object

This object contains platform-specific handles to the shared texture.

* `ntHandle` Buffer (optional) _Windows_ - NTHANDLE holding the shared texture. Note that this NTHANDLE is local to the current process.
* `ioSurface` Buffer (optional) _macOS_ - IOSurfaceRef holding the shared texture. Note that this IOSurface is suggested to be local to the current process (not global).
* `nativePixmap` Object (optional) _Linux_ - Structure containing planes of the shared texture.
  * `planes` Object[] _Linux_ - Each plane's information for the shared texture.
    * `stride` number - The stride and offset in bytes to be used when accessing the buffers via memory mapping. One per plane per entry.
    * `offset` number - The stride and offset in bytes to be used when accessing the buffers via memory mapping. One per plane per entry.
    * `size` number - Size in bytes of the plane. Necessary for mapping the buffers.
    * `fd` number - File descriptor for the underlying memory object (usually dmabuf).
  * `modifier` string _Linux_ - The modifier is retrieved from the GBM library and passed to the EGL driver.
  * `supportsZeroCopyWebGpuImport` boolean _Linux_ - Indicates whether zero-copy import to WebGPU is supported.

### SharedTextureImported Object

This object is a holder for the imported shared texture, providing methods to interact with the underlying resource.

* `getVideoFrame` Function\<[VideoFrame](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame)\> - Creates a `VideoFrame` that uses the imported shared texture in the current process. You can call `VideoFrame.close()` once you're finished using it; the underlying resources will wait for the GPU to finish internally.
* `startTransferSharedTexture` Function\<[SharedTextureTransfer](#sharedtexturetransfer-object)\> - Creates a `SharedTextureTransfer` that can be serialized and transferred to other processes.
* `release` Function - Releases the resources. If you have transferred and obtained multiple `SharedTextureImported` objects, you must call `release` on each of them. The resource in the GPU process will be destroyed when the last one is released.
  * `callback` Function (optional) - Callback when the GPU command buffer has finished using this shared texture. This provides a precise event to safely release dependent resources. For example, if this object was created by `finishTransferSharedTexture`, you can use this callback to safely release the original one that called `startTransferSharedTexture` in other processes. You can also safely release the source shared texture that was used in `importSharedTexture`.


### SharedTextureTransfer Object

This is a serializable object that can be passed between processes. It's designed to transfer the imported shared texture to another process. The transfer does not mean handing over ownership to the receiving peer. The user should not modify any value in this object.

* `transfer` string - The opaque transfer data of the shared texture, which can be transferred across Electron processes.
* `pixelFormat` string - The pixel format of the texture. Can be `rgba` or `bgra`.
* `codedSize` [Size](size.md) - The full dimensions of the shared texture.
* `visibleRect` [Rectangle](rectangle.md) - A subsection of [0, 0, codedSize.width(), codedSize.height()]. In most cases, this is the full area.
* `timestamp` number - A timestamp in microseconds that will be reflected in the `VideoFrame`.


## Guide-level explanation

### Example Usage Walkthrough

In this section, we will use the output of offscreen rendering as an example of an external shared texture, importing and rendering it in another window.

1. Import the `sharedTexture` module.

```js
const { sharedTexture } = require('electron');
```

2. Create a `BrowserWindow` and enable the offscreen rendering feature, setting the target window size.

```js
const osr = new BrowserWindow({
  show: debugSpec,
  webPreferences: {
    offscreen: {
      useSharedTexture: true
    }
  }
});

// It is recommended to use setSize, as setting width and height above in BrowserWindow may be constrained by the screen size.
osr.setSize(3840, 2160)
```

3. Listen for the `paint` event, import the shared texture, and get a transfer object to pass to the renderer process.

The `handle` is a [SharedTextureHandle](#sharedtexturehandle-object), which contains platform-specific information about the native handle.

```js
osr.webContents.on('paint', (event) => {
  const texture = event.texture;

  if (!texture) {
    return;
  }

  // Import the external shared texture
  const imported = sharedTexture.importSharedTexture({
    pixelFormat: texture.textureInfo.pixelFormat,
    colorSpace: texture.textureInfo.colorSpace,
    codedSize: texture.textureInfo.codedSize,
    visibleRect: texture.textureInfo.visibleRect,
    timestamp: texture.textureInfo.timestamp,
    handle: texture.textureInfo.handle,
  });

  // Prepare for transfer to another process (renderer)
  const transfer = imported.startTransferSharedTexture();

  const id = randomUUID();
  capturedTextures.set(id, { imported, texture });

  // Send the shared texture to the renderer process (see preload.js)
  win.webContents.send('shared-texture', id, transfer);
});
```

4. In `preload.js`, expose an API to the main world and receive the transfer object.

```js
const { sharedTexture } = require('electron');
const { ipcRenderer, contextBridge } = require('electron/renderer');

contextBridge.exposeInMainWorld('textures', {
  onSharedTexture: (cb) => {
    ipcRenderer.on('shared-texture', async (e, id, transfer) => {
      // Get the shared texture from the transfer
      const imported = sharedTexture.finishTransferSharedTexture(transfer);

      // Let the renderer render using WebGPU
      await cb(id, imported);

      // Release the shared texture with a callback
      imported.release(() => {
        // When the GPU command buffer is done, we can notify the main process to release
        ipcRenderer.send('shared-texture-done', id);
      });
    });
  }
});
```

5. In `renderer.js`, prepare a canvas and a WebGPU rendering pipeline.

```js
// Import WebGPU utilities
const canvas = document.createElement('canvas');
canvas.width = 128;
canvas.height = 128;
canvas.style.width = '128px';
canvas.style.height = '128px';

document.body.appendChild(canvas);
const context = canvas.getContext('webgpu');

const initWebGpu = async () => {
  // Configure WebGPU context
  const adapter = await navigator.gpu.requestAdapter();
  const device = await adapter.requestDevice();
  const format = navigator.gpu.getPreferredCanvasFormat();
  context.configure({ device, format });

  // Create a function to render VideoFrame
  window.renderFrame = async (frame) => {
    try {
      // Create external texture
      const externalTexture = device.importExternalTexture({ source: frame });

      // Create bind group layout, specifying the external texture type
      const bindGroupLayout = device.createBindGroupLayout({
        entries: [
          {
            binding: 0,
            visibility: window.GPUShaderStage.FRAGMENT,
            externalTexture: {}
          },
          {
            binding: 1,
            visibility: window.GPUShaderStage.FRAGMENT,
            sampler: {}
          }
        ]
      });

      // Create pipeline layout
      const pipelineLayout = device.createPipelineLayout({
        bindGroupLayouts: [bindGroupLayout]
      });

      // Create render pipeline
      const pipeline = device.createRenderPipeline({
        layout: pipelineLayout,
        vertex: {
          module: device.createShaderModule({
            code: `
                @vertex
                fn main(@builtin(vertex_index) VertexIndex : u32) -> @builtin(position) vec4<f32> {
                var pos = array<vec2<f32>, 6>(
                    vec2<f32>(-1.0, -1.0),
                    vec2<f32>(1.0, -1.0),
                    vec2<f32>(-1.0, 1.0),
                    vec2<f32>(-1.0, 1.0),
                    vec2<f32>(1.0, -1.0),
                    vec2<f32>(1.0, 1.0)
                );
                return vec4<f32>(pos[VertexIndex], 0.0, 1.0);
                }
            `
          }),
          entryPoint: 'main'
        },
        fragment: {
          module: device.createShaderModule({
            code: `
                @group(0) @binding(0) var extTex: texture_external;
                @group(0) @binding(1) var mySampler: sampler;
                @fragment
                fn main(@builtin(position) fragCoord: vec4<f32>) -> @location(0) vec4<f32> {
                let texCoord = fragCoord.xy / vec2<f32>(${canvas.width}.0, ${canvas.height}.0);
                return textureSampleBaseClampToEdge(extTex, mySampler, texCoord);
                }
            `
          }),
          entryPoint: 'main',
          targets: [{ format }]
        },
        primitive: { topology: 'triangle-list' }
      });

      // Create bind group
      const bindGroup = device.createBindGroup({
        layout: bindGroupLayout,
        entries: [
          {
            binding: 0,
            resource: externalTexture
          },
          {
            binding: 1,
            resource: device.createSampler()
          }
        ]
      });

      // Create command encoder and render pass
      const commandEncoder = device.createCommandEncoder();
      const textureView = context.getCurrentTexture().createView();
      const renderPass = commandEncoder.beginRenderPass({
        colorAttachments: [
          {
            view: textureView,
            clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
            loadOp: 'clear',
            storeOp: 'store'
          }
        ]
      });

      // Set pipeline and bind group
      renderPass.setPipeline(pipeline);
      renderPass.setBindGroup(0, bindGroup);
      renderPass.draw(6); // Draw a rectangle composed of two triangles
      renderPass.end();

      // Submit commands
      device.queue.submit([commandEncoder.finish()]);
    } catch (error) {
      console.error('Rendering error:', error);
    }
  };
};

initWebGpu().catch((err) => {
  console.error('Failed to initialize WebGPU:', err);
});

```

6. Obtain a `VideoFrame` from the imported shared texture and render it. After submitting the WebGPU command buffer, you can call `close` on the `VideoFrame`.

```js
window.textures.onSharedTexture(async (id, imported) => {
  try {
    // Get VideoFrame from the imported texture
    const frame = imported.getVideoFrame();

    // Render using WebGPU
    await window.renderFrame(frame);

    // Release the VideoFrame as it is no longer needed
    frame.close();
  } catch (error) {
    console.error('Error getting VideoFrame:', error);
  }
});
```

7. After the frame is processed, in `preload.js` notify the main process to release resources.

```js
ipcMain.on('shared-texture-done', (event: any, id: string) => {
  // Release the shared texture resources in the main process
  const data = capturedTextures.get(id);
  if (data) {
    capturedTextures.delete(id);
    const { imported, texture } = data;

    // Release the imported shared texture
    imported.release(() => {
      // Release the shared texture once the GPU is done
      texture.release();
    });
  }
});
```

### Cautions

The lifecycle of objects must be carefully managed. Let's use the above use case as an example:

1. First, obtain an external shared texture, in this example from offscreen rendering, referred to as `originTexture`.
2. Import `originTexture` to get `importedTextureMain`.
3. Obtain a `transferObject` from `importedTextureMain` and send it through the IPC channel.
4. **Caution**: At this point, you cannot release `importedTextureMain` as the import in the other process is not yet complete.
5. Receive the `transferObject` in the renderer process and obtain `importedTextureRenderer`.
6. **Caution**: At this point, you also cannot release `importedTextureMain`, as GPU tasks are asynchronous. All `importedTexture` objects are backed by a `Mailbox`, which has a cross-process reference counter in the GPU process. The counter will not increase until the command buffer actually runs the `importedTextureRenderer` import task. Thus, if you release `importedTextureMain` now, the counter may reset to zero, causing the GPU to destroy the mailbox and making the future import task fail.
7. Obtain a `VideoFrame` from `importedTextureRenderer`, use it in WebGPU, and create an `ExternalTexture`.
8. After submitting the command buffer, all GPU tasks are submitted. You can safely call `close` on the `VideoFrame`, as the command is guaranteed to execute after all submitted tasks, ensured by a `SyncToken` (explained in detail later).
9. For similar reasons, you can then call `release` on `importedTextureRenderer`, as all tasks in the renderer process are complete. Pass a callback to the `release` function, which uses a `SyncToken` internally to ensure the callback is triggered after all submitted tasks are executed by the GPU, making it safe to release further resources.
10. Now, notify the main process to release `importedTextureMain`. It is recommended to also use a callback to safely release `originTexture`.

![Lifecycle](../images/0017/lifecycle.excalidraw.png)

## Reference-level explanation

### Design Considerations

The main goal of the current implementation is to import external shared texture descriptions while reusing as much of the existing Chromium infrastructure as possible to minimize maintenance burden. `SharedImage` is chosen as the underlying holder of the external texture because it provides `SharedImageInterface::CreateSharedImage`, which accepts a `GpuMemoryBufferHandle` containing native shared handle types: `NTHANDLE` for Windows, `IOSurfaceRef` for macOS, and `NativePixmapHandle` with file descriptors for each plane for Linux. However, during research, several critical problems led to the final design described below.

#### Shared handle is local to process

On Windows, a shared D3D11 texture can be created by `GetSharedHandle` (deprecated) or `CreateSharedHandle`. The deprecated method generates a `non-NTHANDLE` that can be globally accessed, while the newer one generates an `NTHANDLE` that is local to the current process. To share it with other processes, you need to call `DuplicateHandle` to create this handle in the remote process.

On macOS, `IOSurface` can also be global by setting `kIOSurfaceIsGlobal`, which is now deprecated. To share an `IOSurface` with other processes, you need to create a `mach_port` from it and pass the `mach_port` through a previously created IPC (which also uses `mach_port` as transport), making it significantly more complex than on Windows.

Given these obstacles, you must ensure that when calling `sharedTexture.importSharedTexture`, the handle is already available to the current process. You might be able to use a global `IOSurface`, but a `non-NTHANDLE` is not an option as described in problem 2 below.

In fact, Chromium's IPC internally handles all these concerns (duplicating handles for remote processes, passing `mach_port` through `mach_port`), which is why the OSR `paint` event can use the handle directly in the main process, even though the original handle is generated in the GPU process—because IPC has transparently handled these issues.

#### Shared handle ownership management

Chromium takes ownership of the `GpuMemoryBuffer` and its native representation. Once the resource is destroyed, the handle will be closed. Thus, the handle the user imports must allow Chromium to take ownership.

On Windows, calling `CloseHandle` on a `non-NTHANDLE` is invalid and will cause Chromium to crash. Therefore, you cannot use a `non-NTHANDLE` when importing. In the future, we may provide a helper for this. To work with this design, when calling `importSharedTexture`, an `NTHANDLE` will be duplicated internally for the user, meaning the user still has ownership of the handle.

On macOS, `IOSurface` is a reference-counted resource. When calling `importSharedTexture`, instead of taking ownership, Chromium can simply retain this resource and increment the reference count, so the user still retains ownership.

#### Transferring the shared texture between processes

Initially, I considered using the WebGPU Dawn Native API to import the external texture as a `WGPUSharedTextureMemory`, but encountered several problems. For example, it was unable to export, difficult to manage the lifetime of a frame, and working with WebGPU was non-ideal.

`SharedImage` has advantages for sharing across processes because it holds a reference to a `Mailbox`, which points to the corresponding `SharedImageBacking` in the GPU process. Therefore, I use `SharedImageInterface::ImportSharedImage` and `ClientSharedImage::Export` to serialize sufficient information to retrieve the `SharedImage` reference in another process, and it can also reuse the mojo serializer to serialize as a string. Everything is handled by Chromium.

Lifecycle management is a bit tricky. When you get a `SharedImage` holding a `Mailbox` from either `ImportSharedImage` or `CreateSharedImage`, the `Mailbox` is just a placeholder; it doesn't ensure the GPU has actually imported the texture and increased the reference count to the `Mailbox`. Chromium doesn't wait for the GPU to finish, typically using a `SyncToken` to ensure GPU task dependencies. For example, releasing a `SharedImage` is also an asynchronous operation, and calling release often requires a `SyncToken` that acts as a barrier to ensure all GPU importing and rendering tasks are finished. In our case, since we can't use a mojo callback to update the `SyncToken` across processes, we use callbacks to notify dependent resources to release.

#### Interprocess resource management

The final challenge is managing the lifetime of a frame. Currently, I use OSR to get an exported shared texture generated by Chromium itself. As the documentation states, the texture needs to be manually released. By importing this texture into a `SharedTextureImported` in Electron, we must ensure the imported one is released before the source texture is released.

What's more challenging is that the `paint` event occurs in the main process, while rendering must occur in renderer processes, so we must use `startTransferSharedTexture` to pass the underlying `SharedImage` to another process.

Most GPU calls are asynchronous, sent to the GPU process's command buffer through the `GpuChannel` of each client process. When we have two `SharedImage` instances referencing the same `Mailbox` in two different processes, we don't know when the GPU has finished using the resources. Typically, this is guaranteed by a `SyncToken`. For example, we can schedule the destruction of a `SharedImage` with an empty `SyncToken` in the main process (where the texture was first imported), but when we use the same `SharedImage` in the renderer process and use it in WebGPU (or WebGL) pipelines, the destruction token will be generated by WebGPU to prevent destruction before the GPU uses it. The destruction won't occur until WebGPU rendering finishes.

Ideally, if we were in Chromium, we could use mojo to create a `PendingRemote` callback and update the main process destruction `SyncToken` to prevent the main process from releasing the frame before the GPU starts working on it. In the future, we may implement this in Electron code to wrap the mojo functionality. Eventually, I found a way to use `gpu::ContextSupport` and register a callback when a specific `SyncToken` is released (signaled). When you call `release()` on the imported shared texture object, if you've used `VideoFrame` and imported it into a WebGPU pipeline, it will wait for WebGPU to finish rendering, then run a callback to notify you to release dependent resources, such as the original imported object in the main process, the source texture, etc.

### New Dependencies

This feature mainly depends on the `SharedImage` API in Chromium.

## Drawbacks

Chromium's `SharedImage` may have minor API changes in the future. For example, `VideoFrame` might eventually manage the lifetime of the underlying `SharedImage` automatically (which would be beneficial), but this could introduce minor maintenance burdens.

## Rationale and alternatives

### Rationale

This design is nearly optimal, as it requires no upstream patches and already takes into account all Chromium internal design constraints.

### Alternatives

Besides using `VideoFrame`, one possible alternative is to use `WebGPU` directly, which would require significant patches to Dawn and Chromium, and was therefore abandoned.

Regarding lifecycle management, I considered passing callbacks like `MessagePort` does. After some research, although it is possible to create an IPC channel using the Blink `MessagePortDescriptor`, I found this does not simplify things much:

- You still have to ensure the remote process has finished calling `finishTransferSharedTexture` (that's when it transmits a new SyncToken back), before you can drop the reference or call release. It's still an asynchronous procedure.
- You still don't know when to release the original native handle. A callback is still needed to notify a safe release.

Therefore, I decided not to use automated `SyncToken` IPC, and to continue using the release callback to provide a precise and safe timing for manually releasing dependencies.

## Prior art

Previously, users could import external images via video streams or bitmaps, which consumed CPU and resulted in visual loss. By using shared textures, we can import GPU resources directly, which is a much more efficient way to display arbitrary content in web applications.

## Unresolved questions

- How to handle native shared handles: can we provide this feature in sandboxed renderer processes?
- Are there any other web standards besides `VideoFrame` that can be used to render the texture?

## Future possibilities

- Performance: It would be meaningful to profile the overhead of managing the lifecycle through Electron IPC callbacks.
- OSR: Since we can import a shared texture as a `SharedImage`, it may be possible to implement OSR using `SharedImage` instead of `FrameSinkVideoCapturer`.
