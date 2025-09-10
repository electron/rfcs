# RFC 
- Start Date: 2025-03-20
- RFC PR: [electron/rfcs#16](https://github.com/electron/rfcs/pull/16)
- Electron Issues: [electron/electron#526](https://github.com/electron/electron/issues/526)
- Reference Implementation: https://github.com/electron/electron/tree/gsoc-2025
- Status: **Proposed**

# Save/Restore Window State API

Currently, Electron does not have any built-in mechanism for saving and restoring the state of BaseWindows, but this is a very common need for apps that want to feel more native.

## Summary

This proposal aims to implement a save/restore window state API for Electron by providing a simple but powerful configuration object `windowStatePersistence` that handles complex edge cases automatically. This approach offers a declarative way to configure window persistence behavior while maintaining flexibility for different application needs. The object would be optional in the `BaseWindowConstructorOptions`.

## Motivation

**Why should we do this?**

Window state persistence represents a core functionality that many production Electron apps implement. By elevating it to a first-class feature, Electron acknowledges its essential nature and provides a standardized approach directly in the framework.

**What use cases does it support?**

It's useful for developers who want to implement save/restore window state for their apps reliably. Different apps have different needs. An example use case for this API would be for when apps want to preserve window position (with display overflow) and size across multi-monitor setups with different display scales and resolutions.

`windowStatePersistence` aims to cover most save/restore use cases while providing room for future extensions.

**What is the expected outcome?**

Electron provides an option to save and restore window state out of the box with multiple configurations which will cover almost all application needs.

While established applications with custom implementations may continue using their existing solutions, this feature will most likely see adoption by many smaller projects.

## Implementation Details  

Before diving into the API specifications which is the next section, I’d like to outline the approach for saving a window’s state in Electron.  

We can use Chromium's [PrefService](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/pref_service.h) to store window state as a JSON object into the already existing Local State folder inside *app.getPath('userData')*.

We schedule an async write to disk every time a window is moved or resized and batch write every 10 seconds (explained later why). We also write to disk on app quit synchronously. **This continuous writing approach would enable us to restore the window state even when there are crashes or improper app exits.**

**Questions that might arise:**

**Why 10 seconds?**
The default time window used by Chromium's PrefService to flush to disk is 10 seconds [[reference](https://source.chromium.org/chromium/chromium/src/+/main:base/files/important_file_writer.cc;l=48;drc=ff37fb5e8b02e0c123063b89ef2ac3423829b010)].
I think reusing the existing implementation and retaining the 10-second commit interval would be a simple and effective approach.

**What if the app crashes in the 10 second window?** The window could possibly restore where it was a few seconds ago (worst case 10 seconds ago). This wouldn't be the worst user experience in the world.

#### *Why PrefService?* 

- Atomic writes using ImportantFileWriter [[reference](https://source.chromium.org/chromium/chromium/src/+/main:base/files/important_file_writer.h;l=28-29)]
- Async writes which means no blocking of the UI thread, preventing jank during window state updates [[reference](https://docs.google.com/document/d/1rlwl_GvvokatMiRUUkR0vGNcraqe4weFGNilTsZgiXA/edit?tab=t.0#bookmark=id.49710km71rd7)]
- Synchronous writes on demand [[reference](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/pref_service.h;l=199-209;drc=51cc784590d00e6a95b48e1e1bf5c3fe099edf64)]
- Offers batched scheduled writes (using ImportantFileWriter) every 10 second window [[reference](https://source.chromium.org/chromium/chromium/src/+/main:base/files/important_file_writer.cc;l=48;drc=ff37fb5e8b02e0c123063b89ef2ac3423829b010)].
- Already implemented in Electron to preserve the devTools bounds [[reference](https://github.com/electron/electron/blob/73a017577e6d8cf67c76acb8f6a199c2b64ccb5d/shell/browser/ui/inspectable_web_contents.cc#L509)]
- Provides identical behavior across Windows, macOS, and Linux, simplifying implementation and maintenance
- Falls back to default values automatically when corrupted data is encountered [[reference](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/pref_service.h;l=233-235;drc=51cc784590d00e6a95b48e1e1bf5c3fe099edf64)]
- Safe failing to ensure apps don't crash on data corruption [[reference](https://source.chromium.org/chromium/chromium/src/+/main:components/prefs/json_pref_store.cc;l=91-95;drc=ccea09835fd67b49ef0d4aed8dda1a5f22a409c8)]

<br/>

Here's the schema I propose for the `windowStatePersistence` object. This is simply for reference. The explanation for each field in the object will be in the API specification section which is next.

<br/>

`BaseWindowConstructorOptions` schema - This would be passed by the developer in the `BaseWindowConstructorOptions`/`BrowserWindowConstructorOptions`

```json
{
    "name": string,
    "windowStatePersistence": {
        "bounds": boolean,
        "displayMode": boolean
    }
}
```
OR

```json
{
    "name": string,
    "windowStatePersistence": boolean
}
```

> [!NOTE]
> The entire window state (bounds, maximized, fullscreen, etc.) would be saved internally if name and windowStatePersistence are passed.
> We would use the saved state and enforce the chosen config during restoration the next time the window is constructed. Setting `windowStatePersistence": true` is equivalent to setting `bounds: true` and `displayMode: true`

#### **State Persistence Mechanism**  

Firstly, all the window states with their `windowStatePersistence` would be to loaded from disk synchronously during the startup process just like other preferences in Electron. Doing so would allow us the have the states in memory during window creation with minimal performance impact. 

Secondly, once the states are loaded into memory, we can use them to restore the window state with the `windowStatePersistence` rules in place during window creation. An algorithm that handles all the edges would be required for this. Chromium's [WindowSizer::AdjustBoundsToBeVisibleOnDisplay](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/window_sizer/window_sizer.cc;drc=0ec56065ba588552f21633aa47280ba02c3cd160;l=350) seems like a good reference point that handles some of the edge cases. Also, the default options provided in the `BaseWindowConstructorOptions`/`BrowserWindowConstructorOptions` constructor options would be overridden (if we are restoring state).

We can respect the min/max height/width, fullscreenable, maximizable, minimizable properties set inside `BaseWindowConstructorOptions` if applicable. Meaning these properties would take a higher priority during the restoration of a window.

Each time a window is **moved** or **resized**, we schedule a write using Chromium's ScopedDictPrefUpdate [[example](https://github.com/electron/electron/blob/4ad20ccb396a3354d99b8843426bece2b28228bf/shell/browser/ui/inspectable_web_contents.cc#L837-L841)].

Here's a reference to the variables we would saving be internally using PrefService https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/views/chrome_views_delegate.cc;drc=1f9f3d7d4227fc2021915926b0e9f934cd610201;l=85. Additional to these we will also save the current displayMode `fullscreen`, `maximized`, or `kiosk`.

#### Developer-Facing Events

I’m also considering emitting only a single event: `restored-window-state` which would be emitted after window construction. I'm unsure about the rest  `will-restore-window-state`, `will-save-window-state` and `saved-window-state`. My doubts regarding these are

1) Static event for `will-restore-window-state`? Since restoration happens during window construction, should we add `BaseWindow.on('will-restore-window-state', ...)` as a static event to allow `e.preventDefault()` call before the JS window object exists? Without a static event it won't be possible to listen to this event meaningfully.
2) Save events: Is it worth adding `will-save-window-state` and `saved-window-state`? Chromium's PrefService doesn't emit events, so this might be tricky to implement cleanly.

#### Event: 'restored-window-state'

Emitted immediately after the window constructor completes.

```js
win.on('restored-window-state', () => {
  console.log('Window state restored');
});
```

## API Specification

Here's how it would land in the electron documentation.

### BaseWindowConstructorOptions

* `name` string (optional) - A unique identifier for the window to enable features such as state persistence. An error will be thrown in the constructor if a window exists using the same identifier. It can only be reused after the corresponding window has been destroyed. An error is thrown if the name is already in use. This is not the visible title shown to users on the title bar.

* `windowStatePersistence` ([WindowStatePersistence] | Boolean) (optional) - Configures or enables the persistence of window state (position, size, maximized state, etc.) across application restarts. Has no effect if window `name` is not provided. _Experimental_

### WindowStatePersistence Object

* `bounds` boolean (optional) - Whether to persist window position and size across application restarts. Defaults to `true` if not specified.

* `displayMode` boolean (optional) - Whether to persist display modes (fullscreen, kiosk, maximized, etc.) across application restarts. Defaults to `true` if not specified. 

We could add a tutorial on how the new constructor option can be used effectively.

```js
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  ...existingOptions, 
  windowStatePersistence: {
    displayMode: false
  },
})
```

### API Design Rationale:
- Everything is set to true by default so that the developer can set whatever they want to exclude to false. This means less lines of code as more options are added to the config.
- Displays are considered the same if they span over the same work_area (x, y) and have the same dimensions.
- A window can never be restored out of reach. I am presuming no apps would want this behavior.
- If window (width, height) reopens on a different display and does not fit on screen auto adjust to fit and resave the value. This would reduce the number of edge cases significantly and I highly doubt that any app would want to preserve an overflow when opened on a different display.
- Not handling scaling as Windows, macOS, and Linux support multimonitor window scaling by default.

### Additional APIs

#### `BaseWindow.clearPersistedState(name)`

Static function over BaseWindow.

* `name` string - The window `name` to clear state for (see [BaseWindowConstructorOptions](structures/base-window-options.md)).

Clears the saved state for a window with the given name. This removes all persisted window bounds, display mode, and work area information that was previously saved when `windowStatePersistence` was enabled.

If the window `name` is empty or the window state doesn't exist, the method will log a warning.

### Algorithm for saving/restoring the window state
As mentioned before, we would take a continuous approach to saving window state if `name` and `windowStatePersistence` are passed through the constructor.

The algorithm would be simple two step approach:

**Calculate new bounds:**
This step involves checking if the window is reopened on the same display by comparing the current work_area to the saved work_area, setting minimum height and width for the window (100, 100), ensuring proper fallback behaviour for when window is restored out of bounds or with overflow
There are some tricks we can use from [WindowSizer::AdjustBoundsToBeVisibleOnDisplay](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/window_sizer/window_sizer.cc;drc=0ec56065ba588552f21633aa47280ba02c3cd160;l=350)

**Set window state:**
Once we have the width, height, x, y, and displayModes we can set the window state. `displayMode` would take precedence over the saved bounds.

## Guide-level explanation

*Explain the feature as if it were already implemented in Electron and you were teaching it to
an Electron app developer.*

Electron is introducing `windowStatePersistence`, an optional object/boolean in the `BaseWindowConstructorOptions` and would also be available in `BrowserWindowConstructorOptions`.
It can be used to save and restore your window state (x, y, height, width, fullscreen, etc.)

`windowStatePersistence` schema - Everything is optional. Developers can choose what they want to preserve and how they want to restore it or simply set it to true. The default behaviour in this case would be `bounds` and `displayMode` both are restored.

```json
"name": string,
"windowStatePersistence": true
```
OR
```json
"name": string, 
"windowStatePersistence": {
    "bounds": boolean,
    "displayMode": boolean
  }
```

Here's an example that would let you restore bounds (x, y, width, height) but not the display mode (maximized, fullscreen etc)
```js
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  ...existingOptions, 
  name: '#1230',
  windowStatePersistence: {
    displayMode: false
  }
})
```

*Discuss how this impacts the ability to read, understand, and maintain Electron code. Will the
proposed feature make Electron code more maintainable? How difficult is the upgrade path for
existing apps?*

It would be the same code for Windows, macOS, and Linux using Chromium's PrefService. I think it would be easy to read, understand, and maintain the new Electron code.

The path to upgrade for apps would be for developers to remove their existing implementation and use this new API if they want to. If their implementation is on the JavaScript side it will always execute their restore logic after window construction which means our implementation would be overridden.

## Reference-level explanation

Covered in the [Implementation Details](#implementation-details) section.

## Drawbacks

- Writing to disk everytime the window is moved or resized in a batched 10 second window might not be necessary and better to write to disk only on window close synchronously.
- Similar outcomes could be achieved via JavaScript APIs with miniscule performance difference. The only issue being the window state is not set at the right time in the window lifecycle.

Why should we *not* do this?

- It's not a critical issue.
- Adds maintenance burden for the Electron team to support this feature long-term.

## Rationale and alternatives

*Why is this design the best in the space of possible designs?* 
  
  Overall, providing a constructor option is the best design in my opinion. It provides maximum flexibility and future-proofing for different requests in the future. It also sets the window properties at the right time in the window lifecycle. Although not perfect right now, it can be improved by the community based on different use cases quite easily. We're also saving the window state in a continuous manner so it can be restored even after crashes.

*What other designs have been considered and what is the rationale for not choosing them?*
  
  I've considered a JavaScript API that could be used. But it's not the best option since it would be setting the window state again after creation.

*What is the impact of not doing this?*
  
It's not a critical issue. App windows might vanish on rare occasions.

*If this is an API proposal, could this be done as a JavaScript module or a native Node.js add-on?*
  
The real value proposition isn't that this functionality can't be implemented in JavaScript - it absolutely can and has been attempted through various community libraries. Rather, the value lies in providing a standardized, well-maintained solution integrated directly into Electron's core. There's also a miniscule performance benefits as it would avoid extra ipc calls while restoring window state. 

## Prior art

Electron devTools persists bounds using PrefService. Implementation can be seen here [inspectable_web_contents.cc](https://github.com/electron/electron/blob/73a017577e6d8cf67c76acb8f6a199c2b64ccb5d/shell/browser/ui/inspectable_web_contents.cc#L509). 
It also seems likely Chrome uses PrefService to store their window bounds [reference](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/prefs/README.md).

From my research, I found out native applications built directly with the operating system's own tools and frameworks often get window state persistence for free (macOS, Windows).
I thought it would be inappropriate to enforce such rules on Electron apps. Thus, `windowStatePersistence` to provide flexibility and the choice to opt-in to this behavior.

## Unresolved questions

*What parts of the design do you expect to resolve through the RFC process before this gets merged?*
- Should we switch from `name` to something more descriptive? Should we use the existing unique identifier `id` and allow developers to pass a unique `id`?
- Static event for `will-restore-window-state`? Since restoration happens during window construction, should we add `BaseWindow.on('will-restore-window-state', ...)` as a static event to allow `e.preventDefault()` call before the JS window object exists? Without a static event it won't be possible to listen to this event meaningfully.
- Save events: Is it worth adding `will-save-window-state` and `saved-window-state`? Chromium's PrefService doesn't emit events, so this might be tricky to implement cleanly.

## Future possibilities

1) Introduce custom behavior under `fallbackBehaviour` and `openBehaviour` parameters based on community requests.

    * `openBehaviour` string (optional) - Special behavior when the window is opened.

      * `openOnLaunchedDisplay` - Opens the window on the display that it was launched from always.

      * `openOnPrimaryDisplay` - Opens the window on the primary display always.


    * `fallbackBehaviour` string (optional) - Fallback behaviour when position is out of bounds.

      * `"snapToNearestDisplayEdge"` - Snaps the window to the nearest display edge.

      * `"centerOnNearestDisplay"` - Centers the window on the nearest display.

      * `"centerOnPrimaryDisplay"` - Centers the window on the primary display.


    We would need an algorithm that calculates bounds based on these parameters. Many things would be needed to be taken into consideration to accomplish this.

    The algorithm to restore window state with the newly introduced options `fallbackBehaviour` and `openBehaviour` is detailed [here](https://gist.github.com/nilayarya/48d24e38d8dbf67dd05eef9310f147c6#algorithm-for-savingrestoring-the-window-state). One particularly cool feature would be to provide an option to restore the window on closest available display/space dynamically.

2) The `bounds` property that is suggested above could possibly accept an object or boolean (boolean in current proposal). The object would allow more configurability and control over reopening windows on different monitors with different dpi scaling and resolution.

3) We could add `allowOverflow` property inside the `bounds` object to control the restore overflow behaviour (some apps would specifically like to not restore in an overflown state). In our current implementation we won't be considering this and can have something like [this](https://source.chromium.org/chromium/chromium/src/+/main:chrome/browser/ui/window_sizer/window_sizer.cc;drc=0ec56065ba588552f21633aa47280ba02c3cd160;l=402) for the time being.

4) APIs to allow changes in `windowStatePersistence` during runtime for apps that want to let users of the application decide save/restore behaviour.

5) APIs to allow changes to the saved window state on disk such as `BrowserWindow.getWindowState([name])` and `BrowserWindow.setWindowState([stateObj])` might be useful for cloud synchronization of window states as suggested by this comment https://github.com/electron/rfcs/pull/16#issuecomment-2983249038

6) Additional API called `win.restorePreferences([options])` to restore other properties on the `BaseWindow`

> [!NOTE]
> We always save these properties internally. Calling this API and restoring these properties would be set on the window in order passed by in the options object. It would be the equivalent of calling the instance methods on BaseWindow/BrowserWindow in the same order. For example, win.setAutoHideMenuBar(true).

#### `win.restorePreferences([options])`

* `options` Object

  * `autoHideMenuBar` boolean (optional) - Restore window's previous autoHideMenuBar state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>Linux</kbd></span>
  
  * `focusable` boolean (optional) - Restore window's previous focusable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `visibleOnAllWorkspaces` boolean (optional) - Restore window's previous visibleOnAllWorkspaces state. Default: `false`<span>&nbsp;&nbsp;  <kbd>macOS</kbd></span>

  * `shadow` boolean (optional) - Restore window's previous shadow state. Default: `false`

  * `menuBarVisible` boolean (optional) - Restore window's previous menuBarVisible state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>Linux</kbd></span>

  * `representedFilename` boolean (optional) - Restore window's previous representedFilename. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `title` boolean (optional) - Restore window's previous title. Default: `false`

  * `minimizable` boolean (optional) - Restore window's previous minimizable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `maximizable` boolean (optional) - Restore window's previous maximizable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `fullscreenable` boolean (optional) - Restore window's previous fullscreenable state. Default: `false`

  * `resizable` boolean (optional) - Restore window's previous resizable state. Default: `false`

  * `closable` boolean (optional) - Restore window's previous closable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `movable` boolean (optional) - Restore window's previous movable state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `excludedFromShownWindowsMenu` boolean (optional) - Restore window's previous excludedFromShownWindowsMenu state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `accessibleTitle` boolean (optional) - Restore window's previous accessibleTitle. Default: `false`

  * `backgroundColor` boolean (optional) - Restore window's previous backgroundColor. Default: `false`

  * `aspectRatio` boolean (optional) - Restore window's previous aspectRatio. Default: `false`

  * `minimumSize` boolean (optional) - Restore window's previous minimumSize (width, height). Default: `false`

  * `maximumSize` boolean (optional) - Restore window's previous maximumSize (width, height). Default: `false`

  * `hiddenInMissionControl` boolean (optional) - Restore window's previous hiddenInMissionControl state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `alwaysOnTop` boolean (optional) - Restore window's previous alwaysOnTop state. Default: `false`

  * `skipTaskbar` boolean (optional) - Restore window's previous skipTaskbar state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `opacity` boolean (optional) - Restore window's previous opacity. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `windowButtonVisibility` boolean (optional) - Restore window's previous windowButtonVisibility state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `ignoreMouseEvents` boolean (optional) - Restore window's previous ignoreMouseEvents state. Default: `false`

  * `contentProtection` boolean (optional) - Restore window's previous contentProtection state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>macOS</kbd></span>

  * `autoHideCursor` boolean (optional) - Restore window's previous autoHideCursor state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `vibrancy` boolean (optional) - Restore window's previous vibrancy state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `backgroundMaterial` boolean (optional) - Restore window's previous backgroundMaterial state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd></span>

  * `windowButtonPosition` boolean (optional) - Restore window's previous windowButtonPosition state. Default: `false`<span>&nbsp;&nbsp; <kbd>macOS</kbd></span>

  * `titleBarOverlay` boolean (optional) - Restore window's previous titleBarOverlay state. Default: `false`<span>&nbsp;&nbsp; <kbd>Windows</kbd> <kbd>Linux</kbd></span>

  * `zoomLevel` boolean (optional) - Restore window webcontent's previous zoomLevel. Default: `false`

  * `audioMuted` boolean (optional) - Restore window webcontent's previous audioMuted state. Default: `false`

  * `isDevToolsOpened` boolean (optional) - Restore window webcontent's previous isDevToolsOpened state. Default: `false`

  * `devToolsTitle` boolean (optional) - Restore window webcontent's previous devToolsTitle. Default: `false`

  * `ignoreMenuShortcuts` boolean (optional) - Restore window webcontent's previous ignoreMenuShortcuts state. Default: `false`

  * `frameRate` boolean (optional) - Restore window webcontent's previous frameRate. Default: `false`

  * `backgroundThrottling` boolean (optional) - Restore window webcontent's previous backgroundThrottling state. Default: `false`


```js
const { BrowserWindow } = require('electron')
const win = new BrowserWindow({
  ...existingOptions,
  name: '#1230',
  windowStatePersistence: true
});


win.restorePreferences({ 
  autoHideMenuBar: true,
  focusable: true,
  visibleOnAllWorkspaces: true,
  shadow: true,
  menuBarVisible: true,
  representedFilename: true,
});
```

Preferences will be restored in the order of the options object passed during savePreferences. Relevant events would be emitted with the state of the restored window in an object.

Overall, the `windowStatePersistence` is very configurable so I think it's future-proof. More configurations can be added based on community requests.
