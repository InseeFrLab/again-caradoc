# Temporal

Every job is run by one Temporal workflow, which may start child workflows. Check a pattern against
https://docs.temporal.io/design-patterns.md before using it.

## Jobs

- A job's workflow id is the job's id.
- A job is started by a route of the Control API, such as the one behind "Scanner maintenant". That route takes the
  session factory. One `transactional_session` block does the checks, prepares what the job needs and creates the `job`
  row as `queued`. The workflow starts only after that block has committed, so its first activity finds the row.
- One job at a time per project. That first block checks this rule with the project's row locked, so two requests at
  the same moment cannot both pass.
- The workflow starts with `id_reuse_policy=WorkflowIDReusePolicy.REJECT_DUPLICATE`. Temporal then refuses a second
  start with the same id. So when a start request times out, the Control API route sends it again: the answer "already
  started" (`WorkflowAlreadyStartedError`) counts as a success.
- If Temporal still does not answer after the retries, the Control API route gives up. In a second
  `transactional_session` block, it marks the job failed and puts back what the first block changed. Example: an
  acquisition's versions leave `in_acquisition`, and their changes go back to the review. It does this only if the job
  is still `queued`: a job already `running` means the workflow started after all, so that job is left alone.
- To cancel a job, the Control API marks it `cancelling`, then asks Temporal to cancel the workflow. The workflow's
  closing steps write `cancelled`.
- A job workflow starts with the activity `mark_job_running`, which sets the job to `running`. If that activity finds
  the job already closed, it raises `job_closed`, and the workflow stops at once. Otherwise, the workflow's closing
  steps run however its main steps ended: a normal end, a cancel, a step that failed for good, or a bug in the
  workflow's own code. They run in a block a cancel cannot interrupt, and end with `close_job`, then `send_email` when
  one is needed.
- On a cancel, a workflow waits for its running activities to stop before it runs its closing steps. To make it wait,
  every activity call sets `cancellation_type=ActivityCancellationType.WAIT_CANCELLATION_COMPLETED`.
- Among activities, only `close_job` writes a job's outcome and reason. An activity that must stop a job only raises its
  error, and the workflow passes it to `close_job`.
- Job progress is read from the catalog, never from Temporal. Only `detect_lost_jobs` asks Temporal, to learn whether a
  job's workflow still exists.
- A workflow gets no retry policy, so Temporal never runs a failed workflow again. When a job fails, its activities
  have already retried what could be retried. The administrator fixes the cause, then starts a new job.

## Workflows

- Temporal can run workflow code again from the start at any time, for example after a worker restart. It then gives
  it back the recorded results of its activities. So workflow code must take the same decisions on every run. It never
  does anything whose result can change from one run to the next: read the database, call a service, read a file or the
  clock, draw a random value. All of that goes in activities. For the time and for random values, it uses
  `workflow.now()`, `workflow.random()` and `workflow.uuid4()`.
- Workflows and activities live in separate modules. Temporal runs each workflow in a sandbox, and reloads the workflow
  module and its imports on every run. Imports placed inside `with workflow.unsafe.imports_passed_through():` skip
  that reload: the module already loaded is reused. A workflow module imports its activities, models and libraries
  that way.
- A setting that a workflow uses, such as a group size, arrives in its input, set by whoever starts it. Workflow code
  never reads settings or the environment: a value that changed between two deployments would break a running workflow.
  Activities are normal code: they get their settings when the worker starts.
- A workflow and an activity each take one input object and return one result object, or nothing. These objects are
  pydantic models, and the client and the workers use `pydantic_data_converter`. A field with a default value can then
  be added later without breaking the workflows already running.
- A workflow calls each activity on the task queue of its worker type.
- A workflow ends as failed by raising `ApplicationError`. It treats a bug in its own code the same way: it catches an
  unexpected exception from its main steps, runs its closing steps with the reason `internal_error`, then raises
  `ApplicationError`. If the exception were left alone, Temporal would retry the workflow forever, and the job would
  stay `running`. One exception must pass through untouched: `ContinueAsNewError`, which is how a workflow continues as
  new.
- Two cases still leave a workflow retrying: a bug in its closing steps, and a deployment that breaks the replay. Roll
  the deployment back or deploy a fix, and the workflow goes on.
- Do not use the worker setting `workflow_failure_exception_types` to end those two cases. It lists the exception types
  that fail a workflow instead of leaving it retrying. But the SDK marks it experimental, and it ends the workflow
  without its closing steps: the job stays open until the nightly run closes it.
