# Mac Versioned Bundle Layout

- Start Date: 2026-08-05
- RFC PR: [electron/rfcs#0000](https://github.com/electron/rfcs/pull/0000)
- Status: **Proposed**

## Summary

Proposal to update Electron's MacOS Bundle layout to include versioned
directories for the `Electron Framework`, Helper plugins, and app resources.

## Motivation

<!-- Why should we do this? What use cases does it support? What is the expected outcome? -->

In enterprise application deployments, it's possible for update scripts or
installers to overwrite the contents of an already running application. In such
cases, an attempt to spawn Electron's child process helpers may result in a
catastrophic action which crashes the application due to ABI incompatibilities.

This is a problem unique to Electron due to its Mac bundle layout. Its binaries
and resources are included under the following directories without any versioning:
- `Electron.app/Contents/Frameworks/Electron Framework.framework/Versions/A`
- `Electron.app/Contents/Frameworks/Electron Helper*.app`
- `Electron.app/Contents/Resources/*.asar`

In such scenarios, apps utilizing ASAR integrity will also risk a crash when
spawning utilityProcess scripts in their app.asar.

### Electron vs Chrome

<details>
<summary>Electron.app bundle layout</summary>

```
Electron.app/
  └── Contents/
      ├── Info.plist
      ├── PkgInfo
      ├── MacOS/
      │   └── Electron                          # main executable
      ├── Resources/
      │   ├── electron.icns
      │   ├── default_app.asar
      │   └── *.lproj/                          # ~60 locale folders
      └── Frameworks/
          ├── Electron Framework.framework/
          │   ├── Electron Framework  → Versions/Current/Electron Framework   (symlink)
          │   ├── Helpers             → Versions/Current/Helpers              (symlink)
          │   ├── Libraries           → Versions/Current/Libraries           (symlink)
          │   ├── Resources           → Versions/Current/Resources           (symlink)
          │   └── Versions/
          │       ├── Current  → A                                           (symlink)
          │       └── A/
          │           ├── Electron Framework                     # the ~191 MB dylib
          │           ├── Helpers/
          │           │   └── chrome_crashpad_handler
          │           ├── Libraries/
          │           │   ├── libEGL.dylib
          │           │   ├── libGLESv2.dylib
          │           │   ├── libffmpeg.dylib
          │           │   ├── libvk_swiftshader.dylib
          │           │   └── vk_swiftshader_icd.json
          │           └── Resources/
          │               ├── Info.plist
          │               ├── MainMenu.nib
          │               └── *.lproj/          # locale.pak per language
          ├── Electron Helper.app
          ├── Electron Helper (GPU).app
          ├── Electron Helper (Plugin).app
          ├── Electron Helper (Renderer).app
          ├── Squirrel.framework                # (also versioned: Versions/A + Current)
          ├── Mantle.framework                  # (versioned)
          └── ReactiveObjC.framework            # (versioned)
```

</details>

For comparison, Google Chrome uses a versioned layout where the version
directory is named after the actual version number, multiple versions are
retained side-by-side, and *all* helper apps live inside the versioned
framework directory:

<details>
<summary>Google Chrome.app bundle layout</summary>

```
Google Chrome.app/
  └── Contents/
      ├── Info.plist
      ├── PkgInfo
      ├── CodeResources
      ├── embedded.provisionprofile
      ├── _CodeSignature/
      ├── MacOS/                                  # main executable
      ├── Library/
      ├── Resources/                              # icons, *.lproj, etc.
      └── Frameworks/
          └── Google Chrome Framework.framework/
              ├── Google Chrome Framework  → Versions/Current/Google Chrome Framework  (symlink)
              ├── Default Apps             → Versions/Current/Default Apps             (symlink)
              ├── Helpers                  → Versions/Current/Helpers                  (symlink)
              ├── Libraries                → Versions/Current/Libraries                (symlink)
              ├── Resources                → Versions/Current/Resources                (symlink)
              └── Versions/
                  ├── Current  → 150.0.7871.212                                        (symlink)
                  ├── 150.0.7871.187/          # previous version, kept for rollback
                  └── 150.0.7871.212/          # active version
                      ├── Google Chrome Framework          # the ~490 MB dylib
                      ├── _CodeSignature/
                      ├── Default Apps/
                      ├── Helpers/
                      │   ├── Google Chrome Helper.app
                      │   ├── Google Chrome Helper (GPU).app
                      │   ├── Google Chrome Helper (Renderer).app
                      │   ├── Google Chrome Helper (Alerts).app
                      │   ├── GoogleUpdater.app
                      │   ├── app_mode_loader
                      │   ├── chrome_crashpad_handler
                      │   └── web_app_shortcut_copier
                      ├── Libraries/
                      │   ├── libEGL.dylib
                      │   ├── libGLESv2.dylib
                      │   ├── liboptimization_guide_internal.dylib
                      │   ├── libvk_swiftshader.dylib
                      │   ├── vk_swiftshader_icd.json
                      │   ├── WidevineCdm/
                      │   ├── MEIPreload/
                      │   ├── IwaKeyDistribution/
                      │   └── PrivacySandboxAttestationsPreloaded/
                      └── Resources/
                          └── *.lproj/           # locale.pak per language
```

</details>

## Guide-level explanation

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

## Reference-level explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- Any new dependencies on Chromium code are outlined.

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work.

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is an API proposal, could this be done as a JavaScript module or a native Node.js add-on
  instead? Does the proposed change make Electron code easier or harder to read, understand,
  and maintain?

## Prior art

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
