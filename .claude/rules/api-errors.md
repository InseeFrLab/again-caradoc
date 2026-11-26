# API errors

This file covers the errors an API returns. Errors inside Temporal activities follow `temporal.md`.

Every error leaves the API in the same shape, so the client reads all errors the same way: the HTTP status, plus the
body `ErrorResponse`: `{error_code, message, request_id, details?}`.

## Two levels of error classes

- **Families** live in `caradoc_core.exceptions`, one per HTTP status: `AuthenticationError` (401),
  `AuthorizationError` (403), `ResourceNotFoundError` (404), `InvalidInputError` (422), ... Each sets `status_code`, a
  generic `error_code` and a default `message`.
- **Conditions** subclass a family and set their own `error_code` and `message`. They live in the `exceptions` module
  of the package that owns them: `<service>.exceptions`.
- A class that needs response headers sets `headers`. Example: `AuthenticationError` sends `WWW-Authenticate: Bearer`.

## Raising

- For a failure you expect, raise an `AppError` subclass. Example: a job is already running for the project, so raise
  `JobAlreadyRunningError`.
- Never raise `HTTPException`, and never return an error dict or a `JSONResponse` from a route. Both skip the shared
  shape.
- FastAPI's security helpers raise `HTTPException` on their own. Create them with `auto_error=False` and raise the
  `AppError` yourself. Example: `HTTPBearer(auto_error=False)`, then `InvalidTokenError` when the token is missing.
- A bare exception (`ValueError`, `KeyError`) is only for real bugs, and for startup or config failures that should
  crash. If you raise a bare exception for an expected case, the safety-net handler treats it as a bug, which goes
  wrong twice: the client gets `500 internal_error` with a generic message, so it cannot tell the user what happened;
  and the log records an error, so monitoring sees an outage that is not one.
- The raise site picks the class. It never passes a status code or an error code. It sets `message` only when the
  default does not fit.

## When to create a condition class

- Create one when the client must react differently. Example: on `JobAlreadyRunningError` the console shows the
  running job; on `ResourceNotFoundError` it shows a "not found" page. Two reactions, two classes.
- Causes that get the same reaction share one class, and the exact cause goes in `context`, which is only logged.
  Example: an expired token and a bad signature both send the user back to the login page, so both raise
  `InvalidTokenError`, one with `reason="expired_token"`, the other with `reason="invalid_signature"`.

## error_code and message

- `error_code` is what the client branches on. It is snake_case and says what went wrong: `job_already_running`, not
  `conflict`, since the status already says conflict.
- Two classes never share an `error_code`. The handler logs it as the event name, so a log search for one code finds
  exactly one case.
- Renaming or reusing an existing `error_code` breaks the clients: mark the commit `!`.
- `message` is for humans. It can change at any time, so a client that parses it will break one day. Clients branch on
  `error_code`.

## details and context

- `details` is sent to the client. Put only what the client needs to show or decide: field errors, limits, retry hints.
- `context` (the `**context` keyword arguments) is logged and never sent. Put internal data there: ids, the exact
  cause, an upstream status code.
- Never put a token, password, secret, key, document content or raw upstream body in `details` or `context`. Both end
  up in the logs, and `details` also reaches the client.
- When unsure, use `context`.

## What the handlers do

`register_exception_handlers` installs four handlers. A new `AppError` subclass needs no handler change.

- `AppError` and its subclasses: the status comes from the class's `status_code`, the body from the exception's
  `error_code`, `message` and `details`. Example: `JobAlreadyRunningError` gives `409` with
  `"error_code": "job_already_running"`.
- `RequestValidationError`, raised by FastAPI when a request does not match its model: 422 / `validation_error`, with
  one entry per bad field in `details`: `{"address.city": "Field required"}`.
- `HTTPException` raised by Starlette or FastAPI's own helpers (unknown route, wrong method): a flat `http_error`. The
  client reads the status for those.
- Any other exception, as a safety net: 500 / `internal_error`. In production the message is a fixed sentence; the
  details are in the log, found with the `request_id`.

Handlers log: a warning for 4xx, an error for 5xx. Do not also log an error where you raise it.

## Streaming (SSE)

A stream sends its `200` status as soon as the route returns it, before the generator starts, so the handlers cannot
change it afterward.

- An error raised before the route returns the stream, in a dependency or the route body, still goes through the
  handlers. So check access and load what the stream needs there.
- Once the stream is returned, send any error as an `error` event carrying the same `ErrorResponse` JSON. Build it
  with `log_and_build_app_error_response` or `log_and_build_generic_error_response`.
- Never end a stream silently on error.
