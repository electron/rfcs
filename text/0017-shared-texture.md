- Start Date: 2025-05-16
- RFC PR: [electron/rfcs#17](https://github.com/electron/rfcs/pull/17)
- Electron Issues: [electron/electron#46779](https://github.com/electron/electron/issue/46779)
- Reference Implementation: [electron/electron#46811](https://github.com/electron/electron/pull/46811)
- Status: **Proposed**

# Import Shared Texture

## Summary

This feature provides a way to import a native shared texture handle into Electron, specifically in the form of [`VideoFrame`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame), which by nature supports several Web rendering systems including `WebGPU`, `WebGL`. This enables developers to integrate arbitrary native rendered content with their web applications.

https://github.com/user-attachments/assets/b09bde3a-d2c5-46c3-b3ef-52dda994eb28

## Motivation

While `Offscreen Rendering` provides a way to export a shared texture to developers who wants to embed the browser's picture into their own rendering programs in a nice fast way, there're certain cases the developer may want a reverse order to import pictures into browser. For example, broadcasting apps may use this feature to embed all sorts of input sources with modern frontend technologies, providing a modern experience to the end users. By using shared texture, we can push the performance to the limit, and provides a standard way to bridge pictures between Electron and native rendering programs.

## API Design

### sharedTexture 

A new `sharedTexture` module will be added, exposed both in sandboxed renderer process and main process, providing the primary interface to import external shared texture handle. It has the following methods: 

#### `sharedTexture.importSharedTexture(options)` _Experimental_

This method takes an object containing necessary info about the shared texture to import, and returns a holder object to the imported shared texture.

* `options` Object - The information of shared texture to import.
  * `pixelFormat` string - The pixel format of the texture. Can be `rgba` or `bgra`.
  * `colorSpace` [ColorSpace](https://github.com/electron/electron/tree/main/docs/api/structures/color-space.md) (optional) - The color space of the texture.
  * `codedSize` [Size](https://github.com/electron/electron/tree/main/docs/api/structures/size.md) - The full dimensions of the shared texture.
  * `visibleRect` [Rectangle](https://github.com/electron/electron/tree/main/docs/api/structures/rectangle.md) (optional) - A subsection of [0, 0, codedSize.width, codedSize.height]. In common cases, it is the full section area.
  * `timestamp` number (optional) - A timestamp in microseconds that will be reflected to `VideoFrame`.
  * `handle` [SharedTextureHandle](#sharedtexturehandle-object) - The shared texture data.

Returns [`SharedTextureImported`](#sharedtextureimported-object) - The imported shared texture.

#### `sharedTexture.finishTransferSharedTexture(transfer)` _Experimental_

This method takes a transfer object that might be passed from another process, and returns a holder object to the source shared texture, increase a cross process reference count to the underlying resource.

* `transfer` [SharedTextureTransfer](#sharedtexturetransfer-object) - The transfer object of the shared texture.

Returns [`SharedTextureImported`](#sharedtextureimported-object) - The imported shared texture from the transfer object.

### SharedTextureHandle Object

This object contains platform specific handles to the shared texture.

* `ntHandle` Buffer (optional) _Windows_ - NT HANDLE holds the shared texture. Note that this NT HANDLE is local to current process.
* `ioSurface` Buffer (optional) _macOS_ - IOSurfaceRef holds the shared texture. Note that this IOSurface is local to current process (not global).
* `nativePixmap` Object (optional) _Linux_ - Structure contains planes of shared texture.
  * `planes` Object[] _Linux_ - Each plane's info of the shared texture.
    * `stride` number - The strides and offsets in bytes to be used when accessing the buffers via a memory mapping. One per plane per entry.
    * `offset` number - The strides and offsets in bytes to be used when accessing the buffers via a memory mapping. One per plane per entry.
    * `size` number - Size in bytes of the plane. This is necessary to map the buffers.
    * `fd` number - File descriptor for the underlying memory object (usually dmabuf).
  * `modifier` string _Linux_ - The modifier is retrieved from GBM library and passed to EGL driver.
  * `supportsZeroCopyWebGpuImport` boolean _Linux_ - Indicates whether supports zero copy import to WebGPU.

### SharedTextureImported Object

This object is a holder to the imported shared texture, providing methods to interact with the underlying resource.

* `getVideoFrame` Function\<[VideoFrame](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame)\> - Create a `VideoFrame` that use the imported shared texture at current process. You can call `VideoFrame.close()` once you've finished using, the underlying resources will wait for GPU finish internally.
* `startTransferSharedTexture` Function\<[SharedTextureTransfer](#sharedtexturetransfer-object)\> - Create a `SharedTextureTransfer` that can be serialized and transfer to other processes.
* `release` Function - Release the resources. If you transferred and get multiple `SharedTextureImported`, you have to `release` it on every one of them. The resource on GPU process will be finally destroyed when last one is released.
  * `callback` Function (optional) - Callback when GPU command buffer finished using this shared texture. It provides a precise event to safely release dependent resources. For example, if this object is created by `finishTransferSharedTexture`, you can use this callback to safely release the original one that called `startTransferSharedTexture` in other processes. You can also release the source shared texture that was used to `importSharedTexture` safely.


### SharedTextureTransfer Object

This method is a serializable object that can be passed between processes, it's designed to transfer the imported shared texture to another process. The transfer doesn't mean hand over ownership to the receiving peer. The user should not modify any of the value in this object.

* `transfer` string - The opaque transfer data of the shared texture, can be transferred across Electron processes.
* `pixelFormat` string - The pixel format of the texture. Can be `rgba` or `bgra`.
* `codedSize` [Size](size.md) - The full dimensions of the shared texture.
* `visibleRect` [Rectangle](rectangle.md) - A subsection of [0, 0, codedSize.width(), codedSize.height()]. In common cases, it is the full section area.
* `timestamp` number - A timestamp in microseconds that will be reflected to `VideoFrame`.


## Guide-level explanation

### Example Usage Walkthrough

In this section, We will use offscreen rendering's output as a example of external shared texture, import and render it at another window.

1. Import `sharedTexture` module.

```js
const { sharedTexture } = require('electron');
```

2. Create a `BrowserWindow`, and enable the offscreen rendering feature, setting target window size.

```js
const osr = new BrowserWindow({
  show: debugSpec,
  webPreferences: {
    offscreen: {
      useSharedTexture: true
    }
  }
});

// It is suggested to use setSize, as setting width and height above at BrowserWindow may be contrainted by the screen size.
osr.setSize(3840, 2160)
```

3. Listen `paint` event, import the shared texture, and get a transfer object to pass it to renderer process.

The `handle` is a [SharedTextureHandle](#sharedtexturehandle-object), which contains platform specific information about the native handle.

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

  // Prepare for transfer to another process (win's renderer)
  const transfer = imported.startTransferSharedTexture();

  const id = randomUUID();
  capturedTextures.set(id, { imported, texture });

  // Send the shared texture to the renderer process (goto preload.js)
  win.webContents.send('shared-texture', id, transfer);
});
```

4. At `preload.js`, expose a API to main world, and receive the transfer object.

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
        // When GPU command buffer is done, we can notify the main process to release
        ipcRenderer.send('shared-texture-done', id);
      });
    });
  }
});
```

5. At `renderer.js`, prepare a canvas, a WebGPU rendering pipeline.

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

      // Create bind group layout, correctly specifying the external texture type
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

6. Get `VideoFrame` out of imported shared texture, and render it. When we submit WebGPU command buffer, we can call `close` on the `VideoFrame`.

```js
window.textures.onSharedTexture(async (id, imported) => {
  try {
    // Get VideoFrame from the imported texture
    const frame = imported.getVideoFrame();

    // Render using WebGPU
    await window.renderFrame(frame);

    // Release the VideoFrame as we no longer need it
    frame.close();
  } catch (error) {
    console.error('Error getting VideoFrame:', error);
  }
});
```

7. After the frame is processed, in `preload.js` we notify main proecss to release resources.

```js
ipcMain.on('shared-texture-done', (event: any, id: string) => {
  // Release the shared texture resources at main process
  const data = capturedTextures.get(id);
  if (data) {
    capturedTextures.delete(id);
    const { imported, texture } = data;

    // Release the imported shared texture
    imported.release(() => {
      // Release the shared texture once GPU is done
      texture.release();
    });
  }
});
```

### Cautions

The lifecycle of objects should be take good care of. Let's take the above use case as a example:

1. We first get a external shared texture, in this example from offscreen rendering, let's name it as `originTexture`.
2. We then import `originTexture` and get `importedTextureMain`.
3. We got a `transferObject` from `importedTextureMain`, and send that through IPC channel.
4. **Caution**: At this point, you can't release `importedTextureMain` as we have not finished importing at another process. 
5. We got the `transferObject` at renderer process, and get `importedTextureRenderer`.
6. **Caution**: At this point, you also can't release ``importedTextureMain`, as GPU tasks are done in async. All `importedTexture` is backed by a `Mailbox`, which has a cross process reference counter at GPU process. The counter won't increase until the command buffer actually run the `importedTextureRenderer` import task. Thus, if we release `importedTextureMain` now, we may let the counter reset to zero, making GPU destroy the mailbox, and cause the future actual import task fail.
7. We then get a `VideoFrame` from `importedTextureRenderer`, we then use `VideoFrame` at `WebGPU` and creates a `ExternalTexture`.
8. After we flush the command buffer, all GPU tasks are submitted. We can safely call `close` on `VideoFrame`, as the command is ensured to be executed after all submitted tasks, this was ensured by `SyncToken` which have a detailed explaination later.
9. For similar reason, we can then call `release` on `importedTextureRenderer`, as we've done everything at renderer process. We pass a callback to the `release` function, which use `SyncToken` to ensure the callback is triggered after all submitted tasks are executed by GPU, and it's safe to do further release.
10. Now, we can notify main process to release `importedTextureMain`. It's suggested to use callback to release `originTexture` safely.

![Lifecycle](../images/0017/lifecycle.png)

## Reference-level explanation

### Design Considerations

The main goal of the current implementation is to import external shared texture descriptions while reusing as much as possible of the existing Chromium infrastructure to minimize the maintainance burden. I choose `SharedImage` as the underlying holder of the external texture because it provides `SharedImageInterface::CreateSharedImage`, which accepts a `GpuMemoryBufferHandle` containing native shared handle types: `NTHANDLE` for Windows, `IOSurfaceRef` for macOS, and `NativePixmapHandle` with file descriptors for each plane for Linux. However, during research, several critical problems results in the final design as below describes.

#### Shared handle is local to process

For Windows, a shared D3D11 texture can be created by `GetSharedHandle` (deprecated) or `CreateSharedHandle`. The deprecated method generates a `non-NTHANDLE` that can be globally accessed, while the newer one generates an `NTHANDLE` that is local to the current process. To share it with other processes, you need to call `DuplicateHandle` to create this handle in the remote process.

For macOS, `IOSurface` can also be global by setting `kIOSurfaceIsGlobal`, which is also a deprecated option. If you want to share an `IOSurface` with other processes, you need to create a `mach_port` from it and pass the `mach_port` through a previously created IPC (which also uses `mach_port` as transport), making it significantly more complex than Windows.

Given these obstacles, you must ensure that when calling `sharedTexture.importSharedTexture`, the handle is already available to the current process. You might be able to use global `IOSurface`, but a `non-NTHANDLE` is not an option as described in problem 2 below. 

In fact, Chromium's IPC internally handles all the concerns about this (duplicate handles for remote processes, passing `mach_port` through `mach_port`), which is why the OSR `paint` event can use the handle directly at main process, while the original handle is generated at GPU process - because IPC has transparently handled these issues.

#### Shared handle ownership management

Chromium takes ownership of the `GpuMemoryBuffer` and its native representation. Once the resource is destroyed, the handle will be closed. Thus, the handle user import must let Chromium take ownership.

For Windows, calling `CloseHandle` on a `non-NTHANDLE` is an invalid operation and will cause Chromium to crash. Therefore, you cannot use a `non-NTHANDLE` when importing. In the future, we may provide a helper for this. To work with this design, when calling `importSharedTexture`, an `NTHANDLE` will be duplicated internally for user, which means user still have ownership of the handle.

For macOS, `IOSurface` is a reference-counted resource. When calling `importSharedTexture`, instead of taking ownership, we can simply let Chromium retain this resource and increment the reference count, which also means user still have the ownership.

#### Transfer the shared texture between processes

I initially considered using WebGPU Dawn Native API to import the external texture as a `WGPUSharedTextureMemory`, but encountered more problems. For example, it was unable to export, difficult to manage the lifetime of a frame, and working with WebGPU was non-ideal.

`SharedImage` has advantages when it comes to sharing across processes because it holds a reference to a `Mailbox`, which points to the corresponding `SharedImageBacking` in the GPU process. Therefore, I use `SharedImageInterface::ImportSharedImage` and `ClientSharedImage::Export` to serialize sufficient information to retrieve the `SharedImage` reference in another process, and it can also reuse the mojo serializer to serialize as a string. Everything is taken care of by Chromium.

The lifecycle management is a little bit tricky, when you get a `SharedImage` which holds a `Mailbox` from either `ImportSharedImage` or `CreateSharedImage`, the `Mailbox` is just a placeholder, it doesn't ensure the GPU actually imported the texture, and increased a reference count to the `Mailbox`. Chromium doesn't wait for the GPU to finish, typically use a `SyncToken` to ensure GPU task dependencies. For example, releasing a `SharedImage` is also a asynchronous operation, and calling a release often requires a `SyncToken` that put a barrier to ensure all GPU importing and rendering tasks are finished. In our case, since we can't use a mojo callback to update `SyncToken` across processes, we have to use callbacks to notify the dependent resources to release.

#### Interprocess resource management

The final obstacle is managing the lifetime of a frame. Currently, I use OSR to get an exported shared texture generated by Chromium itself. As the documentation states, the texture needs to be manually released. By importing this texture into a `SharedTextureImported` in Electron, we must ensure the imported one is released before the source texture is released.

What's more challenging is that the `paint` event occurs in the main process, while we have to render in renderer processes, so we must use `startTransferSharedTexture` to pass the underlying `SharedImage` to another process.

Most GPU calls are asynchronous, sending to the command buffer of the GPU process through the `GpuChannel` of each client process. When we have two `SharedImage` instances referencing the same `Mailbox` in two different processes, we don't know when the GPU has finished using the resources. Typically, this is guaranteed by `SyncToken`. For example, we can simply schedule the destruction of a `SharedImage` with an empty `SyncToken` in the main process (where the texture was first imported), but when we use the same `SharedImage` in the renderer process and use it in WebGPU (or WebGL) pipelines, the destruction token will be generated by WebGPU to prevent destruction before the GPU uses it. The destruction won't occur until WebGPU rendering finishes.

Ideally, if we are in Chromium, we can use mojo to create a `PendingRemote` callback and update the main process destruction `SyncToken` to prevent the main process from releasing the frame before the GPU starts working on it. In the future, we may implement this in Electron code to wrap the mojo functionality. Eventually, I found a way to use `gpu::ContextSupport` and register a callback when a specific `SyncToken` is released (signaled). When you call `release()` on the imported shared texture object, if you've used `VideoFrame` and imported it into a WebGPU pipeline, it will wait for WebGPU to finish rendering, then run a callback to notify you to release dependent resources, such as the original imported object in the main process, the source texture, etc.

### New Dependencies

This feature mainly depends on `SharedImage` API in Chromium.

## Drawbacks

The Chromium's `SharedImage` may have minor API changes in the future, for example `VideoFrame` might be able to manage the lifetime of the underlying `SharedImage` automatically (which is a good thing), that may introduce minor maintainance burden.

## Rationale and alternatives 

### Rationale

This design is almost a optimal solution, since it requires zero upstream patch, and already take into considerations of all Chromium internal design constraints. 

### Alternatives

Besides using `VideoFrame`, one possible alternative is directly using `WebGPU`, which requires big patch to Dawn and Chromium, and it's given up.

About the lifecycle management, I did consider passing callback like `MessagePort` does. After some research, although it can be used to create a IPC channel by using blink `MessagePortDescriptor`, I think this doesn't ease things much:

- You still have to ensure the remote process done calling finishTransferSharedTexture (that's when it transmits a new SyncToken back), until you can drop the reference or call release. It's still a async procedure.
- You still don't know when to release the original native handle. A callback is still needed to notify a safe release.

So, I decide not use the automated `SyncToken` IPC, and continue use release callback to provide a precise and safe timing for you to manually release the dependency.

## Prior art

Prior to this, users may import external images via video streams or bitmaps, which consumes CPU and have visual loss. By using shared texture we can import GPU resources directly, which is a more preferable way to display arbitrary content in web applications. 

## Unresolved questions

- Dealing with native shared handle, can we provide this feature in sandboxed renderer?
- Any other web standard besides `VideoFrame` can be used to let user render the texture?

## Future possibilities

- Performance, I think it is meaningful to profile how much overhead it has when managing lifecycle through Electron IPC callbacks.
- OSR, As we can import a shared texture as a `SharedImage`, it is possible to implement OSR using `SharedImage` instead of `FrameSinkVideoCapturer`.
