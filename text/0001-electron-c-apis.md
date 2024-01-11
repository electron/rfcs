- Start Date: 2024-01-11
- RFC PR: [electron/rfcs#3](https://github.com/electron/rfcs/pull/3)
- Status: **Proposed**

# Electron C APIs

## Summary

Similar to the [Node-API (also known as N-API) of Node.js][napi], the Electron C APIs is a set of C APIs providing access to Electron and Chromium’s resources, which can be used in Node.js native modules.

## Motivation

### Easier to add new APIs

The majority of Electron’s JavaScript APIs are about accessing native resources of the operating system; some of them could not be implemented as native modules because they need to interact with Electron and Chromium’s resources.

For example, the `TouchBar` and `inAppPurchase` APIs must live as built-in APIs because the JavaScript bindings must interact with internal `ElectronApplicationDelegate` and `ElectronNSWindowDelegate`, and there is no non-intrusive way doing that outside Electron.

With Electron C APIs we can make it possible to have those APIs as native modules. This can make Electron less bloating since we don't have to make every new API a built-in API , and for users who want to add a large set of new APIs they can now only request a few minimal C APIs in Electron and then implement anything they want in native modules.

### Reduce Electron forks

Some people may already know that the some Electron apps like VS Code and Discord use custom builds instead of the offical one, this is because they have some modifications that can not be upstreamed (for example some hacks to the `<video>` tag). By exposing certain internal Chromium APIs as Electron C APIs, we can make those modifications live as native modules instead of having a custom build.

## Guide-level explanation

To make use the Electron C APIs, module authors should check corresponding feature flags and include correct headers.

An example of checking the Electron version:

```c
#if !defined(ELECTRON_API)
#error "The target Electron version does not have C APIs."
#endif

#include "electron/version.h"

#if ELECTRON_MAJOR_VERSION < 25
#error "Building this module requires Electron >= 25."
#endif
```

An example of importing the imagenary skia APIs:

```c
#if !defined(ELECTRON_API_SKIA)
#error "The target Electron version does not have skia APIs."
#endif

#include "electron/skia.h"
```

## Reference-level explanation

### C Defines

In the GYP files shipped in the headers tarball of Electron, it should have the `ELECTRON_API` define which indicates the Electron C APIs are supported.

For every new set of APIs added, a new define should be added to allow feature testing in the native modules, and the define's name should start with `ELECTRON_API_`. For example, when adding the imagenary skia APIs, the `ELECTRON_API_SKIA` define should be added.

The defines should only be added for new set of APIs instead of individual APIs. For example when adding the imagenary `sk_path_add_arc` API to the existing skia APIs, no define should be added and module authors should check the Electron version to test the existence of the API.

### C Headers

The header files of Electron C APIs are put in the `electron/` directroy in the headers tarball,
which can be included in native modules with `#include "electron/xxx.h"`.

The `electron/version.h` header defines the current version of Electron.

For new APIs added, their header file should be a single `electron/xxx.h` file, or be placed under the `electron/xxx/` directory, or both.

Using the imagenary skia APIs as example, the implementation may choose to use a single `electron/skia.h` header file, or split the APIs into multiple headers and put them into the `electron/skia/` directory, for example `electron/skia/path.h` and `electron/skia/surface.h` files. It is also allowed to use both, i.e. a `electron/skia.h` header to include all APIs while still having `electron/skia/xxx.h` for sub-APIs.

### API names

There is no restriction on API names, and it is not recommeneded to prefix the APIs with `electron_` unless they directly manipulates things of Electron.

This is because the actual implementations of C APIs may not necessarily reside in Electron's codebase, and we may not have control of its API names.

Using the imagenary skia APIs as example, instead of implementing the C APIs inside Electron, we may choose to use a popular open source implementation and build it as part of Electron. The open source implementation may use `sk_` to prefix API names and we don't want to change that.

## Drawbacks

### Maintainance burden

Most components of Chromium are written in C++, and to expose C APIs for them a separate C wrapper must be written, and they may require extra maintaince work during Chromium upgrades.

A possible stratege to solve this is to ask the implementation of C wrapper to live outside Electron's codebase, and only be included in Electron as an optional componenet which can be easily turned off, so it won't add up to the maintaince work of Chromium upgrades.

Using the imagenary skia APIs as example, there is already an open source implementation of C APIs for skia (https://github.com/richardwilkes/cskia), building it as a component of Electron should only require minimum maintaince work.

### External JavaScript bindings

For Electron apps to make use of the C APIs, they usually want to use JavaScript APIs instead of writing all the code in C. In which case a JavaScript bindings also needs to be implemented for the C APIs, and the separated layer can be difficult to use when the JavaScript bindings get out of sync with changes in the C APIs.

## Rationale and alternatives

The alternative of C APIs is to expose C++ APIs, and by simply modifying visibility settings is possible to just expose C++ APIs of a component in Chromium without any C API wrapper, which adds almost no overheads.

But the C++ APIs of Chromium components are very unstable and may change for every Chromium upgrade, which would basically make them unusable for module authors. Even for the stable ones like V8, correctly building the native module would still be a challenge.

## Prior art

* [Node-API][napi]
* [CEF's C APIs](https://bitbucket.org/chromiumembedded/cef/wiki/UsingTheCAPI.md)

## Unresolved questions

The API and ABI stability of C APIs are not covered in this RFC. I think if a set of C APIs has been depended on so much that we actually need to promise API and ABI stability for them, we should probably make them built-in JavaScript APIs, or the independent C library version of the APIs should have already gained enough popularity that it no longer needs Electron team's resources to go on developing.

## Future possibilities

With Electron C APIs, it will be easier to determine whether a JavaScript API should be added to Electron.

We can ask following questions when a new feature is requested:

* Will the API benefit only a very small portions of apps?
* Is the API a simple wrapper of Chromium components?
* Can the API be implemented as a native module?

If they are all true we can suggest making the feature an Electron C API instead.

[napi]: https://nodejs.org/api/n-api.html