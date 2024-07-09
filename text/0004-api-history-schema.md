- Start Date: 2024-06-11
- RFC PR: [electron/rfcs#6](https://github.com/electron/rfcs/pull/6)
- Electron Issues: Related to
[electron/electron#39877](https://github.com/electron/electron/issues/39877)
- Reference Implementation:
[electron/lint-roller (feat/api-history-schema-rfc)](https://github.com/electron/lint-roller/commit/d88115ad9ef0df7966976220adb7846f4338caf3)
- Status: **Proposed**

# API History Schema

## Summary

> [!NOTE]
> This RFC is part of the **2024 Electron GSoC**
> [[Project](https://summerofcode.withgoogle.com/programs/2024/projects/wenKR2i3)]
> [[Proposal](https://gist.github.com/piotrpdev/9ed9bd7f0f3ab192a5ae58a35fbe03e7)].
>
> This RFC is **just for the schema**, not the API History feature as a whole.

This proposal aims to introduce a
[JSON Schema](https://json-schema.org/overview/what-is-jsonschema)
for (YAML) API history blocks in Electron documentation.
It will be used to enforce and validate the structure and content of these blocks,
which will include historical information about changes to the various APIs in Electron.

## Motivation

The API history blocks need to be written in a well-designed format in order to
convey information efficiently and ensure readability. These blocks will also be
transformed into tables in `electron/website` to allow users to see what/when/how
Electron APIs have changed. In order to do so, the blocks need to have a consistent
and parsable structure.

## Guide-level explanation

Before this RFC, when a developer created a new feature they added documentation
for it like this:

`electron/docs/api/browser-window.md`

```markdown
#### `win.setTrafficLightPosition(position)` _macOS_

* `position` [Point](structures/point.md)

Set a custom position for the traffic light buttons. Can only be used with `titleBarStyle` set to `hidden`.
```

After this RFC, they will consult the schema, and also add an API history block:

````markdown
#### `win.setTrafficLightPosition(position)` _macOS_

<!--
```YAML history
added:
  - pr-url: https://github.com/electron/electron/pull/22533
```
-->

* `position` [Point](structures/point.md)

Set a custom position for the traffic light buttons. Can only be used with
`titleBarStyle` set to `hidden`.
````

Which, after several PRs, may reach this state:

```yaml
added:
  - pr-url: https://github.com/electron/electron/pull/22533
changes:
  - pr-url: https://github.com/electron/electron/pull/26789
    description: Made `trafficLightPosition` option work for `customButtonOnHover` window.
deprecated:
  - pr-url: https://github.com/electron/electron/pull/37094
    breaking-changes-header: deprecated-browserwindowsettrafficlightpositionposition
```

> [!NOTE]
>
> - *"Why no version numbers?"*
>   - They will be derived from the PRs. This removes the need to change API History
> on backports.
>
> - *"Why no `removed` property?"*
>   - When an API is removed from Electron, it is also removed from the documentation.
>
> For further details, see [**Rationale and alternatives**](#rationale-and-alternatives).

Meanwhile, the Electron website displays this information nicely in a table.
Users can then use this information to help them upgrade their Electron version,
migrate from deprecated APIs, etc.

The schema provides information to the developer about what to include in the
block, the minimum/maximum length of text, the required properties, etc.

Documentation for older APIs will also eventually be changed to include these
API history blocks.

## Reference-level explanation

The JSON Schema looks like this:

```json
{
  "title": "JSON schema for API history blocks in Electron documentation",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$comment": "If you change this schema, remember to edit the TypeScript interfaces in the linting script.",
  "definitions": {
    "baseChangeSchema": {
      "type": "object",
      "properties": {
        "pr-url": { "type": "string", "pattern": "^https://github.com/electron/electron/pull/\\d+$" },
        "breaking-changes-header": { "type": "string", "minLength": 3 },
        "description": { "type": "string", "minLength": 3, "maxLength": 72 }
      },
      "required": [ "pr-url" ],
      "additionalProperties": false
    },
    "addedChangeSchema": {
      "allOf": [ { "$ref": "#/definitions/baseChangeSchema" } ]
    },
    "deprecatedChangeSchema": {
      "$comment": "TODO: Make 'breaking-changes-header' required in the future.",
      "allOf": [ { "$ref": "#/definitions/baseChangeSchema" } ]
    },
    "changesChangeSchema": {
      "allOf": [ { "$ref": "#/definitions/baseChangeSchema" }, { "required": [ "description" ] } ]
    }
  },
  "type": "object",
  "properties": {
    "added": { "type": "array", "items": { "$ref": "#/definitions/addedChangeSchema" } },
    "deprecated": { "type": "array", "items": { "$ref": "#/definitions/deprecatedChangeSchema" } },
    "changes": { "type": "array", "items": { "$ref": "#/definitions/changesChangeSchema" } }
  },
  "additionalProperties": false
}
```

The developer will include information about when/where/how/why an API was:

- Added
- Changed
- Deprecated

Along with this, they will include a link to the PR where this change was made
and preferably include a short description of the change. If applicable, they
will also include the header for that change from the
[breaking changes documentation.](https://github.com/electron/electron/blob/main/docs/breaking-changes.md)

The description is limited to a short length for efficiency, if the user wants
to know more they can look at the included PR.

## Drawbacks

- Adhering to a strict schema limits flexibility. Maybe the developer wants to
include additional properties for a change.

## Rationale and alternatives

- I believe this schema is simple and features everything the user/developer
would need.
- The website will use the PR URLs to determine and display Electron version
numbers for the changes. This is more efficient than including the version
numbers in the Markdown docs themselves. It also removes the need to manually
update the docs when a backport is made.
  > "...There’s one somewhat significant change we’d like to call out about
  the proposal, which came up during discussion when we were reviewing proposals.
  [...] [we] decided that the approach with the least drawbacks would be to only
  use PR URLs (the root PRs to main) instead of hardcoded version strings as in
  the proposal. This can serve as a single source of truth which can then be used
  to derive exact version numbers, and no further documentation changes on main
  are necessary if the change is backported to other branches."
  — @dsanders11 via Slack
- The Node.js API history [includes a `removed` property](https://github.com/nodejs/node/blob/0db95d371274104a5acf09214bf8325c45bfb64a/doc/api/errors.md?plain=1#L3432-L3442),
  however that will not be included in this schema because:
  - When an API is removed from Electron, it is also removed from the documentation.
    - This currently cannot be changed because [the documentation is used to generate
    TypeScript definitions](https://github.com/electron/docs-parser/blob/d95a2a2e58d6dafce1247d4bc820ca60516fe10c/README.md?plain=1#L36-L43)
    for the Electron API.
- Using [Zod](https://zod.dev/) validation logic may be simpler. It's also easy
to understand for a developer even if they've never worked with it before:

  ```ts
  const changeSchema = z.object({
    'pr-url': z.string().url().startsWith('https://github.com/electron/electron/pull/'),
    'breaking-changes-header': z.string().min(3).optional(),
    description: z.string().min(3).max(72).optional(),
  });

  const historySchema = z.object({
    added: z.array(changeSchema).optional(),
    deprecated: z.array(changeSchema).optional(),
    changes: z.array(changeSchema).optional(),
  });
  ```

  - > *"Why Zod specifically?"*
    - First-class TypeScript support and [static type inference](https://zod.dev/?id=type-inference).
    - [Most starred](https://byby.dev/js-object-validators) JavaScript object
    validation library.
    - [Zero dependencies](https://www.npmjs.com/package/zod?activeTab=dependencies).
    - Liked by developers
    [[1]](https://www.reddit.com/r/reactjs/comments/z739b9/yup_vs_zod_what_do_you_prefer_and_why/)
    [[2]](https://medium.com/@weidagang/zod-schema-validation-made-easy-195f86d82d44)
    [[3]](https://rasitcolakel.medium.com/exploring-zod-a-comprehensive-guide-to-powerful-data-validation-in-javascript-typescript-2c4818b5646d).
    - Liked in comparisons:
    [[1]](https://zod.dev/?id=comparison)
    [[2]](https://blog.logrocket.com/comparing-schema-validation-libraries-zod-vs-yup/#zod-vs-yup)
    [[3]](https://www.bitovi.com/blog/comparing-schema-validation-libraries-ajv-joi-yup-and-zod).

## Prior art

I couldn't find an example of something exactly like this being implemented
elsewhere.

[Semantic Versioning](https://semver.org/) and
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) are
tangentially related to this concept and both *suggest* a consistent format
for information about changes but not necessarily *enforce* it.

Of course, plenty of documentation includes versioning.

I think enforcing a schema on API change documentation will massively improve
the experience of a user upgrading major versions of Electron in their project.
The process should ideally be as effortless as possible.

## Unresolved questions

- What should be allowed in the descriptions?
  - Should full Markdown be supported?
    - Keep in mind that the API history blocks should be easy to read after
    being processed by the website *and* in raw Markdown form.

- Does the developer or user need any other features in the schema?
  - Is there any other information that should be included by the developer?
  - Would the user benefit from additional information?

- Should another property be included specifically for migration instructions?
  - Is it enough to include that information in the description?
  - Maybe add a property to include a link to a new API after one is deprecated.

- Should a property be added to specify the platform for a change?
  - Maybe the developer should be able to specify a change was only made to
  the Linux implementation of a feature.
    - Maybe this can be inferred from the PR tags instead?

## Future possibilities

- Once the RFC has been implemented for a while, a script could be made that
users can run that would display all of the places in their code that need to
be changed because of breaking changes, deprecations, etc.

- Maybe a more precise type of change could be specified.
  - Maybe the developer or user could benefit from seeing/filtering changes
  based on security fixes, bug fixes, etc.
    - Maybe this can be inferred from the PR tags instead?
