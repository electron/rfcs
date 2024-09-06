# RFC Template

- Start Date: 2024-09-06
- RFC PR: [electron/rfcs#0000](https://github.com/electron/rfcs/pull/0000)
- Electron Issues: [electron/electron#0000](https://github.com/electron/electron/issue/0000)
- Reference Implementation: [electron/electron#0000](https://github.com/electron/electron/pull/0000)
- Status: **Proposed**

# Corner Smoothing

## Summary

Add a corner smoothing primitive to Blink's paint procedures, along with a CSS property to control this new behavior.

## Motivation

Integrating with the operating system and its design language is an important aspect of desktop application design.

macOS is well known for its use of smooth round corners. macOS desktop applications that use similar smooth corners in their shapes, even when they deviate from the system controls' designs in other ways, are more harmonious with the system.

## Guide-level explanation

New CSS property: `-electron-border-radius-smoothing`

```css
.round-rect {
  border-radius: 32px;
  -electron-border-radius-smoothing: 1.0;
}
```

<!--
Explain the feature as if it were already implemented in Electron and you were teaching it to
an Electron app developer.

This section should:

- Introduce new named concepts.
- Show concrete examples of how the feature is used.
- Explain how the feature will impact existing use cases of Electron.
- If applicable, describe the migration path from an older set of Electron features or APIs.
- Discuss how this impacts the ability to read, understand, and maintain Electron code. Will the
  proposed feature make Electron code more maintainable? How difficult is the upgrade path for
  existing apps?

When writing this section, make sure to clearly account for API differences or considerations for
Windows, macOS, and Linux.
-->

## Reference-level explanation

Blink is the 'Web implementation' in Chromium, also used in Electron. To effectively implement this feature we have to modify some of Blink's internals—specifically the CSS rules definitions, the Paint procedures, and the glue between these two areas of the codebase.

<!-- TODO: diagram -->

<!-- TODO: smoothing method -->

Prototype: https://github.com/electron/electron/tree/clavin/smooth-corner-rounding

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- Any new dependencies on Chromium code are outlined.

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work.
-->

#### 🧠 Brainstorm: ⛈️
- Medium complexity patch
- Blink paint phase modified to understand smoothing
- CSS property addition
  - Plumbing property to Blink paint phase

## Drawbacks

Electron tends to avoid adding features to the Web implemenation, such as the one proposed here.

<!-- Why should we *not* do this? -->

## Rationale and alternatives

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is an API proposal, could this be done as a JavaScript module or a native Node.js add-on
  instead? Does the proposed change make Electron code easier or harder to read, understand,
  and maintain?
-->

#### 🧠 Brainstorm: ⛈️
- Alternative: CSS Houdini APIs + image masking
  - Horrifically inefficient and noisy code changes
- Alternative: Web standards proposal
  - Less applicable use case
- Add a WebContents option controlling this feature, default off
  - Potentially avoid fingerprinting

## Prior art

<!--
Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what
this can include are:

- Does this feature exist in other frameworks and what experience have their community had?
- Does this feature exist as a userland implementation, and what can be learned from it?
- Is this related to a change upstream in Chromium or Node.js?
- Does this proposal help Electron further align with evolving web standards?

This section is intended to encourage you as an author to think about the lessons from prior
implementations to provide readers of your RFC with a fuller picture. If there is no prior art,
that is fine - your ideas are interesting to us whether they are brand new or if it is an
adaptation from other technologies.
-->

#### 🧠 Brainstorm: ⛈️
- Figma corner smoothing blog post
- iOS & macOS design
- `-webkit-app-region` in Electron's early days

## Unresolved questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature
  before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the
  future independently of the solution that comes out of this RFC?
-->

#### 🧠 Brainstorm: ⛈️
- Out of scope: Committing to a specific rounding strategy or algorithm

## Future possibilities

<!--
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
-->

#### 🧠 Brainstorm: ⛈️
- Web standardization