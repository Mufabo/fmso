---
title: "How to Contribute to Open Source Projects, Using O3DE as an Example"
date: 2026-11-08
draft: true
description: "A practical guide to contributing to open source projects, walked through with the Open 3D Engine (O3DE): finding where help is needed, the fork-and-pull-request workflow, DCO sign-off, Special Interest Groups, RFCs, code review, and non-code contributions like documentation."
summary: "Contributing to a big open source project like the Open 3D Engine looks intimidating from the outside, but the process is mostly the same everywhere: find the right people, pick a small issue, fork, branch, sign off your commits, open a pull request and respond to review. Here is how that works in practice, with O3DE as the worked example."
tags: ["open source", "o3de", "github", "git", "gamedev", "game engines", "contributing"]
categories: ["Programming"]
---

In my [look at open source game engines in 2026](/posts/20261108_state-of-open-source-game-engines-2026/) I called the Open 3D Engine (O3DE) "the heavyweight": a large, modular C++ engine descended from Amazon Lumberyard and now run by the Open 3D Foundation under the Linux Foundation. That size is exactly why it's a good case study for contributing to open source. If you can find your way around a project this big, smaller ones will feel easy.

This post covers the general approach first, then walks through how it works for O3DE specifically.

## The general shape of an open source contribution

Almost every project on GitHub or GitLab follows the same basic loop:

1. **Read the rules.** Look for `CONTRIBUTING.md`, a code of conduct, and any licensing requirements (a CLA or DCO) in the repository root.
2. **Find the people.** Big projects split into teams or working groups. Find the one that owns the area you care about and where they talk (Discord, Matrix, a forum, mailing list, regular meetings).
3. **Pick something small.** Look for labels like `good first issue`, `help wanted` or `documentation`. Your first contribution is about learning the process, not about impressing anyone.
4. **Claim it.** Comment on the issue so nobody duplicates your work, and so maintainers can tell you early if your plan is wrong.
5. **Fork, branch, change, commit.** Work on a branch in your own fork, following the project's coding standards.
6. **Open a pull request.** Describe what you changed and why, and link the issue.
7. **Respond to review.** Expect change requests. This is normal and is how you learn the codebase.
8. **Get merged, then do it again.** Trust in open source is built over several contributions, not one.

The single biggest mistake newcomers make is skipping steps 2 and 4: writing a large feature in isolation and then dropping it on maintainers unannounced. Review time is the scarcest resource in open source. Godot made that explicit this year with a policy that requires maintainer approval before new contributors submit features or significant refactors[^1]. Even where it isn't a written rule, talking first is always the faster route.

## Contributing to O3DE

### 1. Start from the official guides

O3DE keeps its process documentation in two places:

- The **Contributor Guide** on the docs site, which explains how the project is organised and links to sections on contributing code, documentation and security, plus roadmaps and the code of conduct[^2].
- A **`CONTRIBUTING.md`** in each GitHub repository. The project says contribution guidelines for the engine and for each Special Interest Group live in the root of their repositories[^3].

