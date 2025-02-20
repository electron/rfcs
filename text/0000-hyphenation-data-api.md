# RFC Template

- Start Date: 2025-02-20
- RFC PR: [electron/rfcs#0000](https://github.com/electron/rfcs/pull/0000)
- Electron Issues: https://github.com/electron/electron/issues/33692
- Reference Implementation: 
- Status: **Draft**

# Hyphenation data support

## Summary

Chromium uses a folder called `hyphen-data` downloaded after installation to enable automatic hyphenation of words in various 
languages. This RFC is to make the same hyphenation feature available to Electron apps.

## Motivation

One of the promises of Electron is that it renders the same as Chromium. Without hyphenation, this is not the case for 
languages like German or Dutch. As a result, text in these languages is not hyphenated across different text lines, 
causing the text to have different dimensions, break at different places and generally look worse than in Chromium.

Adding support for hyphenation data will bring the rendering of long words across text lines in line with the rendering 
in Chromium

## Guide-level explanation

We propose an API for hyphenation data similar to the existing spellChecker API in Electron. The reason for this is that spell checking data in Chromium is similarly downloaded on launch (rather than shipping with the renderer) and Electron replicates this behavior through an API.

### Enabling hyphenation

To enable hyphenation in your Electron app, you can use the following code:

```javascript
const myWindow = new BrowserWindow({
  webPreferences: {
    hyphenation: true
  }
})
```
_We should consider making this behavior the default, as the spellChecker API is also enabled by default_

This will enable hyphenation for the languages that are supported by the hyphenation data file, which is automatically downloaded.

### Use of Google services

Like the spellChecker API, hyphen-data files are downloaded from a Google CDN. If you want to avoid this you can provide an alternative URL to download the hyphen-data from: 
```javascript
myWindow.webContents.session.setHyphenationDataDownloadURL('https://example.com/hyphen-data/')
```

### Consideration: list supported languages

The spellcheck API has a method to list the supported languages. We should consider adding a similar method to list the supported languages for hyphenation, if that is feasible.

```js
// an array of all available languages for hyphenation
const possibleLanguages = myWindow.webContents.session.availableHyphenationLanguages
```
 
## Reference-level explanation
<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- Any new dependencies on Chromium code are outlined. 

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work.

-->

Hyphenation data is downloaded using the [ComponentUpdater](https://source.chromium.org/chromium/chromium/src/+/main:components/component_updater/README.md) in Chromium and used in `ContentBrowserClient::GetHyphenationDictionary()`

In Electron, `GetHyphenationDictionary()` can be implemented on `ElectronBrowserClient`

## Drawbacks

This adds a new API to Electron, which increases the complexity of the API surface. 

It also adds a new Google CDN download to Electron, which could be seen as a privacy concern.

## Rationale and alternatives

Conceptually, the hyphenation data is similar to the spellChecker data, which is already available in Electron. Keeping the APIs similar will make it easier for developers to use. 

An alternative would be to ship the hyphenation data with Electron and read it from disk, but this would increase the size of the Electron download and would not be in line with the way Chromium handles hyphenation data.

Not adding hyphenation data means that people reading text in languages that make greater use of hyphenation, like German or Dutch, will have a worse experience in Electron than in Chromium.

Because the hyphen-data needs to be read in by the renderer, it is not feasible to implement this in userland.

## Prior art

This feature brings Electron in line with Chromium.

## Unresolved questions

- What parts of Electron will need to be modified to support this feature?
- Do we need to patch Chromium to make this work?
- Should the API be enabled by default?
- Should we provide a way to list the supported languages for hyphenation?

## Future possibilities

The primary goal is to bring Electron in line with Chromium. 