---
name: nauvim-release
description: Prepare or resume a Nauvim specification release and coordinate its core, plugin, and GUI follow-up PRs. Use for a requested specification release or an individual stage of that release.
---

# Release the Nauvim specification

Start from the requested `vX.Y.Z`. Follow any requested repository or stage
limits. For a full release, continue through the stages below as the required
merges and publication become available. The maintainer reviews and merges PRs.

Use the local checkouts of `nauvim-spec`, `nauvim-core`, `nauvim-plugin` and
`nauvim` under the `nauvim` GitHub organization. Read each target's repository
instructions and start new branches from its current remote `main`. Use a
separate worktree when the current checkout contains other work. Use the GitHub
connector or an authenticated `gh` for GitHub operations.

## 1. Release the specification

Look up the remote tag and the `release/vX.Y.Z` PR, including merged PRs. If the
tag already exists, resolve it to its commit and verify the requested version
in the tagged files. Otherwise, prepare the `release/vX.Y.Z` branch and compare
its specification changes with the previous release tag. Complete both checks
before committing the version bump.

1. **Terminology consistency.** Check changed terms against their definitions
   and related uses in `specification.md` and `type-notation.md`. Check that
   component names, identifiers, and API terminology remain consistent.
2. **ISO/IEC drafting style.** Apply the repository's
   [drafting rules](../../../AGENTS.md#draft-in-the-established-specification-style).
   Check clause structure and references, the strength of modal verbs, and the
   placement of requirements in normative text.

Commit any editorial fixes before the version bump. Preserve the specified
behaviour and requirement strength. Then set the version in `specification.md`
and `README.md` in a separate commit. Check that the release diff contains the
intended changes and that the version follows clause 5.4. Create or update the
release PR and record both checks there. When resuming, review changes made
since the recorded checks.

The **Tag specification release** workflow creates the tag after merge. If a
merged release PR has no tag, check the workflow result. Its manual input
`pr` takes the same PR number. Use the remote tag's commit for the follow-up work.

## 2. Update core and plugin

Run these follow-ups independently. In each repository, compare the specification
revision recorded on its `main` with the new tag. Follow the structure of that
repository's previous specification update issue. Include the old revision,
new tag and commit, comparison link, and changes relevant to that component.
Examples are [core issue #6](https://github.com/nauvim/nauvim-core/issues/6) and
[plugin issue #3](https://github.com/nauvim/nauvim-plugin/issues/3).

Search all states for the issue titled `spec: Update to vX.Y.Z` and the PR
from `spec/vX.Y.Z`. Reuse each existing issue, branch, or open PR. Record the
merge result for a merged PR. If a PR was closed without merging, ask whether
to resume it. For a pending follow-up, implement the affected behavior,
update the pinned specification references, run the affected checks, and
link the issue in the PR. Apply the repository's version policy when changing
the core crate version. A follow-up with no behavior changes may only need
reference and version updates.

## 3. Publish core

After both follow-up PRs merge, take the package version from `Cargo.toml` at
the core PR's merge result. Check for an existing publish run or a published
crate at that version before dispatching. The exact published version and its
specification revision must agree with the release. The crate archive includes
`.cargo_vcs_info.json` and its packaged README for checking provenance.

If publication is needed, dispatch **Publish nauvim-core** on `main` for the
merged core PR. For example:

```sh
gh workflow run publish.yml --repo nauvim/nauvim-core --ref main -f pr=123
```

If the available GitHub tools cannot dispatch workflows, give the maintainer
the [workflow link](https://github.com/nauvim/nauvim-core/actions/workflows/publish.yml)
and PR number. Check the run outcome and confirm that the exact version can be
resolved from crates.io before updating the GUI.

## 4. Update the GUI

Use the same issue and branch conventions in `nauvim`. Follow the structure of
[GUI issue #13](https://github.com/nauvim/nauvim/issues/13). Include the target
specification tag and exact published core version. Update the specification
reference, Rust dependency and lockfile, and any affected Flutter, Rust bridge,
or native runner code. Run the affected checks and link the issue in the PR.

## Resume

Use GitHub and crates.io to determine the current stage. When waiting on a
maintainer or an unavailable permission, report the PR or workflow link and the
pending action. The maintainer can resume with the same request, or request a
single stage and defer the rest. From another repository's Codex chat, give it
this file and the stage to run.
