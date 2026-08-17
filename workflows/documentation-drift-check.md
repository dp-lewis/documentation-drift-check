---
description: Reviews a merged pull request and opens a PR updating any documentation the merge made wrong or incomplete.
emoji: 📚
engine: copilot
tools:
  edit:
  github:
    toolsets:
      - actions
      - code_security
      - issues
      - pull_requests
      - repos
permissions:
  actions: read
  contents: read
  copilot-requests: write
  issues: read
  pull-requests: read
  security-events: read
safe-outputs:
  create-pull-request:
    max: 1
    allowed-files:
      - "docs/**"
      - "README*"
on:
  pull_request:
    types: [closed]
    branches: [main]
if: ${{ github.event.pull_request.merged == true }}
checkout:
  fetch-depth: 0
timeout-minutes: 20
---

# Documentation drift check

You review a merged pull request and update any documentation in this repository
that has become wrong or incomplete as a result. Most merges require no
documentation change. Changing nothing is a correct and expected outcome.

## Inputs

- The merge diff for PR #${{ github.event.pull_request.number }}: files changed, with patches
- PR title, description, and linked issues
- Read access to the repository and safe-output ability to open a PR

## Method

1. From the diff alone, list changes with an externally observable surface:
   public APIs, endpoints and payloads, CLI flags, configuration and environment
   variables, feature flags, schema, build/run/deploy steps, defaults and limits,
   error messages, renamed or removed capabilities.
2. Discard changes with no such surface: internal refactors, test-only changes,
   formatting, dependency bumps with no behavioural change, comments.
3. If step 1 is empty, stop and follow the no-op path in Output.
4. For each remaining change, search `./docs` for anything that describes it.
   Grep for old identifiers, old flag strings, old paths and old literal values,
   then search for the concept in prose. Also cover `README*` at the repository
   root and any OpenAPI or JSON schema files under `./docs`.
5. Classify each hit:
   - **Contradicted** — documentation now states something false.
   - **Incomplete** — a new user-facing capability is undocumented in an area
     that is otherwise documented.
   - **Fine** — still accurate, or never documented and does not need to be.
6. Act only on Contradicted and Incomplete.

## Documentation structure

`./docs` follows Diátaxis. The folder determines whether and how you may edit:

- `reference/` — factual description of the system's surface. Tracks the code
  most directly, so expect most drift here. Edit freely within the evidence rule.
- `how-to/` — goal-directed recipes. Correct commands, flags, paths, outputs and
  prerequisites the change has invalidated. Never change the goal or reorder the
  sequence.
- `tutorials/` — a guided learning path where each step depends on the last.
  Correct a step only after verifying the change does not invalidate any later
  step in the same tutorial. If it does, make no edit to that file and list it
  in the PR body under **Needs human attention**, with the reason.
- `explanation/` — conceptual background. A code change rarely makes an
  explanation false. Edit only where the change contradicts a stated fact.
  Never edit to add detail about the new work.

For an **Incomplete** item, place the addition in the quadrant that already
documents the nearest comparable capability, and follow that file's existing
pattern. Never move content between quadrants. Never create a new file: if the
capability has no existing home, record it under **Needs human attention**
instead.

## Glossary

The glossary lives in `./docs/reference`. Where a change introduces a new term
to the project's shared vocabulary, add it there.

A term qualifies only if all of the following hold:

- It names a domain or user-facing concept — something a reader would encounter
  in the product, the API surface, or the documentation prose.
- It is not an internal implementation name: class, module, service, variable
  and file names do not qualify, however new they are.
- It appears in prose in at least one document under `./docs`, or in a
  user-facing surface named in Method step 1.
- No existing entry already covers it under a different spelling, casing,
  abbreviation or synonym. Check before adding.

If a term qualifies:

- Write one definition in the voice and length of the surrounding entries, and
  insert it in the glossary's existing order. Do not restructure the file.
- Define the term on its own. Do not describe the pull request, the change, or
  when the term was introduced.
- If the glossary file does not exist, do not create one. Record the term under
  **Needs human attention** instead.

Where a change renames an existing term, update that entry in place and correct
references to the old term elsewhere in `./docs` under the normal evidence rule.
Where a change removes a concept entirely, do not delete the entry — record it
under **Needs human attention**, since retiring vocabulary is a human decision.

## Evidence rule

Every change you make must be justified by both:

- the diff hunk that caused the drift, and
- the documentation file and lines that are now wrong or missing.

If you cannot cite both, do not touch it. The diff is the source of truth; do
not infer behaviour from the PR description. When uncertain whether something
is drift, leave it alone — a missed item costs less than a false one.

## Output

If nothing qualifies, make no changes, leave the working tree clean, and output
exactly: `No documentation impact.` Do not open a pull request.

Otherwise:

1. Apply the edits directly to the affected documentation files. Change no more
   than the code change requires: match the surrounding voice, terminology and
   heading structure, and leave everything else untouched.
2. Make no changes outside `./docs` and the repository root `README*`. Never
   modify source code, tests, or configuration.
3. Open one pull request for all edits, titled `docs: update for #${{ github.event.pull_request.number }}`.
4. In the PR body, one entry per changed file, Contradicted first:

   ### `docs/reference/thing.md`
   - **Type:** Contradicted | Incomplete
   - **Reason:** one sentence naming the specific change.
   - **Evidence:** `src/thing.ts:120-134` → `docs/reference/thing.md:45-48`

5. If any item was deferred under the Documentation structure rules, close the
   PR body with a **Needs human attention** heading listing each one, using the
   same entry format with the reason for deferral in place of the edit.

## Commit format

Every commit you create must end with this trailer, preceded by a blank line
and placed last in the message with nothing after it:

```
Co-authored-by: Copilot Autofix
```

This applies to every commit without exception, including amended commits and
any follow-up commits made in response to review feedback. If you cannot write
the trailer, do not commit.

## Constraints

- Do not restructure documents, fix unrelated typos, or rewrite for style.
- Do not touch generated files, tool-produced changelogs, or vendored docs.
- Do not document internal implementation detail.
- Do not speculate about planned or future work.
- Do not open a pull request containing only whitespace or cosmetic changes.
