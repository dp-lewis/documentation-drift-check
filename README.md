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
gh aw add dp-lewis/documentation-drift-check@v2.0.0
```

To pick up later upstream changes:

```bash
gh aw update documentation-drift-check
```

An install pinned to a tag updates to the latest release within the same major
version. Crossing a major version — where the workflow's write scope or tool
access has widened — needs `gh aw update documentation-drift-check --major`, so
that you review the change before adopting it.

## Requirements in the consuming repository

- **Engine:** GitHub Copilot. No API-key secret is needed — the workflow requests
  `copilot-requests: write` and uses the repository's Copilot entitlement.
- **Actions setting:** *Allow GitHub Actions to create and approve pull requests*
  must be enabled (Settings → Actions → General), or the safe output has no way
  to open the PR.
- **Docs layout:** none assumed. The workflow looks for a `docs/` tree — falling
  back to `doc/`, `documentation/` or `website/docs/` — plus `README*` at the
  repository root, and decides how cautiously to edit each file from that file's
  own structure rather than from its path. A repository with no docs tree simply
  has less to change.
- **Conventions:** commit format, pull request titles and prose style are read
  from wherever your repository already keeps them, not imposed by the workflow.
  See [Conventions](#conventions).
- **Glossary (optional):** an existing glossary is corrected like any other
  reference file. Adding *new* terms is opt-in — it happens only where your
  repository's conventions ask for the glossary to be maintained; otherwise
  candidate terms are listed for a human instead. The workflow never creates one.
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
2. Searches the docs tree and root `README*` for anything describing those
   surfaces, and classifies each hit as **Contradicted**, **Incomplete**, or
   **Fine**.
3. Edits only the Contradicted and Incomplete files, editing each one as freely as
   its structure safely allows — reference-style entries stand alone and are
   corrected directly, while ordered procedures are left untouched if the fix
   would invalidate a later step.
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

## Conventions

The workflow imposes as little of its own house style as it can. Commit format,
pull request titles and prose style are repository-wide concerns that exist
independently of drift checking, so they are read from where your repository
already keeps them rather than restated here.

gh-aw's engines load instruction files automatically: the Copilot engine reads
`.github/AGENTS.md`, and the Claude engine reads `CLAUDE.md`. The workflow
additionally reads `AGENTS.md` at the repository root, which is the more common
location but is not loaded automatically. Whatever those files say about commit
messages and trailers governs — the workflow mandates no trailer of its own.

Where your repository says nothing, the defaults are a pull request titled
`docs: update for #<PR number>` and no commit trailer.

## Safety

The workflow is read-only against the repository and writes exclusively through
gh-aw [safe outputs](https://github.github.com/gh-aw/reference/safe-outputs/):

- `permissions` grants only `read` scopes, plus `copilot-requests: write`.
- `safe-outputs.create-pull-request.allowed-files` restricts the PR to the
  common documentation roots (`docs/**`, `doc/**`, `documentation/**`,
  `website/docs/**`) and `README*`. Source, tests and configuration cannot be
  modified. This list is the security boundary and is deliberately not inferred at
  runtime — narrow it to the single root your repository actually uses.
- `tools.bash` is an allowlist of read-only inspection commands (`ls`, `cat`,
  `grep`, `find`, `git log`, and similar) needed to search the docs tree. gh-aw
  adds the git commands its pull request machinery requires; nothing else is
  permitted.
- `max: 1` — at most one pull request per run.
- gh-aw's own defaults additionally protect top-level dot-folders, common
  manifest and lockfile paths, and common top-level documents — including
  `CONTRIBUTING.md`, `CHANGELOG.md`, `SECURITY.md` and `AGENTS.md`. Those stay
  protected.
- `protected-files.exclude` lifts that protection for `README.md` alone. gh-aw
  protects it by default, which conflicts with this workflow's purpose: a
  protected file makes the push fail and the run degrade into an issue carrying
  manual recovery steps, rather than opening the pull request. `README.md` is
  still bounded by `allowed-files`; every other protected path is untouched.

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
`copilot-requests: write` permission can then be dropped.

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
