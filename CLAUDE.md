# Project instructions

## What this repo is

A RAG platform for one French public administration. Administrators bring documents in from three kinds of source
(upload, Git repository, S3 bucket), review what changed, and publish. The administration's users question the
published documents in a chat and get answers that cite them.

A document goes through four steps: scan, review by an administrator, acquisition (parse, chunk, index), publication.
Each scan, acquisition and publication is a job, and every job is one Temporal workflow. A nightly run does the
housekeeping.

## Commands

No command exists yet: the repo is being bootstrapped. Commands will run from the repo root, through the root
`Makefile`.

## Architecture

- **Catalog**: PostgreSQL, one database.
- **Search**: Elasticsearch, one index. One entry per chunk, with a `status`: `draft`, `published` or `inactive`.
- **Files**: S3-compatible storage. The documents bucket holds the original files, with versioning on. The processing
  bucket holds what the platform builds from them.
- **Jobs**: Temporal, self-hosted, Python SDK. One task queue per worker type: `control`, `sync`, `parse`, `chunk`,
  `index`, `publish`, `delete`, `notify`. The `control` worker also runs the workflow code.
- **APIs**: FastAPI. Each API verifies the identity provider's token itself. The Control API serves the administrators
  and starts the jobs. The Jobs API serves job progress to the console, which polls it every few seconds. The Query
  API serves the chat: retrieval and generation.
- **Models**: one LLM gateway (LiteLLM).
- **Identity**: the administration's identity provider (OIDC). No local accounts.
- **Front-ends**: two single-page applications, the console and the chat, built with the French State design system
  (DSFR).

## Vocabulary

Each word has one meaning in the code.

- **User**: a person of the administration, logged in through the identity provider.
- **Administrator**: a user listed on a project, who manages it in the console. **Platform admin**: a user with the
  `platform_admin` role of the identity provider, who creates projects.
- **Origin**: where the files of a Git or S3 source live. An upload source has no origin. **Deposit**: the files sent to
  an upload source in one upload session.
- **Document**: one file at one path inside one source. **Version**: the content of a document as it was fetched at one
  time. A document has several versions.
- **Change**: what happened to a document: an addition, a modification or a removal. A removal is a change, never a
  version.
- **Published project version**: the project's documents as the last publication left them. **Draft**: the next one,
  built by the acquisitions and invisible to the chat.
- **Original**: a version's file, as it was fetched. **Artifacts**: everything the acquisition built from it: the parsed
  file, the chunks file, the chunk rows and the indexed chunks.
- **Job**: one run of a workflow that an administrator starts, such as a scan, an acquisition or a publication. It has
  one row in `job`. The nightly run is not a job.

Names use the full term: `document_version_id`, never `version_id` or `doc_id`.

One verb per action. For deletion, it is `delete`: no `erase`, `clean` or `purge`. When only a part goes, the name says
which part: `delete_superseded_version_artifacts`.

## Hard rules

### Data

- The catalog is the source of truth. Elasticsearch is corrected towards it, never the other way round.
- When one operation writes to the catalog and to Elasticsearch or S3, write the catalog last. A publication switches
  the index first, then commits one catalog transaction. A deletion empties the other stores first, then records it in
  the catalog.
- A deleted document or version keeps its row in the catalog, as a trace.
- The chat reads the published project version only. Only a publication changes it.
- Every call to a store or an external service has its own timeout, so a stuck call ends with a clear error.

### Security

- A source's secrets, such as a Git token, live in the secret manager only. The catalog holds a reference. A
  component's own credentials come from its settings. Never a secret in the catalog, a log, a response or a command
  line.
- Access is checked on the server, on every request. Find an object's project from the object itself, in the catalog,
  never from a project id sent in the request. Example: to delete a document, load it, then check the user's rights on
  its project.
- Every LLM, embedding and reranker call goes through the LLM gateway. Never call a provider SDK directly.
- Logs are structured JSON. Each line carries the ids that apply: the request id in an API, the job id in a job, the
  project id when there is one. Never file content.

### Scope

- Keep v1 simple. Do not add edge-case handling, caches or options that were not asked for.
- Read the current official docs of a library or framework before using it or proposing a pattern with it. Prefer its
  docs MCP server or its bundled skill when one is available. Use current APIs and recommended patterns, never
  remembered ones.
- Consult me before any important decision.

## Conventions

### Code

- Screen labels are French. Code, identifiers, comments and documentation are English.

### API

- Each API has its own versioned prefix: the Control API under `/v1/control/`, the Jobs API under `/v1/jobs/`, the
  Query API under `/v1/chat/`.
- Every request carries the identity provider's bearer token. No cookie, no API session.
- Starting a job answers `202` with `{job_id}`.
- A FastAPI dependency is exported with an `Annotated` alias next to it, so a route declares a type instead of repeating
  `= Depends(...)`.

## Known traps

Add an entry here every time we lose time on something.

- Always sync with `uv sync --all-packages`, never a plain `uv sync`. A plain `uv sync` installs only what the root
  project needs, so it removes from `.venv` what only the members need. The FastAPI skill link, which points into
  `.venv`, then leads nowhere.
