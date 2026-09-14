# Transaction execution outcome metrics

This note provides an implementation contract for [issue #35659](https://github.com/ethereum/go-ethereum/issues/35659). The goal is operational observability: expose aggregate execution outcomes without changing transaction validation, consensus behavior, or RPC responses.

## Outcome vocabulary

The metric names should describe the lifecycle stage, not the application error text:

| Outcome | Meaning | Counted when |
| --- | --- | --- |
| `success` | The transaction reached execution and completed without an EVM error | the execution result is finalized |
| `reverted` | The EVM returned a revert error and state changes were rolled back | the execution result is finalized |
| `failed` | Execution terminated with a non-revert EVM failure (for example, out of gas) | the execution result is finalized |
| `rejected` | The transaction was not executable and never entered the EVM execution path | validation rejects it at the relevant boundary |

`reverted` and `failed` are execution outcomes. `rejected` is a pre-execution outcome and must not be inferred from a missing receipt. If a code path cannot distinguish these categories, it should not increment a more specific counter.

## Naming and labels

Prefer a small counter family such as `tx_execution_outcomes_total`, with a bounded `outcome` label containing only the vocabulary above. Do not add transaction hashes, addresses, selectors, peer IDs, error strings, or calldata as labels: those values create unbounded cardinality and can disclose user activity. If the existing metrics package requires separate counters, use the equivalent stable names and document the mapping in the registration site.

The metric should be registered exactly once, alongside the existing execution metrics. Registration must be independent of whether metrics collection is enabled, matching the repository convention for disabled collectors.

## Exactly-once rule

The increment belongs at the narrowest point where an outcome is final and cannot be retried by the same transaction-processing path. In particular:

- increment `success`, `reverted`, or `failed` after the EVM result is known;
- increment `rejected` at the boundary that owns the rejection decision;
- do not increment again when a receipt is assembled, a block is written, or a retry is logged;
- do not count a transaction that is merely queued, replaced, dropped, or reintroduced by a reorg as executed.

If a transaction is retried internally, only the terminal result should be observable in this family. Tests should make the retry boundary explicit instead of relying on log output.

## Verification matrix

The implementation PR should cover at least:

1. a successful call;
2. an explicit EVM revert;
3. a non-revert execution failure such as out of gas;
4. a validation rejection before EVM entry;
5. two processing attempts that produce one terminal metric increment;
6. disabled metrics collection, proving no execution behavior depends on the collector.

Tests should assert counter deltas, not global absolute values, so they remain isolated when the package is exercised in a larger suite.

## Non-goals and compatibility

This change must not alter gas accounting, receipt status, pool admission, block validity, or error propagation. Metrics are diagnostic and best-effort: consumers must tolerate missing samples during startup and must not use them as consensus data. The first code PR should keep the family aggregate-only; dashboards and per-client attribution can be proposed separately once the lifecycle semantics are reviewed.