- A change to the steps of a workflow breaks the replay of the runs in progress: an activity added, removed or moved, a
  new timer. Such a change goes through one of Temporal's versioning methods. Which one is not decided yet: ask before
  the first change of this kind.
- To wait for several tasks, workflow code uses `workflow.wait()` and `workflow.as_completed()`. It never uses
  `asyncio.wait()`, `asyncio.as_completed()` or `asyncio.TaskGroup`: with those, the order of the tasks can change
  from one run to the next.
- No workflow waits for a person or sleeps for days. A human step sits between two workflows. A delay is a date written
  in the catalog, which the nightly run reads. The reason: a waiting workflow keeps its job open, which blocks its
  project, and it must survive every deployment made while it waits. If a feature seems to need a long wait, stop and
  ask.
- A workflow never holds a whole list of rows. It asks an activity for the next few rows when it needs them, so its
  recorded history stays small.
- Temporal stores inputs, outputs, error messages and stack traces as readable text. Pass ids, numbers and fixed
  values. Never document content, personal data or secrets. An error message never repeats an origin's answer as is: for
  example, a Git error can hold the repository URL with its token.

## Activities

- Each activity does one thing. Do not merge unrelated work into one activity.
- Every activity is safe to run twice. It overwrites its own results or skips what is already done. It never appends.
- A file that fails for its own reason is a result, not an error. The activity records the failure, and the other files
  go on.
- An activity reads rows a page at a time, with one query per page. Never one query per row.
- Every activity call sets a time limit per try (`start_to_close_timeout`) and a retry policy of its own, sized for
  what the activity does. Temporal's default policy never gives up, so never keep it. The waits between tries must
  outlast the outage the activity can meet: a quick first retry, then waits that grow to minutes. Otherwise, with
  waits of only a few seconds, all the tries are over before a restarting service is back.
- An activity raises a failure it expects as `ApplicationError`, with a `type`. The `type` is a string we choose, not
  a Python class. It names the reason, and it is one of our own error types. Example:
  `raise ApplicationError("The origin refused the credentials", type="access_lost", non_retryable=True)`.
- An outage is retried, so its error is raised without `non_retryable`. Its type is `origin_unreachable` when the
  origin does not answer, and `platform_error` when one of the platform's own services does not answer, such as
  PostgreSQL or Elasticsearch.
- A failure that a retry cannot fix is raised with `non_retryable=True`. Example: `access_lost`, when the origin
  refuses our credentials.
- A failure we did not raise comes in two kinds. A bug gives its Python class name as the type, such as `KeyError`. A
  dead worker, or a try that runs past its time limit, raises nothing: Temporal ends it with a timeout, and that
  failure has no type. Temporal retries both. If one of them ends the job, the workflow records it as
  `internal_error`: the team must look at it.
- Raise `ApplicationError` itself, with its `type` set. Never subclass it or any other Temporal error class: Temporal
  may fail to carry a subclass between processes.
- Let an `ApplicationError` pass as it is. Never catch it and raise a plain exception in its place, as in
  `raise RuntimeError("scan failed") from error`. Temporal only looks at the last error raised, so the type and the
  "do not retry" mark of the first one are lost, and the activity is retried.
- An activity that can run long sends a heartbeat every few seconds. A small asyncio task, started by the activity,
  sends them beside the main work, so they keep going during one long step, such as a big download. The activity is
  called with a short heartbeat timeout. Temporal then sees a dead worker quickly and runs the activity again. The
  retry goes on from the position saved with the last heartbeat, or skips what is already done. A cancel also reaches
  the activity through its heartbeat. The worker does not send every heartbeat, so a cancel or a saved position can be
  late by up to 80% of the heartbeat timeout.
- An activity is async, since it uses the catalog. A sync call inside it, such as a parser, would block the event loop
  and stall every activity and heartbeat on the worker. So it runs in a thread: `await asyncio.to_thread(...)`.
- A sync call keeps its thread busy until it ends. So a worker that runs 20 activities at once, each in a sync call,
  needs 20 threads. Python's default thread pool is small (the CPUs plus 4): give the worker a pool as large as its
  `max_concurrent_activities`, or lower that number. Threads do not speed up CPU work: a CPU-heavy worker takes few
  activities at once, and more capacity comes from more worker processes.

## Tests

- A workflow is tested with Temporal's time-skipping test environment and mocked activities, including its cancel and
  failure paths.
- An activity is tested by running it twice: the second run changes nothing.
