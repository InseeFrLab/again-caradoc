# Python conventions

## Functions and side effects

Default to pure: same arguments in, same value out.

- Never mutate an argument. Return a new value.
- Importing a module must not do work: no file, network or database access, no `datetime.now()` at module
  level. Do it in a function called at startup.
- Pass the specific values a function needs, not the whole settings object. Only startup code reads settings.

## Typing and syntax

- Target Python 3.14.
- `str | None`, never `Optional[str]`.
- Annotate every return, including `-> None`. A function that always raises returns `NoReturn`.
- Alias a composed type you repeat, with `type`: `type Embedding = list[float]`.
- Docstrings follow the Google format.

## Packages and imports

- Every package is a workspace member. No `sys.path` edits.
- Absolute imports by package name.
- Do not import one service from another. Shared code goes in `backend/libs/`.
- Do not duplicate a helper that exists in `backend/libs/`. Search first.
- A model that two components exchange lives in a shared library under `backend/libs/`.

## Settings

- All settings load from environment variables through typed settings classes. No hardcoded URL, model name, key or
  timeout.
- New config goes in as a settings mixin, never into one big class.
- A new setting goes into its settings class and `.env.example` in the same change.

## Async

- Async for anything doing I/O.
- Never call a blocking API inside a coroutine. If it cannot be avoided, tell me, and we will find a proper
  solution. In an activity, `temporal.md` gives the solution.

## Naming

Spell names out. The reader should not have to look up what something holds.

- `settings`, not `s`. `http_client`, not `http`.
- Name explicitly what an argument is: `search_input`, not `params`.
- Functions start with a verb that says what they actually do.
- A name must not promise more than the code does: `extract_text`, not `process_document`, if it only
  extracts text.

## Layout

- Dicts, lists and other collections: one entry per line, including as call arguments. The formatter joins
  short collections onto one line; a **trailing comma on the last entry** keeps them one per line, so write one.
- Signatures with two or more parameters: one per line, with the same trailing comma.
- Calls with two or more arguments: one per line, each one named. Positional only where the function
  forbids keywords: `getattr(obj, name, default)`.
- Imports at the top of the module, never inside a function.
- Text on several lines: a triple-quoted string.
- Never glue adjacent literals or chain `+`: implicit concatenation drops the space at line breaks.
