# RFCs for Electron

In Electron, the Request for Comments (RFC) process is designed to provide a consistent and
controlled path for “substantial” proposals to be incorporated into future releases of Electron.

Many changes to Electron can be directly introduced via pull requests to the
`electron/electron` repository. However, some changes need additional process to ensure that
stakeholders can achieve consensus on their benefits for the present and future of Electron.

Such “substantial” changes should be introduced via the RFC process.

**The RFC process is open to anyone in the community.**

## When to use this process

The RFC process is intended to be used to introduce “substantial” changes to the core Electron
project. The definition of a “substantial” change is subjective and evolves with project norms,
but would include:

- a new feature that requires a large amount of patches or new Chromium dependencies
- a new module in Electron’s API surface
- or a refactor of Electron’s core startup code

However, more straightforward changes can be instead addressed directly with a PR. This type of
change includes:

- updates to documentation
- objective quality improvements (performance, platform support, better error handling, etc.)
- refactors invisible to end-users
- fixes for objectively incorrect behavior

If you submit a pull request to implement a new feature without going through the RFC process,
it may be closed with a polite request to submit an RFC first.

## Stages

An RFC can exist in four different stages:

- **Proposed:** An RFC is Proposed if it has an open pull request with an API spec. At this point,
  community feedback can be gathered to drive consensus on the spec. Maintainers can approve and
  merge into the `rfcs` repo to move the RFC to the Active stage or close the pull request to mark
  it as Declined.
- **Active:** An RFC is Active once it is merged into the `electron/rfcs` repository. An
  implementation PR(s) can be opened in `electron/electron` as a technical reference.
  **An Active RFC means that the idea is worth being implemented and explored, not that the
  maintainers are committing to adding it to Electron.** If a new development causes an Active
  RFC to become unnecessary in any way, maintainers can choose to ultimately decline it.
- **Completed:** Once the implementation for an Active RFC is merged into Electron’s `main` branch
  and is slated for an upcoming release, it is marked as Completed with a PR to `electron/rfcs`
  containing the status change and a link to the `electron/electron` commit and target Electron release.
- **Declined:** A Proposed RFC can be marked as Declined by maintainers after public discussion. When
  a proposal is declined, a maintainer should add a comment summarizing the reasons for the decision
  and close out the pull request.

## Process

### Adding a new RFC proposal

- **Fork** the `electron/rfcs` repo.
- **Copy the template** markdown file (`0000-template.md`) into the `text` folder, and rename it
  after your feature (e.g. `text/0000-my-cool-electron-feature.md`).
- **Fill out the RFC template.** Put care into the details: RFCs that do not present convincing
  motivation, demonstrate understanding of the impact of the design, or are disingenuous about the
  drawbacks or alternatives tend to be poorly-received.
- **Open a pull request.** Once the pull request is open, the RFC is considered Proposed and is
  open to general feedback. RFC authors are expected to engage with this feedback to arrive at a
  consensus with community stakeholders.
- Eventually, Electron’s API WG will decide if the RFC is a candidate for inclusion in Electron.
- This will trigger a one-month final comment period for the RFC. If enough consensus is achieved,
  the RFC will be merged and marked as Active.

### Working on an Active RFC

- Once the RFC is marked as Active, the author can open an implementation PR on `electron/electron`
  and attach it to the RFC document in this repository.
- If a tracking issue does not exist on `electron/electron`, it will also be created and attached
  to the RFC document.
- Modifications to active RFCs can be done in followup PRs. We strive to write each RFC in a manner
  that it will reflect the final design of the feature; but the nature of the process means that we
  cannot expect every merged RFC to actually reflect what the end result will be at the time of the
  next major release; therefore we try to keep each RFC document somewhat in sync with the language
  feature as planned, tracking such changes via followup pull requests to the document.
- If the implementation PR gets merged into `electron/electron`, the RFC will be marked as Completed.

### Details on Active RFCs

- An Active RFC is not a guarantee that the feature will be approved and merged. Rather, it means that
  Electron’s maintainer team has agreed to it in principle and is amenable to merging it.
- Furthermore, the fact that a given RFC has been accepted and is 'active' implies nothing about what
  priority is assigned to its implementation, nor whether anybody is currently working on it.

## RFC review

**Feedback on Proposed RFCs is open to the community**, although Electron’s API working group will make
the ultimate decision regarding their accepting or declining the proposed RFC. Depending on the nature of the proposal,
additional working groups in Electron Governance may be tagged to review the RFC as well.

The API WG cannot commit to a timeline for reviewing all RFC proposals, but will try to read each
submission within a few weeks of submission.

## Community vs maintainer RFCs

Electron's RFC process is designed to open up substantial change contributions to Electron to all
developers in the community.

However, Electron maintainers often design and build features internally outside of the
RFC process. In this case, RFCs from maintainers may be published once feature experimentation is
already done and consensus for the feature is established among maintainers. They serve primarily as
a period of community feedback for things that the core team already intends to ship.

We apply the same level of rigour both to maintainer RFCs and RFCs from the community. The primary
difference between them is in the design phase: maintainer RFCs tend to be submitted at the end of
the design process whereas community RFCs tend to be submitted at the beginning as a way to
kickstart it.

## Credits

Electron's RFC process was modeled on many established open source RFC processes. Inspiration for many ideas
and bits of copywriting go to:

- [emberjs/rfcs](https://github.com/ember/rfcs)
- [reactjs/rfcs](https://github.com/reactjs/rfcs)
- [rust-lang/rfcs](https://github.com/rust-lang/rfcs)
- [tauri-apps/rfcs](https://github.com/tauri-apps/rfcs)
- [vuejs/rfcs](https://github.com/vuejs/rfcs)
- [yarnpkg/rfcs](https://github.com/yarnpkg/rfcs)