The [o3de.org/contribute](https://o3de.org/contribute/) page also links lists of good first issues for both docs and code[^4], which is the easiest place to find a starting point.

### 2. Understand the SIGs

O3DE is too large for one team to own, so it is divided into **Special Interest Groups (SIGs)**. Each SIG plans, reviews and maintains its own area of the engine[^3]. The current SIGs, each with its own GitHub repository, are[^5]:

| SIG | Area |
| --- | --- |
| [sig-build](https://github.com/o3de/sig-build) | Build system, CMake, CI |
| [sig-content](https://github.com/o3de/sig-content) | Editor, content creation tools, scripting |
| [sig-core](https://github.com/o3de/sig-core) | Core engine frameworks |
| [sig-docs-community](https://github.com/o3de/sig-docs-community) | Documentation and community |
| [sig-graphics-audio](https://github.com/o3de/sig-graphics-audio) | Atom renderer and audio |
| [sig-network](https://github.com/o3de/sig-network) | Networking and multiplayer |
| [sig-platform](https://github.com/o3de/sig-platform) | Platform support |
| [sig-release](https://github.com/o3de/sig-release) | Release management |
| [sig-security](https://github.com/o3de/sig-security) | Security |
| [sig-simulation](https://github.com/o3de/sig-simulation) | Physics, simulation, robotics |
| [sig-testing](https://github.com/o3de/sig-testing) | Test infrastructure |
| [sig-ui-ux](https://github.com/o3de/sig-ui-ux) | Editor UI and UX |

The project-wide hub is [o3de/community](https://github.com/o3de/community), which describes the development process and points newcomers to the code of conduct, values and contributing guide before reaching out to the SIG for the area they want to work on[^6].

Why this matters for you: the SIG that owns the code you touch is the one that reviews and merges your PR. O3DE uses GitHub **CODEOWNERS** to map directories to SIGs, so the right reviewers are added to a pull request automatically[^7]. If you're unsure where something belongs, ask in that SIG's repository or its channel on the O3DE Discord.

### 3. Big changes go through an RFC

For anything larger than a bug fix, O3DE uses **RFCs (Requests for Comments)**: a proposal filed as an issue in the relevant SIG repository, discussed in public, and accepted or rejected before the work is merged. You can see real examples in the SIG repositories, such as the sig-testing RFC that introduced the CODEOWNERS file[^7] or proposals on the sig-docs-community repo[^8].

You don't need an RFC for your first contribution. You do need one before spending weeks on a new feature.

### 4. Set up the code

O3DE's main repository is [github.com/o3de/o3de](https://github.com/o3de/o3de), and work is merged into the **`development`** branch. The documented workflow is the standard fork-based one[^9]:

```bash
# 1. Fork o3de/o3de on GitHub, then clone your fork
git clone https://github.com/<you>/o3de.git
cd o3de

# 2. Track the official repository as "upstream"
git remote add upstream https://github.com/o3de/o3de.git
git fetch upstream

# 3. Start your branch from the latest upstream development
git checkout -b fix/my-issue upstream/development
```

Before you change anything, make sure you can build and run the engine from source. The docs cover [setting up O3DE from GitHub](https://docs.o3de.org/docs/welcome-guide/setup/setup-from-github/), with separate guides for Windows and Linux. Be warned that this is a large C++ project: check the [system requirements](https://docs.o3de.org/docs/welcome-guide/requirements/) and expect the first build to take a while.

Read the [O3DE C++ coding standards](https://docs.o3de.org/docs/contributing/to-code/) before writing code. The code contribution section also covers the code review guidelines, deprecation guidelines and the project's validation tool[^10].

### 5. Sign off every commit (DCO)

O3DE requires a **Developer Certificate of Origin (DCO)** sign-off on every commit. The sign-off is your statement that the contribution is your own work, or that you otherwise have the right to submit it, and PRs without it won't be reviewed or merged[^11].

In practice it's one flag:

```bash
git commit -s -m "Fix crash when deleting an entity with a pending prefab override"
```

This appends a line like:

```text
Signed-off-by: Your Name <you@example.com>
```

The name and email must match your Git configuration. If you forget:

```bash
# Fix the last commit
git commit --amend -s

# Fix the last N commits
git rebase --signoff HEAD~N

git push --force-with-lease
```

One easy-to-miss trap: if a reviewer suggests a change and you apply it with the "Commit suggestion" button in GitHub's PR interface, that commit isn't signed off. O3DE's docs tell you to paste your `Signed-off-by:` line into the commit message field when you do this[^12]. You'll see commits titled "DCO Remediation Commit" in the O3DE history; that's someone fixing exactly this problem after the fact[^13].

### 6. Keep up to date and open the pull request

If your branch has been around for a while, bring in the latest upstream changes before opening the PR. The O3DE workflow shows merging `upstream/development` into your branch with `--signoff`, so even the merge commit carries a sign-off[^9]:

```bash
git fetch upstream
git merge upstream/development --signoff
git push origin fix/my-issue
```

Then, on GitHub, open a pull request from your fork's branch to **`o3de/o3de:development`**. Give it a clear title, a short description of the scope of the change, link the issue it fixes, and explain how you tested it[^9].

### 7. Review, Automated Review and merge

Two things then happen:

- **Human review** by the owning SIG. Reviewers may request changes; push further (signed-off) commits to the same branch to address them[^9].
- **Automated Review (AR)**, O3DE's CI gate that builds and tests pull requests before they can merge into shared branches like `development`[^14]. A red AR run is something you need to look into, though occasionally it's a flaky test rather than your change; ask on the PR if in doubt.

Once the PR is approved and AR passes, a SIG maintainer (or a delegate) merges it into `development`[^9]. Contributors don't merge their own pull requests.

## You don't have to write C++

Some of the most valuable contributions to O3DE aren't engine code at all.

**Documentation.** The docs live in a separate repository, [o3de/o3de.org](https://github.com/o3de/o3de.org), and are plain Markdown processed by Hugo, which the project says requires no prior web development experience[^15]. The process mirrors the code one: every contribution starts with a GitHub issue, you work on a fork, sign off your commits, and open a PR against `o3de.org:main`[^15][^11]. Documentation PRs need two reviewers, and deeply technical content also needs a review from an O3DE code contributor for accuracy[^11]. Follow the [docs style guide](https://docs.o3de.org/docs/contributing/to-docs/style-guide/) and you'll get through review faster.

If you understand how a part of the engine works internally, the **Engine Developer Guide** specifically asks for help documenting architecture that is too broad for code comments[^16].

**Other ways to help:**

- **Bug reports** with clear reproduction steps, engine version, platform and logs.
- **Triage**: reproducing existing issues and adding missing details.
- **Testing** release candidates and reporting regressions.
- **Answering questions** on Discord or GitHub discussions.
- **Community work**: the project actively asks for people interested in writing, speaking, organising meetups and developer advocacy[^4].
- **Security**: there's a separate [security contribution guide](https://docs.o3de.org/docs/contributing/security/). Don't report vulnerabilities in public issues.

## Tips that apply to any project

- **Read merged PRs before writing yours.** They show you what reviewers expect better than any guide.
- **Small PRs get reviewed. Large ones wait.** One logical change per pull request.
- **Write tests.** A fix with a test is much easier for a maintainer to accept.
- **Be patient, then polite.** If a PR sits for a couple of weeks, a friendly ping on the PR or in the SIG's channel is fine.
- **Disclose AI assistance if the project asks for it.** Projects increasingly have rules on AI-generated code; Godot's 2026 policy is the most visible example[^1]. Whatever tools you use, you need to understand and be able to defend every line you submit.
- **Show up to the meetings.** O3DE SIGs hold public meetings; listening in is the fastest way to learn what's actually being worked on.

## A first-contribution checklist for O3DE

- [ ] Read the [Contributor Guide](https://docs.o3de.org/docs/contributing/) and the [code of conduct](https://docs.o3de.org/docs/contributing/code-of-conduct/)
- [ ] Join the O3DE Discord and find the SIG for your area
- [ ] Pick a good first issue from [o3de.org/contribute](https://o3de.org/contribute/) and comment to claim it
- [ ] Fork the repo and add `upstream`
- [ ] Build O3DE from source and reproduce the issue
- [ ] Branch from `upstream/development` (or `main` for the docs repo)
- [ ] Make the change, following the coding or docs style guide
- [ ] Commit with `git commit -s`
- [ ] Open a PR to `o3de/o3de:development`, link the issue
- [ ] Respond to review, keep every commit signed off, and wait for AR to pass

## References

[^1]: [Changes to our Contribution Policies](https://godotengine.org/article/contribution-policy-2026/), Godot Foundation, 30 June 2026.
[^2]: [Contributor Guide for Open 3D Engine](https://docs.o3de.org/docs/contributing/), O3DE documentation.
[^3]: [Contribute](https://o3de.org/contribute/), Open 3D Engine: contribution guidelines and Special Interest Groups.
[^4]: [Get involved with O3DE](https://o3de.org/?p=467), Open 3D Engine: good first issues and community advocacy.
[^5]: [O3DE Community repository](https://awesome.ecosyste.ms/projects/github.com%2Fo3de%2Fcommunity), via Ecosyste.ms: list of active SIGs.
[^6]: [o3de/community](https://github.com/o3de/community), GitHub.
[^7]: [RFC: Adding Github Codeowners File #25](https://github.com/o3de/sig-testing/issues/25), o3de/sig-testing on GitHub.
[^8]: [Proposed RFC: Merge access control and designated ownership of o3de/o3de.org #61](https://github.com/o3de/sig-docs-community/issues/61), o3de/sig-docs-community on GitHub.
[^9]: [GitHub Workflow](https://docs.o3de.org/docs/contributing/to-code/git-workflow/), O3DE documentation.
[^10]: [Contributing to O3DE Code Sources](https://docs.o3de.org/docs/contributing/to-code/), O3DE documentation.
[^11]: [Overview of the Docs Contribution Process](https://docs.o3de.org/docs/contributing/to-docs/overview/), O3DE documentation: DCO sign-off and review requirements.
[^12]: [Submit a PR to O3DE Docs](https://version-2205--o3deorg.netlify.app/docs/contributing/to-docs/submit-a-pr), O3DE documentation (22.05).
[^13]: [o3de/o3de pull request #17292](https://github.com/o3de/o3de/pull/17292), GitHub: an example merged PR with DCO sign-offs and a DCO remediation commit.
[^14]: [RFC: Intermittent Failure Intervention #43](https://github.com/o3de/sig-testing/issues/43), o3de/sig-testing on GitHub: definition of Automated Review.
[^15]: [Overview of the Docs Contribution Process](https://docs.o3de.org/docs/contributing/to-docs/overview/), O3DE documentation.
[^16]: [Contributing to the O3DE Engine Developer Guide](https://www.docs.o3de.org/docs/engine-dev/contributing), O3DE documentation.
