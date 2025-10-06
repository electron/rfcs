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

**APIs to Preserve**
* `clipboard.clear([type])`
  * Not supported through existing Web APIs or in the specification.
* `clipboard.readText([type])`
  * Already implemented per specification.
* `clipboard.writeText(text[, type])`
  * Already implemented per specification.
* `clipboard.has(format[, type])` 
  * Convenience method to quickly check if a particular format is available.

**APIs to Remove**
* `clipboard.availableFormats([type])`
  * Superseded by `clipboard.read` [Web API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read)
* `clipboard.readBookmark()`
  * Superseded by `clipboard.read` with custom `electron/bookmark` MIME type
* `clipboard.writeBookmark(title, url[, type])`
  * Superseded by `clipboard.write` with custom `electron/bookmark` MIME type
* `clipboard.readBuffer(format)`
  * Superseded by `clipboard.read`[Web API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read)
* `clipboard.writeBuffer(format, buffer[, type])`
  * Superseded by `clipboard.write` [Web API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write)
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
* `clipboard.read([clipboardType])`
  * `clipboardType` string (optional) - Can be `selection` or `clipboard`; default is `clipboard`. `selection` is only available on Linux.
  * This API will be modified to bring into spec compliance with the [Web API clipboard.read](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read)
  * Returns a Promise that resolves with an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing the 
    clipboard's contents. The promise is rejected if permission to access the clipboard is not granted.
  * Ensure that raw formats are preserved
  * Custom MIME types e.g. `electron application/filepath` will be used to allow support of non-standard clipboard formats.  This follows 
    the W3C proposal for supporting [Web Custom formats for Async Clipboard API](https://github.com/w3c/editing/blob/gh-pages/docs/clipboard-pickling/explainer.md#custom-formats).
    The exception here is that instead of using the `web` prefix, we will use the `electron` prefix to prevent possible collisions with custom web formats.
* `clipboard.write(data[, clipboardType]])​`
  * `data` an array of [ClipboardItem](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem) objects containing data to be written to the clipboard.
  * `clipboardType` string (optional) - Can be `selection` or `clipboard`; default is `clipboard`. `selection` is only available on Linux.
  * This API will be modified to bring into spec compliance with the [Web API `clipboard.write`](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write).
  * Returns a Promise which is resolved when the data has been written to the clipboard. The promise is rejected if the clipboard is unable to complete the clipboard access.
  * Ensure that raw formats are preserved
  * Custom MIME types e.g. `electron application/filepath` will be used to allow support of non-standard clipboard formats.  This follows 
    the W3C proposal for supporting [Web Custom formats for Async Clipboard API](https://github.com/w3c/editing/blob/gh-pages/docs/clipboard-pickling/explainer.md#custom-formats).
    The exception here is that instead of using the `web` prefix, we will use the `electron` prefix to prevent possible collisions with custom web formats.
  * clipboardType is only used for Linux to specify if clipboard is regular clipboard or `selection` clipboard.

## Reference-level explanation

### Example migration code
```js
const { serialize, deserialize } = require("v8")

async function readClipboard(format, clipboardType) {
  const clipboardItems = await clipboard.read(clipboardType);  
  const foundItem = clipboardItems.find(clipboardItem => {
    return clipboardItem.types.includes(format);
  });
  if (foundItem) {
    const buffer = await findTextItem.getType(format);
    return buffer.toString();
  }
}

async function writeClipboard(format, text, clipboardType) {
  return clipboard.write([
    {
      [format]: Buffer.from(text)
    }
  ], clipboardType);
}

async function readBuffer(format, clipboardType) {
  const clipboardItems = await clipboard.read(clipboardType);
  const foundItem = clipboardItems.find(clipboardItem => {
    return clipboardItem.types.includes(format);
  });
  if (foundItem) {
    const buffer = foundItem.getType(format);
    return buffer;
  }
}

async function writeBuffer(format, buffer, clipboardType) {
  return clipboard.write([
    {
      [format]: buffer
    }
  ], clipboardType);
}

async function availableFormats(clipboardType) {
  const clipboardItems = await clipboard.read(clipboardType);
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
  const clipboardItems = await clipboard.read(clipboardType);
  const matchingClipboardItem = clipboardItems.find(clipboardItem => {
    return clipboardItem.types.includes(format);
  });
  if (matchingClipboardItem) {
    return true;
  }
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
  const clipboardItems = await clipboard.read(clipboardType);
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
  const buffer = image.getBitmap();
  const dataUrl = image.toDataURL();
  const regex = /^data:(.+\/.+);.*$/;
  const matches = dataUrl.match(regex);
  return clipboard.write([
    {
      [matches[1]]: buffer
    }
  ], clipboardType);
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

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature
  before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the
  future independently of the solution that comes out of this RFC?

## Future possibilities

Think about what the natural extension and evolution of your proposal would be and how it would
affect the project as a whole in a holistic way. Try to use this section as a tool to more fully
consider all possible interactions with the project in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but
otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you
cannot think of anything.

Note that having something written down in the future possibilities section is not a reason to
accept the current or a future RFC; such notes should be in the section on motivation or
rationale in this or subsequent RFCs. The section merely provides additional information.