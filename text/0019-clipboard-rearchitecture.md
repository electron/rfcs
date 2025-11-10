# RFC

- Start Date: 2025-08-07
- RFC PR: [electron/rfcs#19](https://github.com/electron/rfcs/pull/19)
- Electron Issues:
  - [electron/electron#23156](https://github.com/electron/electron/issues/23156)
  - [electron/electron#31130](https://github.com/electron/electron/issues/31130)
  - [electron/electron#31115](https://github.com/electron/electron/issues/31115)
  - [electron/electron#30699](https://github.com/electron/electron/issues/30699)
  - [electron/electron#26377](https://github.com/electron/electron/issues/26377)
  - [electron/electron#11838](https://github.com/electron/electron/issues/11838)
  - [electron/electron#9035](https://github.com/electron/electron/issues/9035)
  - [electron/electron#5078](https://github.com/electron/electron/issues/5078)
  - [electron/governance#229](https://github.com/electron/governance/pull/229)
- Reference Implementation: 
- Status: **Proposed**

# Clipboard Module Rearchitecture

## Summary

This RFC outlines a rearchitecturing of the [`clipboard`](https://www.electronjs.org/docs/latest/api/clipboard) module in Electron to improve alignment with the Clipboard API as specified by the [W3C](https://w3c.github.io/clipboard-apis/#clipboard-interface).

## Motivation

Chromium has made significant changes to their Clipboard API, including the ability to handle [Custom Formats](https://bugs.chromium.org/p/chromium/issues/detail?id=106449). Electron has had to break small aspects of its clipboard API to account for this, and it's unclear what other changes we may be forced to adapt to as they continue this work. Beyond this, users have been asking for an improved Clipboard API as well as surfaced a number of bugs with its current implementation - the most detailed of which is [here](https://github.com/electron/electron/issues/23156).

Electron provided a [`clipboard`](https://www.electronjs.org/docs/latest/api/clipboard) API to its users preceding the specification of the [Clipboard API by the W3C](https://w3c.github.io/clipboard-apis/#clipboard-interface). As a result, we locked into an API surface and created an API contract with our users that became progressively harder to alter without breaking said contract while the web drove a more standardized approach. On top of this confusion, the [`clipboard`](https://www.electronjs.org/docs/latest/api/clipboard) module provided by Electron is a dual-process module, meaning that it exists in parallel to the more robust API provided by Chromium in the renderer process via `navigator.clipboard` global. 

At this point, it becomes clear that it is in users' and maintainers' best interest to reconsider how we approach the `clipboard` module - breaking changes might be difficult in the short term but provide more consistency and functionality as we look to the future of the module.

## Guide-level explanation

### API Design

The largest consideration for changes to the `clipboard` module is to what degree we should choose to align with the W3C specification. As we consider this, we should consider what needs or use cases might exist for Electron uses which necessarily wouldn't exist for Chrome users, and how we would plan to account for that in our new design.
 
#### Preferred Approach/Solution

This design recommends alignment with W3C where possible, with some additional APIs exposed to handle desktop use cases which aren’t considered by the specification.  Given this approach, we will preserve some of the existing APIs, remove some of the existing APIs, and, modify the rest of the APIs to be in compliance with the W3C spec as much as possible.

**APIs to add**

### Event: 'clipboardchange'

Returns:

* `event` Event
  * `types` string[] An array of MIME types that are available in the clipboard.

The clipboardchange event fires whenever the contents of the system clipboard are changed.

On Linux, instead of passing a type of `selection` to access the selection clipboard, the following
APIS will be added to work with the selection clipboard:
* `clipboard.selection.clear()` _Linux_
  * Clears the selection clipboard.  Replaces `clipboard.clear('selection')`.
* `clipboard.selection.has(format)` _Linux_
  * Check if a particular format is available on the selection clipboard.  Replaces `clipboard.has(format,'selection)`.
* `clipboard.selection.read()` _Linux_
  * Reads from the selection clipboard following the [Web API clipboard.read](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read) spec.  Replaces `clipboard.read('selection')` and `clipboard.readBuffer('selection)`.
  * Returns a Promise that resolves with an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing the 
    clipboard's contents.
  * Custom MIME types e.g. `electron application/filepath` will be used to allow support of non-standard clipboard formats.  This follows 
    the W3C proposal for supporting [Web Custom formats for Async Clipboard API](https://github.com/w3c/editing/blob/gh-pages/docs/clipboard-pickling/explainer.md#custom-formats).
    The exception here is that instead of using the `web` prefix, we will use the `electron` prefix to prevent possible collisions with custom web formats.
  * OS specific raw formats will be handled using the custom MIME type `electron application/osclipboard` with a parameter `format` containing the OS specific format, eg `electron application/osclipboard;format="CF_TEXT"` would represent the `CF_TEXT` format on Windows.
* `clipboard.selection.readText()` _Linux_
  * Read text from the selection clipboard. Replaces `clipboard.readText('selection)`.
* `clipboard.selection.write(data)​`
  * `data` an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing data to be written to the clipboard.
  * Writes to the selection clipboard following the [Web API `clipboard.write`](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write).  Replaces `clipboard.write(data, 'selection)` and `clipboard.writeBuffer(format, buffer, 'selection')`.
  * `data` an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing data to be written to the clipboard.
  * This API will be modified to bring into spec compliance with the [Web API `clipboard.write`](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write).
  * Returns a Promise which is resolved when the data has been written to the clipboard. The promise is rejected if the clipboard is unable to complete the clipboard access.
  * Custom MIME types e.g. `electron application/filepath` will be used to allow support of non-standard clipboard formats.  This follows 
    the W3C proposal for supporting [Web Custom formats for Async Clipboard API](https://github.com/w3c/editing/blob/gh-pages/docs/clipboard-pickling/explainer.md#custom-formats).
    The exception here is that instead of using the `web` prefix, we will use the `electron` prefix to prevent possible collisions with custom web formats.
  * OS specific raw formats will be handled using the custom MIME type `electron application/osclipboard` with a parameter `format` containing the OS specific format, eg `electron application/osclipboard;format="CF_TEXT"` would represent the `CF_TEXT` format on Windows.  
* `clipboard.selection.writeText(text)` _Linux_
  * Write text to the selection clipboard.  Replaces `clipboard.writeText(text, 'selection)`.

**APIs to Remove**
* `clipboard.availableFormats([type])`
  * Superseded by `clipboard.read` [Web API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read)
* `clipboard.readBookmark()`
  * Superseded by `clipboard.read` with custom `electron/bookmark` MIME type
* `clipboard.writeBookmark(title, url[, type])`
  * Superseded by `clipboard.write` with custom `electron/bookmark` MIME type
* `clipboard.readBuffer(format)`
  * Superseded by `clipboard.read`[Web API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read) with a custom MIME type `electron application/osclipboard` with a parameter `format`, eg `electron application/osclipboard;format="CF_TEXT"` would represent the `CF_TEXT` format on Windows.
* `clipboard.writeBuffer(format, buffer[, type])`
  * Superseded by `clipboard.write` [Web API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write) with a custom MIME type `electron application/osclipboard` with a parameter `format`, eg `electron application/osclipboard;format="CF_TEXT"` would represent the `CF_TEXT` format on Windows. 
* `clipboard.readFindText()` _macOS_
  * Superseded by `clipboard.read` with custom `electron/findtext` MIME type
* `clipboard.writeFindText(text)`
  * Superseded by `clipboard.write` with custom `electron/findtext` MIME type
* `clipboard.readHTML()`
  * Superseded by `clipboard.read` with `text/html` MIME type
* `clipboard.writeHTML(markup[, type])`
  * Superseded by `clipboard.write` with `text/html` MIME type
* `clipboard.readImage([type])`
  * Superseded by `clipboard.read()` with `image/*` (where * is bmp, gif, jpeg, png, svg+xml, etc.)
* `clipboard.writeImage(image[, type])`
  * Superseded by `clipboard.write`  with `image/*` (where * is bmp, gif, jpeg, png, svg+xml, etc.)
* `clipboard.readRTF([type])`
  * Superseded by `clipboard.read` with `application/rtf` MIME type
* `clipboard.writeRTF(text[, type])`
  * Superseded by `clipboard.write` with `application/rtf` MIME type

**APIs to Modify**
* `clipboard.clear()`
  * This is not supported through existing Web APIs or in the specification, so keep and modify
    to drop type parameter in favor of `clipboard.selection.clear()` when clearing the selection
    clipboard on Linux.
* `clipboard.readText()`
  * Already implemented per specification. Modify to drop type parameter in favor of
    `clipboard.selection.readText()` when reading text from the selection clipboard on Linux.
* `clipboard.writeText(text)`
  * Already implemented per specification. Modify to drop type parameter in favor of
    `clipboard.selection.writeText(text)` when writing text from the selection clipboard on Linux.
* `clipboard.has(format)`
  * This is not supported through existing Web APIs or in the specification, so keep and modify
    to drop type parameter in favor of `clipboard.selection.has(format)` to check the selection
    clipboard on Linux.

* `clipboard.read()`
  * This API will be modified to bring into spec compliance with the [Web API clipboard.read](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read)
  * Returns a Promise that resolves with an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing the 
    clipboard's contents.
  * Custom MIME types e.g. `electron application/filepath` will be used to allow support of non-standard clipboard formats.  This follows 
    the W3C proposal for supporting [Web Custom formats for Async Clipboard API](https://github.com/w3c/editing/blob/gh-pages/docs/clipboard-pickling/explainer.md#custom-formats).
    The exception here is that instead of using the `web` prefix, we will use the `electron` prefix to prevent possible collisions with custom web formats.
  * OS specific raw formats will be handled using the custom MIME type `electron application/osclipboard` with a parameter `format` containing the OS specific format, eg `electron application/osclipboard;format="CF_TEXT"` would represent the `CF_TEXT` format on Windows.
* `clipboard.write(data)​`
  * `data` an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing data to be written to the clipboard.
  * This API will be modified to bring into spec compliance with the [Web API `clipboard.write`](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write).
  * Returns a Promise which is resolved when the data has been written to the clipboard. The promise is rejected if the clipboard is unable to complete the clipboard access.
  * Custom MIME types e.g. `electron application/filepath` will be used to allow support of non-standard clipboard formats.  This follows 
    the W3C proposal for supporting [Web Custom formats for Async Clipboard API](https://github.com/w3c/editing/blob/gh-pages/docs/clipboard-pickling/explainer.md#custom-formats).
    The exception here is that instead of using the `web` prefix, we will use the `electron` prefix to prevent possible collisions with custom web formats.
  * OS specific raw formats will be handled using the custom MIME type `electron application/osclipboard` with a parameter `format` containing the OS specific format, eg `electron application/osclipboard;format="CF_TEXT"` would represent the `CF_TEXT` format on Windows.

**Removing the Clipboard API from the renderer**

Currently, the Electron clipboard API is available from non-sandboxed renderer processes.  This currently represents a security risk that should be closed.
Additionally, given the refactor of the Electron clipboard API to match the W3C implementation it would be confusing to have two APIs available in the renderer with the same API surface but two different implementations.  The Electron clipboard API will still be able via the [contextBridge API](https://www.electronjs.org/docs/latest/api/context-bridge).

  ### API Migration Table

  | Old API | New API |
  |---------|---------|
  | `clipboard.availableFormats([type])` | `clipboard.read()` - iterate through ClipboardItem array and collect types |
  | `clipboard.clear()` | `clipboard.clear()` (no type parameter) |
  | `clipboard.clear('selection')` _Linux_ | `clipboard.selection.clear()` |
  | `clipboard.has(format)` | `clipboard.has(format)` (no type parameter) |
  | `clipboard.has(format, 'selection')` _Linux_ | `clipboard.selection.has(format)` |
  | `clipboard.read()` | `clipboard.read()` (returns Promise with ClipboardItem array) |
  | `clipboard.read('selection')` _Linux_ | `clipboard.selection.read()` (returns Promise with ClipboardItem array) |
  | `clipboard.readBookmark()` | `clipboard.read()` with `electron application/bookmark` MIME type |
  | `clipboard.readBuffer(format)` | `clipboard.read()` with `electron application/osclipboard;format="..."` MIME type |
  | `clipboard.readBuffer('selection')` _Linux_ | `clipboard.selection.read()` with `electron application/osclipboard;format="..."` MIME type |
  | `clipboard.readFindText()` _macOS_ | `clipboard.read()` with `electron application/findtext` MIME type |
  | `clipboard.readHTML([type])` | `clipboard.read()` with `text/html` MIME type |
  | `clipboard.readImage([type])` | `clipboard.read()` with `image/*` MIME type |
  | `clipboard.readRTF([type])` | `clipboard.read()` with `application/rtf` MIME type |
  | `clipboard.readText()` | `clipboard.readText()` (no type parameter) |
  | `clipboard.readText('selection')` _Linux_ | `clipboard.selection.readText()` |
  | `clipboard.write(data)` | `clipboard.write(data)` (accepts ClipboardItem array, returns Promise) |
  | `clipboard.write(data, 'selection')` _Linux_ | `clipboard.selection.write(data)` (accepts ClipboardItem array, returns Promise) |
  | `clipboard.writeBookmark(title, url[, type])` | `clipboard.write()` with `electron application/bookmark` MIME type |
  | `clipboard.writeBuffer(format, buffer[, type])` | `clipboard.write()` with `electron application/osclipboard;format="..."` MIME type |
  | `clipboard.writeBuffer(format, buffer, 'selection')` _Linux_ | `clipboard.selection.write()` with `electron application/osclipboard;format="..."` MIME type |
  | `clipboard.writeFindText(text)` _macOS_ | `clipboard.write()` with `electron application/findtext` MIME type |
  | `clipboard.writeHTML(markup[, type])` | `clipboard.write()` with `text/html` MIME type |
  | `clipboard.writeImage(image[, type])` | `clipboard.write()` with `image/*` MIME type |
  | `clipboard.writeRTF(text[, type])` | `clipboard.write()` with `application/rtf` MIME type |
  | `clipboard.writeText(text)` | `clipboard.writeText(text)` (no type parameter) |
  | `clipboard.writeText(text, 'selection')` _Linux_ | `clipboard.selection.writeText(text)` |

## Reference-level explanation

### Example migration code
```js
const { serialize, deserialize } = require("v8")

function getClipboardToUse(clipboardType) {  
  if (clipboardType == 'selection') {
    return clipboard.selection;
  } else {
    return clipboard;
  }
}

async function readClipboard(format, clipboardType) {
  const clipboardToUse = getClipboardToUse(clipboardType);
  const clipboardItems = clipboardToUse.read();
  const foundItem = clipboardItems.find(clipboardItem => {
    return clipboardItem.types.includes(format);
  });
  if (foundItem) {
    const buffer = await findTextItem.getType(format);
    return buffer.toString();
  }
}

async function writeClipboard(format, text, clipboardType) {
  const clipboardToUse = getClipboardToUse(clipboardType);
  return clipboardToUse.write([
    {
      [format]: Buffer.from(text)
    }
  ]);
}

async function readBuffer(format, clipboardType) {
  const clipboardToUse = getClipboardToUse(clipboardType);
  const clipboardItems = await clipboardToUse.read();
  const foundItem = clipboardItems.find(clipboardItem => {
    return clipboardItem.types.includes(format);
  });
  if (foundItem) {
    const buffer = foundItem.getType(format);
    return buffer;
  }
}

async function writeBuffer(format, buffer, clipboardType) {
  const clipboardToUse = getClipboardToUse(clipboardType);
  return clipboardToUse.write([
    {
      [format]: buffer
    }
  ]);
}

async function availableFormats(clipboardType) {
  const clipboardToUse = getClipboardToUse(clipboardType);
  const clipboardItems = await clipboardToUse.read();
  const clipboardFormats = [];  
  for (const clipboardItem of clipboardItems) {
    for (const type of clipboardItem.types) {
      if (!clipboardFormats.includes(type)) {
        clipboardFormats.push(type);
      }
    }
  }
  return clipboardFormats;
}

async function has(format, clipboardType) {
  const clipboardToUse = getClipboardToUse(clipboardType);
  return clipboardToUse.has(format);
}

const BOOKMARK_MIME_TYPE = 'electron application/bookmark';
async function readBookmark() {
  const bookmarkItem = await readBuffer(BOOKMARK_MIME_TYPE);
  if (bookmarkItem) {
    return deserialize(bookmarkItem);
  }
}

async function writeBookmark(title, url) {
  const buffer = serialize({
    text: title,
    bookmark: url    
  });
  return writeBuffer(BOOKMARK_MIME_TYPE, buffer);
}

const FILE_PATH_MIME_TYPE = 'electron application/filepath';
async function readFilePaths() {
  const filePathsItem = await readBuffer(FILE_PATH_MIME_TYPE);
  if (filePathsItem) {
    return deserialize(filePathsItem);
  }
}

async function writeFilePaths(paths) {
  const filePathsBuffer = serialize(paths);
  return writeBuffer(BOOKMARK_MIME_TYPE, filePathsBuffer);
}

const FIND_TEXT_MIME_TYPE = 'electron application/findtext';
async function readFindText() {
  return readClipboard(FIND_TEXT_MIME_TYPE);
}

async function writeFindText(text) {
  return writeClipboard(FIND_TEXT_MIME_TYPE, text);
}

const HTML_MIME_TYPE = 'text/html';
async function readHTML(clipboardType) {
  return readClipboard(HTML_MIME_TYPE, clipboardType);
}

async function writeHTML(markup, clipboardType) {
  return writeClipboard(HTML_MIME_TYPE, markup, clipboardType);
}

const PNG_MIME_TYPE = 'image/png';
const JPEG_MIME_TYPE = 'image/jpeg';
async function readImage(clipboardType)​ {
  const clipboardToUse = getClipboardToUse(clipboardType);
  const clipboardItems = await clipboardToUse.read();
  //Look for PNG first  
  let foundItem = clipboardItems.find(clipboardItem => {
    return clipboardItem.types.includes(PNG_MIME_TYPE);
  });
  if (!foundItem) {
    foundItem = clipboardItems.find(clipboardItem => {
      return clipboardItem.types.includes(JPEG_MIME_TYPE);
    });
  }
  if (foundItem) {
    let buffer;
    if (foundItem.types.includes(PNG_MIME_TYPE)) {      
      buffer = foundItem.getType(PNG_MIME_TYPE);
    } else {
      buffer = foundItem.getType(JPEG_MIME_TYPE);
    }    
    return nativeImage.createFromBuffer(buffer);
  }
}

async function writeImage(image, clipboardType)​ {
  const clipboardToUse = getClipboardToUse(clipboardType);
  const buffer = image.getBitmap();
  const dataUrl = image.toDataURL();
  const regex = /^data:(.+\/.+);.*$/;
  const matches = dataUrl.match(regex);
  return clipboardToUse.write([
    {
      [matches[1]]: buffer
    }
  ]);
}

const RTF_MIME_TYPE = 'application/rtf';
async function readRTF(clipboardType) {
  return readClipboard(RTF_MIME_TYPE, clipboardType);
}

async function writeRTF(text, clipboardType)​ {
  return writeClipboard(RTF_MIME_TYPE, text, clipboardType);
}
```

## Drawbacks

This is a major refactor of an API with a pretty large surface area; however there are parts
of the current [`clipboard`](https://www.electronjs.org/docs/latest/api/clipboard) module that 
are still marked as `Experimental`.

## Rationale and alternatives

1. Strict alignment with W3C specification - no additional APIs exposed.
2. Leave API as-is and continue to adapt it as required by upstream changes by Chromium development.

## Prior art

- [feat(clipboard): support read/write files PR](https://github.com/electron/electron/pull/47975)
- [feat: add clipboard.writeFilePaths/readFilePaths APIs PR](https://github.com/electron/electron/pull/26693)
