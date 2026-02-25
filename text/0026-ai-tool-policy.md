# AI Tool Policy

- Start Date: 2025-12-17
- RFC PR: [electron/rfcs#26](https://github.com/electron/rfcs/pull/26)
- Status: **Proposed**

> [!NOTE]
> RFC meta-text is first, then [proposed policy text](#proposed-ai-tool-policy) follows.

## Summary

This RFC proposes an official policy on AI-assisted contributions to Electron. The goals are to:

1. Set clear expectations for contributors using AI tools
2. Give maintainers something to reference when closing low-quality submissions
3. Encourage transparency and good practices
4. Protect maintainer time and energy

## Motivation

AI coding assistants have become widespread. While these tools can genuinely help contributors be more productive, they also make it trivially easy to generate (a lot of) superficially plausible content without understanding it.

As a result, maintainers across the open source ecosystem are seeing an increase in low-effort submissions: pull requests that don't compile, issues that misunderstand the project, and comments that confidently add nothing. This creates an unfair burden on the scarce resource of maintainer time. Our project is no exception.

### Prior art

This policy heavily draws inspiration from:

- [(WIP) LLVM AI Tool Policy RFC](https://discourse.llvm.org/t/rfc-llvm-ai-tool-policy-human-in-the-loop/89159) — especially the "human in the loop" framing and "extractive contributions" philosophy
- [Chromium AI Coding Policy](https://chromium.googlesource.com/chromium/src/+/main/agents/ai_policy.md) — strong notion of accountability
- [Nextest's `AGENTS.md` "for humans" section](https://github.com/nextest-rs/nextest/blob/main/AGENTS.md) — "quality multiplier" framing
- [Fedora Policy on AI-Assisted Contributions](https://docs.fedoraproject.org/en-US/council/policy/ai-contribution-policy/) — encouragement of AI for accessibility/translation
- [OpenInfra Policy for AI Generated Content](https://openinfra.org/legal/ai-policy) — `Assisted-By:` / `Generated-By:` metadata

Electron has explicitly discouraged PRs created with AI in repos that see a high volume of (would-be) policy violations: https://github.com/electron/website/pull/784

## Rationale

These standards of quality, effort, and accountability have always applied to contributions; we're making them explicit now. Therefore, this policy technically isn't exclusively about AI; it's about low-effort contributions. AI is specifically emphasized because it's a disproportionally sore spot right now.

## Execution

If the proposed policy is accepted, we will:

1. Add the policy to the [`policy/` directory of the Governance repo](https://github.com/electron/governance/tree/main/policy).
2. Link to the policy in our [contributing guidelines](https://github.com/electron/electron/blob/main/CONTRIBUTING.md).
3. Link to the policy when we enforce it.
4. Refactor existing PR template guidance to refer to this policy.

This policy supersedes any repository-specific AI policies previously established. Individual repositories may adopt stricter stances where necessary, but the baseline expectations here apply project-wide.

---

# Proposed: AI Tool Policy

Electron welcomes contributions crafted with many different tools: IDEs, linters, Web community resources, and so on. LLMs—and AI tools built on them—are becoming more prevalent in contributors' toolkits. Like any tool, what matters is not *how* you wrote something, but whether **you understand it, can defend it, and take responsibility for it**.

These standards of quality, effort, and accountability have always applied to contributions; we're making them explicit now because AI tools make it trivially easy to generate superficially plausible content without understanding it.

Our core principle is simple: **there must be a human in the loop**.

Contributors **must** review, understand, and be able to explain all content they submit, regardless of how it was produced. If you use AI tools to help write code, documentation, comments, or any other contribution, **you are still the author and bear full responsibility for the work**.

This means:

* You have **read and reviewed** content before submitting it
* You can **answer questions** about your work during review
* You have **built and tested** code contributions

Submitting unreviewed AI-generated content wastes scarce maintainer time and is unacceptable behavior. Contributions that lack thoughtfulness and care may be declined outright.

## What's allowed

* Drafting code or documentation that you then review, understand, and edit in-depth
* AI-assisted language utilities: spelling & grammar checkers, translators
* Authorized bots and agents owned by the Electron project (e.g. trop, roller, CI integrations, merged agent workflows)

## What's not allowed

> [!NOTE]
> *The list below is illustrative, not exhaustive. Read the rest of this policy.*

* Submitting AI-generated content you haven't reviewed or don't understand
* Posting AI-generated comments directly to issues or PRs without meaningful human input
* Unauthorized, automated agents that take action without human input (e.g. bots that post comments or open PRs autonomously)
* Automated agents that post subjective commentary or code review feedback without human review
* Passing maintainer feedback to an LLM and posting the response as your own reply

## Handling violations

Violations of this policy will be handled according to our [Code of Conduct](https://github.com/electron/electron/blob/main/CODE_OF_CONDUCT.md) [enforcement guidelines](https://github.com/electron/electron/blob/main/CODE_OF_CONDUCT.md#enforcement-guidelines). In short: we use a mix of private correction notes, public warnings, and temporary/permanent bans to enforce this policy.

Maintainers may close pull requests, close issues, and hide comments that appear to violate this policy without further review, citing a link to this policy. Contributors are welcome to ask clarifying questions, revise, and resubmit once they can demonstrate understanding of their contribution.

## Scope

This policy applies to, but is not limited to, the following kind of contributions:

* Code (pull requests; PRs)
* Issues, bug reports
* Comments, discussions
* Code reviews, feedback
* Documentation
* RFCs, proposals

The Electron project as a whole is covered by this policy, including (but not limited to) the main `electron/electron` repository as well as other project repositories: `electron/forge`, `electron/website`, etc.

## Conventions

### Disclosure in code contributions

We encourage disclosure of AI tool assistance in code contributions. This practice helps facilitate productive code reviews, and is not used to police tool usage.

When AI tools meaningfully assist in your contribution, note it in the commit message with a [trailer](https://git-scm.com/docs/git-interpret-trailers):

```plaintext
Assisted-By: Claude Opus 4.5, Claude Code

#  also acceptable:

Co-Authored-By: Claude <noreply@anthropic.com>
Generated-By: GitHub Copilot
```

Include the models used and any additional wrappers around them. At minimum, the "brand" of model or user-facing tool used is required when writing a trailer.

Disclosure is mandatory when AI-generated code is accepted largely as-written. That is, you reviewed it and found it acceptable without significant restructuring or rewriting. We recommend the `Co-Authored-By:` trailer in these situations because it is featured more prominently on GitHub and other tools.

Trailer conventions may evolve as the broader open-source community converges on standards; we'll update this guidance accordingly.

### Philosophy for contributors

We encourage contributors to treat AI tools as a **quality multiplier**, not just a speed multiplier. Reinvest time savings in:

* Tackling the tedious work that often gets skipped
* Writing better tests, covering more edge cases
* Refactoring for clarity

### Guidance for new contributors

We aspire to be a welcoming community that helps new contributors grow. Our guidance: **start small**. Submit contributions you can fully understand, get feedback, and iterate.

As maintainers, we want *your* contributions, not the tools’ outputs. Learning involves taking small steps; passing maintainer feedback directly to an LLM doesn't help anyone grow, and doesn't sustain our community.

---

## Implementation

Once this proposal is accepted, the following steps will bring the policy into practice:

1. **Commit the policy to the Governance repo** — `policy/ai.md` will be the canonical source for the policy text and any future updates.

2. **Update the PR template in `electron/electron`** — Add an acknowledgment checkbox for contributors to confirm they've read and are following this policy. Include an HTML comment in the template that instructs AI models not to check the acknowledgment box autonomously.

3. **Link the policy from `CONTRIBUTING.md`** — Make the policy discoverable to new contributors by referencing it alongside existing contribution guidelines.

4. **Announce the policy** — Post a brief notice so the entire team is aware the policy is in effect.
