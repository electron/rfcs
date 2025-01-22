# RFC Template

- Start Date: 2024-11-26
- RFC PR: [electron/rfcs#0012](https://github.com/electron/rfcs/pull/12)
- Electron Issues: None <!-- [electron/electron#0000](https://github.com/electron/electron/issue/0000) -->
- Reference Implementation: None (yet) <!-- [electron/electron#0000](https://github.com/electron/electron/pull/0000) -->
- Status: **Proposed**

# Corner Smoothing

![There is a black rectangle on the left using simple rounded corners, and a blue rectangle on the right using smooth rounded corners. Inbetween those rectangles is a magnified view of the same corner from both rectangles overlapping to show the subtle difference in shape.](../images/0012/Summary.svg)

## Summary

Applying `border-radius` to an element makes its corners circular. This proposal adds "smoother" round corners, controlled by a prefixed CSS property. Applications can use this feature to further integrate with macOS's design language, or simply for aesthetic purposes.

```css
.my-button {
  border-radius: 48px;
  -electron-corner-smoothing: 60%;
}
```

## Motivation

A simple strategy for making round corners is to replace each corner with a quarter circle. This strategy, dubbed "simple rounding" for this proposal, is [used by the Web for `border-radius`](https://www.w3.org/TR/css-backgrounds-3/#border-radius). More rounding strategies exist to satisfy additional visual and mathematical design constraints.

Apple is well-known for having smooth corners in their hardware and software product designs. Apple's macOS is one of Electron's major supported platforms, and round corners are a core part of its design language, used across UI elements such as buttons and windows. The round corners used in macOS are a different shape than those created by the Web's `border-radius` rule, featuring corners that are smoother than in simple rounding.

Integrating with the operating system and its design language is important to many desktop applications. For example, Chromium emulates [macOS's scrollbar design](https://source.chromium.org/chromium/chromium/src/+/main:ui/views/controls/scrollbar/cocoa_scroll_bar.h?q=symbol%3A%5Cbviews%3A%3ACocoaScrollBar%5Cb%20case%3Ayes). Like scroll bars, the shape of a rounded corner can be a subtle detail to many users. However, aligning closely to the system's design language that users are familiar with makes the application's design feel familiar too.

Beyond the macOS platform, designers may decide to use smoother round corners for many reasons. The popular Web-based design tool [Figma](https://www.figma.com/) provides a "Corner Smoothing" control on design elements for creating designs with smooth corners.

## Guide-level explanation

The corners of elements rounded by the CSS `border-radius` rule can be smoothed out using the `-electron-corner-smoothing` feature.

<table>
<tr>
<th>Code</th>
<th>Result</th>
</tr>
<tr>
<td>
<pre lang="css">
.box {
  width: 128px;
  height: 128px;
  border-radius: 24px;
  -electron-corner-smoothing: 100%;
}
</pre>
</td>
<td>
<img src="../images/0012/Rectangle.svg" width="128" alt="" />
</td>
</tr>
</table>

The `-electron-corner-smoothing` CSS rule is **only implemented for Electron** and has no effect in browsers. Avoid using this rule outside of Electron.

The `-electron-corner-smoothing` CSS rule controls how far the corner's curvature can extend into the edge of the element. It is a percentage between `0%` (default) and `100%`. `-electron-corner-smoothing` affects the shape of borders, outlines, and shadows on the target element.

To match the system rounding in macOS, use a value of `60%`. `-electron-corner-smoothing` is available across all Electron platforms.

![A chart demonstrating values of the CSS rule and their visual effects on the same rectangle. The values of 0%, 30%, 60%, and 100% are demonstrated, with the 60% value being labeled as macOS.](../images/0012/Values.svg)

This smooth rounding is similar to Apple's "continuous" rounded corners in SwiftUI as well as Figma's "corner smoothing" control on design elements.

## Reference-level explanation

Integration with macOS's design language is a primary goal for this proposal. Enabling design customization is a secondary goal.

To implement corner smoothing we will patch Blink with two additions: a CSS property and support for painting smooth corners.

### CSS Property

CSS properties are [declared in a large JSON list](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/css/css_properties.json5). We will create a new CSS rule by adding a new entry to this list and configuring its properties appropriately. Doing so automatically generates C++ classes and glue code at compile time.

The `-electron-` vendor prefix was added to the CSS rule to [satsify some concerns](https://www.w3.org/TR/css-2023/#proprietary). It clearly identifies this rule as an Electron-specific extension to developers, removing the expectation that other platforms will also implement this extension.

### Painting

The painting procedures in Blink are defined across multiple "Painter" classes that use the [`GraphicsContext` class](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/graphics/graphics_context.h) to render graphics. The API is very similar to the Web's [2D canvas rendering API](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D).

We will add new logic to these Painter classes to identify elements with smooth corners, then use a new procedure to paint them.

#### Smooth Corner Construction

The procedure for smoothing the roundness of a corner is based on the research in the article [*Desparately seeking squircles* by Daniel Furse, published on the Figma blog](https://www.figma.com/blog/desperately-seeking-squircles/).

At a high level, we're taking the points where the edges connect to the corner circle and stretching them back into the edge. Thus, instead of constructing a round corner with a quarter circle, we use two cubic Bézier curves to construct the corner.

![A chart showing two constructions of a rounded corner. The first construction shows a quarter circle connected to two straight edges, with the points of connection highlighted. Arrows by those points indicate a stretching motion twoard the straight edge. The second construction shows the same corner with the connection points stretched backward, demonstrating the Bézier construction of a rounded corner.](../images/0012/Construction.svg)

The two Bézier curves' control points are carefully chosen to retain the most important qualities of a rounded corner, including the percieved radius. The original article provides a full intuition for how the curve's control points are derived.

This method provides some creative control in choosing how far along the edge the corner's curve can extend into.

### Interactions & Corner Cases

<!-- Haha, "corner" cases. -->

#### Simple Rounding vs Smooth Rounding

Applying smoothing to an existing round corner should retain as much of the original shape's important points and properties as possible. The smoothing procedure chosen satisfies a few important mathematical constraints:

* At the points where the box's edge meets the curve:
  * The curvature is zero, creating a seamless transition from edge to corner.
* At the point where the two curves meet:
  * The curvature is equal, continuing the "roundness" of the corner seamlessly.
  * The position is the same as the "middle" point in simple rounding.

#### Accessory Shapes (Borders, Outlines, Shadows)

More than just the corners of the element itself, `border-radius` also applies to other shapes, including borders, outlines, and shadows. These other shapes are derived from the element's shape. `-electron-corner-smoothing` also applies to these same shapes to maintain consistency.

## Drawbacks

Electron rarely makes changes to Blink. The few exceptions include exposing the `<webview>` element, exposing the `-webkit-app-region` CSS rule, and increasing the threshold for Web Storage APIs. With this feature, we would maintain a medium-sized patch for a feature *addition*.

Testing this feature will require reliable visual testing and a comprehensive test suite. Writing and running these tests may require more resources than other Electron features.

## Rationale and alternatives

Implementing this feature within Blink uniquely offers the best performance and simplest developer experience in this problem space. Other potential solutions and avenues for development suffer from technical complexity, performance pitfalls, or aim to solve larger and more complex problems than this proposal is targeting.

### Alternative: CSS Paint Worklet + Masking

It may be possible to implement this feature within the Web platform:

* The [CSS Painting API](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Painting_API) allows developers to write flexible custom painting code that produces CSS image values.
* The [`mask-image` CSS property](https://developer.mozilla.org/en-US/docs/Web/CSS/mask-image) uses an image as a masking layer for an element. This property can accept CSS image values.
  * The [`mask-border` CSS property](https://developer.mozilla.org/en-US/docs/Web/CSS/mask-border) (currently available as the [`-webkit-mask-box-image` property](https://developer.mozilla.org/en-US/docs/Web/CSS/-webkit-mask-box-image)), in conjunction with [border slicing](https://developer.mozilla.org/en-US/docs/Web/CSS/mask-border-slice), could be used instead of `mask-image` to reduce the size of the bitmaps in the paint worklet.

It's not yet known what the drawbacks of this approach are. There may be problems with border shaping and performance pitfalls.

### Alternative: Web Standards

There are existing proposals in the same or similar problem spaces for some CSS standards:

* [CSS Borders Level 4 Issue](https://github.com/w3c/csswg-drafts/issues/10653)
* [CSS Backgrounds Level 4 Issue](https://github.com/w3c/csswg-drafts/issues/6296)
* [CSS Shapes Level 2 Issue](https://github.com/w3c/csswg-drafts/issues/10993)

Development on these standards moves slowly and their scopes extend far beyond just smooth round corners. These standards may never reach completion, or may not support smooth round corners in their completed form. Their chosen solutions for smooth round corners may not be compatible with macOS's design language, an important use case of this feature for our developers, as their priorities are not exactly the same as ours.

To be clear, this feature is not intended to be standardized as it is designed now. Instead, the intent is to satisfy one very specific need of developers at present. If we were to pursue the standardization route, we would have to fudamentally rethink our appraoch.

## Prior art

### Smooth Rounded Corners

* [Figma: Corner smoothing for sqiurcles](https://help.figma.com/hc/en-us/articles/360050986854-Adjust-corner-radius-and-smoothing#h_01HE150KTYDMPMNZ86K0GSTV68)
  * [Figma blog: *Desparately seeking squircles* by Daniel Furse](https://www.figma.com/blog/desperately-seeking-squircles/)
* [SwiftUI: `RoundedCornerStyle.continuous`](https://developer.apple.com/documentation/swiftui/roundedcornerstyle/continuous)
* CSS Specification Drafts
  * [CSS Borders Level 4 Issue](https://github.com/w3c/csswg-drafts/issues/10653)
  * [CSS Backgrounds Level 4 Issue](https://github.com/w3c/csswg-drafts/issues/6296)
  * [CSS Shapes Level 2 Issue](https://github.com/w3c/csswg-drafts/issues/10993)

### CSS

Electron has dabbled in having its "own" CSS rules before. `-webkit-app-region` is a CSS rule that allows developers to define what regions in their application can drag the window. While implemented by Chromium before Electron existed, most developers know of the CSS rule because of Electron and have only used it here. This CSS rule has been [proposed for standardization](https://github.com/w3c/csswg-drafts/issues/7017) as part of the proposed Window Controls Overlay feature.

## Unresolved questions

### Resolve Through RFC Process

- Is fingerprinting a major concern of this feature?
- Should this feature be gated by a `WebContents` option?

### Resolve Through Implementation

- Should patching be done on the different "Painter" classes or only the `GraphicsContext` class?
- Testing strategy

## Future possibilities

### Other Shapes

There may be desire from developers and designers for other shapes with smooth round corners, like the superellipse. 

### Standardization

A future Web standard may one day provide an equivalent feature. In that event, we should provide developers with a clear migration path to the standardized feature and deprecate this one.