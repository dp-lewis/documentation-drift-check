# Documentation Drift Check

📚 A [GitHub Agentic Workflow](https://github.github.com/gh-aw/) that reviews each
merged pull request and opens a follow-up PR updating any documentation the merge
made wrong or incomplete.

Most merges require no documentation change. The workflow is written so that
changing nothing is a correct and expected outcome — it reports
`No documentation impact.` and opens no PR.

## Install

Requires the [`gh aw` CLI extension](https://github.com/github/gh-aw):

```bash
gh extension install github/gh-aw
```

Then, from the repository you want to check:

```bash
gh aw add dp-lewis/documentation-drift-check
```

That installs the workflow package and compiles it into
`.github/workflows/documentation-drift-check.md` plus its generated
`.lock.yml`. Commit both.

To install a specific release:

```bash
gh aw add dp-lewis/documentation-drift-check@v1.0.0
```

To pick up later upstream changes:

```bash
gh aw update documentation-drift-check
```

## Requirements in the consuming repository

- **Engine:** GitHub Copilot. No API-key secret is needed — the workflow requests
  `copilot-requests: write` and uses the repository's Copilot entitlement.
- **Actions setting:** *Allow GitHub Actions to create and approve pull requests*
  must be enabled (Settings → Actions → General), or the safe output has no way
  to open the PR.
- **Docs layout:** the workflow expects a `./docs` tree organised by
  [Diátaxis](https://diataxis.fr/) — `reference/`, `how-to/`, `tutorials/`,
  `explanation/` — and treats each quadrant differently. It also covers `README*`
  at the repository root. A repository without `./docs` will simply find nothing
  to change.
- **Glossary (optional):** if `docs/reference` contains a glossary, new
  domain terms are added to it. The workflow never creates one.
- **Branch cleanup (recommended):** enable *Automatically delete head branches*
  (Settings → General). See the note below.

### Note on the checked-out ref

gh-aw always appends its own "Checkout PR branch" step for `pull_request`
events, and there is no frontmatter option to suppress it independently of
disabling checkout altogether. For a *merged* PR this means:

- If the head branch was deleted on merge (the usual case), that step fails
  benignly and the run proceeds on the base branch at the merge — which is what
  you want.
- If the head branch still exists, the agent works from the PR head rather than
  the post-merge state of `main`, and the docs PR is based on it.

Enabling automatic head-branch deletion keeps the workflow on the first path.
`checkout: fetch-depth: 0` is set so the full history is available for the
merge-base calculation that backs the generated pull request.

## What it does

On a pull request merged into `main`:

1. Reads the merge diff and lists changes with an externally observable surface —
   public APIs, endpoints, CLI flags, config and environment variables, schema,
   build and deploy steps, defaults and limits, error messages, renamed or
   removed capabilities. Internal refactors, tests, formatting and dependency
   bumps are discarded.
2. Searches `./docs` and root `README*` for anything describing those surfaces,
   and classifies each hit as **Contradicted**, **Incomplete**, or **Fine**.
3. Edits only the Contradicted and Incomplete files, following the rules for the
   Diátaxis quadrant each one lives in.
4. Opens one pull request titled `docs: update for #<PR number>`, with an entry
   per changed file citing both the diff hunk that caused the drift and the
   documentation lines that were wrong.

Anything the workflow declines to change on its own — a tutorial step whose
correction would invalidate later steps, a capability with no existing home, a
retired glossary term — is listed in the PR body under **Needs human attention**
rather than edited silently.

Every change must cite both the causing diff hunk and the affected documentation
lines. Where that evidence is missing, the workflow leaves the file alone: a
missed item costs less than a false one.

## Safety

The workflow is read-only against the repository and writes exclusively through
gh-aw [safe outputs](https://github.github.com/gh-aw/reference/safe-outputs/):

- `permissions` grants only `read` scopes, plus `copilot-requests: write`.
- `safe-outputs.create-pull-request.allowed-files` restricts the PR to
  `docs/**` and `README*`. Source, tests and configuration cannot be modified.
- `max: 1` — at most one pull request per run.
- gh-aw's own defaults additionally protect top-level dot-folders and common
  manifest and lockfile paths.

The agent never pushes to a branch directly; the PR is created by a separate
job from the collected safe output.

## Configuration

Everything lives in the frontmatter of
[`workflows/documentation-drift-check.md`](workflows/documentation-drift-check.md).
After installing, edit your local copy in `.github/workflows/` and recompile:

```bash
gh aw compile documentation-drift-check
```

Common adjustments:

| Change | Field |
| --- | --- |
| Watch a different base branch | `on.pull_request.branches` |
| Widen or narrow what the PR may touch | `safe-outputs.create-pull-request.allowed-files` |
| Use a different agent | `engine` (`copilot`, `claude`, `codex`, `gemini`) |
| Raise the run budget | `timeout-minutes` |

Switching `engine` away from `copilot` requires the corresponding secret in the
consuming repository (for example `ANTHROPIC_API_KEY` for `claude`), and the
`copilot-requests: write` permission can then be dropped. The `Co-authored-by:
Copilot Autofix` commit trailer in the prompt body assumes the Copilot engine —
update it if you switch.

## Repository layout

```
aw.yml                                  package manifest
workflows/documentation-drift-check.md  the workflow source
.github/workflows/compile-check.yml     CI: validates the package on every push
```

The workflow is *not* installed into this repository — this repo is the package
source. Compiled `.lock.yml` files are generated at install time in the consuming
repository and are not checked in here.

## Development

```bash
gh aw validate --dir workflows --strict   # validate without emitting lock files
gh aw compile --dir workflows             # compile to inspect the generated YAML
```

CI runs the validate command on every push and pull request.

## License

MIT
