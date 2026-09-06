# Diagnosis: permanent source-processing errors are treated as transient

## Responsible module, function, and mistaken assumption

The primary defect is in `commands/source_commands.py`, in the retry decorator and exception handling of `process_source_command` (lines 38–50 and 132–145). Its retry policy excludes only `ValueError` and `ConfigurationError`. The catch-all `except Exception` labels every other failure a **transient error**, logs it at DEBUG, and re-raises it into automatic retry. The mistaken assumption is that any exception outside that narrow stop list is a temporary failure, like the SurrealDB transaction conflicts for which the retry policy was configured.

A provider's HTTP 400 rejection because the input exceeds its context window is permanent for the unchanged request. Waiting and rerunning the same source processing does not reduce that input or change the selected model.

## Error propagation

1. `process_source_command` awaits `source_graph.ainvoke` with the source and requested transformations (`commands/source_commands.py:96–104`).
2. `transform_content` passes `source.full_text` to the transformation graph (`open_notebook/graphs/source.py:258–271`). `run_transformation` builds the model payload and awaits `chain.ainvoke(payload)` (`open_notebook/graphs/transformation.py:23–70`).
3. On a provider exception, `run_transformation` calls `classify_error` and raises the resulting domain exception. In `open_notebook/utils/error_classifier.py`, the context-length rule explicitly maps messages containing `context length`, `maximum context`, `context_length_exceeded`, `token limit`, or `max_tokens` to `ExternalServiceError`. Unrecognized errors also fall back to that type.
4. `ExternalServiceError` inherits from `OpenNotebookError`, not `ValueError` or `ConfigurationError` (`open_notebook/exceptions.py`). Consequently, a recognized context-length rejection bypasses the command's permanent-error handler and retry stop list. The classifier also uses this same exception type for temporary provider availability failures, so the type does not distinguish retryability.

## Why it appears stuck without an error

The command emits INFO messages when starting each attempt, but its catch-all failure message is DEBUG. The decorator also sets `retry_log_level` to `debug`, explicitly to suppress transaction-conflict noise. Recognized context-length errors return from `classify_error` without its fallback warning. Thus the application path suppresses the actionable failure at ordinary INFO logging levels while retrying; the command comment delegates final failure logging to the worker framework.

The source UI follows the command's reported status, rather than treating each unsuccessful attempt as a terminal source failure. `Source.get_status` and `Source.get_processing_progress` read the linked command status and `error_message` (`open_notebook/domain/notebook.py:436–474`). `get_source_status` returns generic messages such as “Source processing in progress” for `running` (`api/routers/sources.py:843–902`). `useSourceStatus` keeps polling active statuses (`frontend/src/lib/hooks/use-sources.ts:234–259`); `SourceCard` displays the generic status message and exposes retry actions only for `failed`. This delays a visible terminal failure while automatic retries continue. The status API can expose the worker's error through `processing_info.error`; the code does not establish that errors remain invisible forever.

## Scope and limits of the finding

The checked-in policy specifies **15 attempts**, with exponential jitter and waits configured between 1 and 120 seconds. It is not an explicitly infinite retry loop. Long model calls plus repeated backoff can make processing look indefinite, but literal endless retries would require additional runtime evidence. The directly supported defect is retrying deterministic provider failures under a broad transient-error assumption, combined with DEBUG-only reporting during those retries.

This diagnosis is based on tracing the checked-in command, graph, classifier, exception hierarchy, and status-display paths. No live provider failure was reproduced, and no implementation files were changed.
