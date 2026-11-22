# Agent instructions

## Repository purpose and document roles

This repository maintains the Nauvim specification. It contains documentation,
not the application implementation.

| File | Role |
| --- | --- |
| [specification.md](specification.md) | The authoritative technical contract, including requirements, recommendations and expressly unspecified behaviour. |
| [README.md](README.md) | A concise, informative reading guide with a table of contents, section summaries and API shortcuts. |
| [LICENSE](LICENSE) | The repository's CC BY 4.0 license text. |
| AGENTS.md | Instructions for maintaining these documents; not an additional product specification. |

Write repository documents in English. Respond to the user in the language of
their current request, unless they request another language.

## Establish the intended change

- Follow applicable higher-priority instructions. Within this project, the
  user's latest instructions determine the requested change, including changes
  that supersede earlier decisions. Use the current specification for all
  unaffected behaviour.
- Start with the README and the relevant specification clauses. Read related
  definitions, common rules, exceptions and examples before editing. Do not
  reconstruct the current contract from memory or assume access to past chats.
- Distinguish an editorial correction from a behavioural change. Editorial
  work must preserve requirement strength, defaults, public interfaces and
  observable behaviour.
- Inspect the working tree and preserve unrelated changes. Keep edits focused
  while updating every passage affected by the requested change.
- Complete sufficiently specified changes without asking for routine
  confirmation. If a material conflict prevents a coherent result, identify the
  exact clauses or APIs involved, explain what the existing rules do not decide,
  and ask a focused question after completing the independent work.

## Preserve deliberate specification boundaries

- Keep **unspecified**, **implementation-dependent**, **undefined behaviour**
  and **unresolved design decisions** distinct. The first three are deliberate
  provisions, subject to their surrounding constraints. Do not turn them into
  TODOs or choose additional behaviour unless the user changes the contract.
- Before reporting an omission, check whether definitions, common rules,
  precedence rules, error handling or lifecycle requirements already determine
  the result. A consequence of an existing rule is not a new open decision.
- Do not add APIs, validation, retries, fallback behaviour, waiting requirements
  or completion guarantees merely to make a description appear more complete.
  Add them only when justified by the requested change or existing requirements.
- Respect the project's defined terms and any explicit definitions supplied by
  the user. Do not replace deliberate Nauvim behaviour with familiar industry
  defaults or preferred implementation practices.
- Preserve the stated scope of external conformance. Using an external format
  or citing a standard does not by itself require full compatibility. In
  particular, a best-effort recommendation must not become a conformance
  requirement through an editorial change.

## Draft in the established specification style

- Maintain the ISO/IEC-style structure and clear English used in
  `specification.md`.
- Use **shall** for requirements, **shall not** for prohibitions, **should** for
  recommendations, **may** for permissions and **can** for possibilities or
  capabilities. Do not interchange these forms or substitute **must** or **will**
  for a requirement.
- Put requirements and recommendations in normative text. Notes, examples and
  informative annexes explain the contract without adding independent
  obligations or information essential to determining behaviour.
- Use defined terms consistently and name the component that performs an
  operation. Keep host and client environments, sessions, connections, channels
  and UI attachments distinct.
- Keep process client IDs, Neovim channel IDs and resource handles distinct.
  Distinguish public Lua signatures from internal RPC arguments, and request
  errors from errors displayed by Nauvim.
- Separate a returned handle from resource readiness or operation completion.
  Preserve distinctions between requesting cancellation, task termination and
  resource deletion as specified for the affected operation.
- Use fenced `lua` blocks for Lua examples and `text` blocks for descriptive
  signatures or type notation. Keep examples consistent with the normative
  rules without introducing defaults or guarantees through example values.

## Update related provisions together

Use this table to locate affected material, not as a second definition of the
technical contract. Consult the current specification for the actual rules.

| Change area | Related material to review |
| --- | --- |
| Lua API or RPC | Public signature, method mapping, wire argument order, validation, results, error handling, no-effect rules, feature flags and examples. |
| Asynchronous operation or resource | Scheduling, handle allocation and lookup, idempotency, readiness queries where present, cancellation, `clear`, and applicable cleanup paths. |
| Session lifecycle | Startup, ownership, attachment, detachment, host switching, host termination, interruption recovery, shutdown and resource ownership. |
| Configuration | Types, defaults, validation, resolution and merge rules, host/client interpretation, path handling, examples and affected README summaries. |
| GUI or clipboard | Responsibilities of core, plugin and UI backend, provider or presentation interfaces, input rules and error handling. |

Review the applicable detachment, switching, recovery and shutdown provisions
when changing resource cleanup. Preserve method-specific exceptions to general
rules; do not infer stronger stopping or completion guarantees.

When a change relies on an external API or protocol, consult authoritative
primary documentation for the version relevant to the specification. Preserve
the distinction between normative references and informative sources. Record
supporting references using the document's existing citation conventions.

## Maintain navigation and repository consistency

- In `specification.md`, keep clause numbers and cross-references consistent.
  Number main-text tables consecutively; use separate annex sequences such as
  `Table A.1`. Give each table a caption and refer to it in the text.
- Update all affected references when renaming or renumbering a section or
  table. Check bibliography labels and references when adding or changing a
  source.
- Update README links and summaries when a change makes them inaccurate. Keep
  its overview brief and its contents at a useful reading level; do not copy
  detailed requirements into it.
- Use repository-relative links between files and stable public URLs for
  external sources. Check GitHub heading anchors after heading changes.
- Keep chat-only citation markers, temporary workspace paths, tool-run
  identifiers and tool-specific storage links out of repository documents.
- Preserve the license text and notices unless a licensing change is requested.
  Do not add separate licensing exceptions for code examples on your own.
- Keep technical details in `specification.md`. Update this file only when the
  user requests a change to the maintenance instructions.

## Specification releases

- Use the [release skill](.agents/skills/nauvim-release/SKILL.md) for a release
  or an individual follow-up stage.
- Prepare releases on `release/vX.Y.Z` and set the same version in
  `specification.md` and `README.md`. Leave PR merges to the maintainer.
- Use `spec/vX.Y.Z` for follow-up PRs in the implementation repositories.

## Commit messages

When creating commits, write the subject and body in English using this format:

```text
<kind>: <short summary>

<summary>
```

Use `spec`, `docs` or `chore` for `<kind>`, following the
[branch naming categories](README.md#branch-naming). Choose the kind according
to the changes in each commit. Keep the subject concise and separate it from
the body with one blank line. The body summarises what changed and why.

## Verify and report

Before finishing:

1. Read the edited passages with their affected definitions and common rules.
2. Inspect the diff for unintended changes to behaviour, normative strength,
   signatures, defaults, error handling and the scope of unspecified behaviour.
3. Check affected numbering, table references, Markdown structure, relative
   links, heading anchors and examples with lightweight checks appropriate to
   the change.
4. Confirm that README summaries and informative material still match the
   specification and that unrelated user changes remain intact.

This documentation repository does not require an application build or a
runtime test suite to validate prose. Do not introduce implementation code,
dependency manifests, test suites or CI jobs merely for a documentation edit.
Use existing tools where useful; do not install dependencies or broaden checks
without a concrete need.

Report what changed, why, which consistency checks were performed and any
material unresolved conflict. State only the verification actually completed.
Do not claim that an implementation was tested when only documentation was
reviewed. Publish, push or modify implementation repositories only when those
actions are part of the user's request.
