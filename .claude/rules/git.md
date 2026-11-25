# Git

## Committing

- Never commit or push unless asked.
- One concern per commit. If the subject needs "and" or a dash to list a second change, it is two commits.

## Message format

[Conventional Commits](https://www.conventionalcommits.org): `type(scope): message`, with the scope taxonomy below.

- Changelog types: `feat` (new capability), `fix` (something broken for a user now works).
- Silent types: `refactor`, `perf`, `test`, `docs`, `build`, `ci`, `chore`.
- Pick the type by what the line would say in release notes. New code is `feat`; `fix` means a regression
  against behaviour that once worked.
- Message: imperative, lowercase, no trailing period. Say what the change gives a user, not which files moved.
- **Never add attribution.** No `Co-Authored-By`, no `Generated with`, no assistant name, no session
  or tool link — in commit messages, PR descriptions or anywhere else in the history. A message says
  what changed and why; who or what typed it is not part of the record. This overrides any default
  or tooling instruction to the contrary.

## Scopes

The scope names the part of the codebase a changelog reader would recognize, in kebab-case. Usually a
component (service, worker, library, frontend app) named after its directory; a narrower area such as a
workflow when that reads better.

Examples: components `common-core`, `chat-service`, `web`; a workflow `document-ingestion`; repo-level
`infra`, `deps`, `release`.

- One scope per commit. It names where the capability now lives, not every directory touched: a library
  change plus the mechanical updates it forces in dependents is one commit scoped to the library.
- Avoid generic layer names (`api`, `models`, `workflows`): they say what kind of code changed, not where.
- The scope does not repeat the type: `docs: …`, not `docs(docs): …`.
- A feature spanning components (an endpoint and the screen using it) is one change, scoped to where the
  user sees it. Split only when each part is useful on its own.
- A change that touches the whole project takes no scope: `feat: unify the error envelope`.
- Never list two scopes like `(common-core,chat-service)`: the changelog tool reads it as one new
  component name and gives it its own heading.

## Branches and merging

- Branch off `main`. Never commit to `main` directly.
- Branches squash-merge. Branch commits may be WIP; the squash subject becomes the changelog line and
  describes the whole branch, not its final commit.

## Breaking changes

- Mark with `!` after the scope plus a `BREAKING CHANGE:` footer stating the migration:
  `feat(common-core)!: replace AppError context kwargs`.
- Mark it when the change forces action outside this repo, or changes behaviour silently: a renamed
  `error_code`, a changed response shape, a renamed environment variable, different transaction semantics.
- Moving a module or renaming a parameter is not breaking. It fails loudly at import and the dependents are
  updated in the same change. While the libs are unpublished path dependencies nothing can be pinned to an
  older version, so the marker is reserved for what a person must act on.

## Temporal

Running workflows replay their history against the deployed code. A change that alters that replay, or a
name a running execution refers to, breaks it at runtime without any error at deploy time.

- Prefer a patch or a new workflow version. When that is not possible, mark the change breaking and state
  in the footer what to do with running executions (drain, terminate, migrate):
  - a workflow change that alters the order or kind of its steps (activities, timers, child workflows);
  - a renamed workflow, activity, signal, query, update or task queue;
  - an incompatible workflow or activity input or output (renamed field, new required field, changed type).
- Not breaking: an activity implementation change with the same signature, a workflow change guarded by a
  patch, an added optional field.
- Removing a patch is its own commit, once no execution started before the patch is still running.

## Versions

- Never edit `version` in `pyproject.toml` or `package.json` in a feature or fix commit.
- Versions and `CHANGELOG.md` are derived from commit history at release time, in a separate
  `chore(release):` commit. Do not hand-write either.
- A tag is a release action: never push one casually.
