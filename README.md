# GenLayer Hardened Prediction Market

A hardened variant of GenLayer's football prediction market example, focused on more robust handling of external web failures, AI-generated outputs, unresolved markets, and untrusted input.

This project was built and tested using GenLayer Studio.

## Motivation

The original GenLayer prediction market example demonstrates an important Intelligent Contract workflow:

1. Fetch football results from BBC Sport.
2. Ask an LLM to extract the match result.
3. Use GenLayer's Equivalence Principle to reach validator consensus.
4. Persist the result on-chain.

During adversarial testing, the original example behaved correctly for normal match resolution and nonexistent matchups.

However, when the external BBC resolution page returned a `404 Not Found`, the web renderer raised `WEBPAGE_LOAD_FAILED`, causing the entire `resolve()` transaction to terminate with an error.

This project explores a more defensive resolution flow.

## Improvements

### Graceful source failure handling

External webpage failures are converted into:

```json
{
  "status": "SOURCE_UNAVAILABLE",
  "score": "-",
  "winner": -1
}
```

instead of aborting the entire resolution transaction.

The market remains unresolved and can be retried later.

### Explicit resolution states

The resolver uses four normalized states:

- `RESOLVED`
- `UNRESOLVED`
- `SOURCE_UNAVAILABLE`
- `INVALID_RESULT`

This makes external-data and AI failures easier for applications to handle.

### Structured AI output validation

The resolver validates the LLM response before modifying contract state.

Checks include:

- required JSON fields;
- valid winner values (`-1`, `0`, `1`, `2`);
- numerical score format;
- consistency between score and winner;
- unresolved states must use `score = "-"` and `winner = -1`;
- malformed or inconsistent results become `INVALID_RESULT`.

### Reduced hallucination surface

Before asking an LLM to interpret a match, the contract checks whether both requested team names occur in the fetched source data.

If they are absent, the resolver returns:

```json
{
  "status": "UNRESOLVED",
  "score": "-",
  "winner": -1
}
```

without asking the model to infer a nonexistent matchup.

### Prompt/input hardening

Team names and webpage content are explicitly treated as untrusted data in the LLM prompt.

Constructor validation also rejects:

- empty team names;
- identical teams;
- excessive team-name length;
- newline/control-style characters;
- malformed date strings.

## Test Results

The contract was tested in GenLayer Studio using Normal (Full Consensus) mode.

| Test case | Original example | Hardened resolver |
| --- | --- | --- |
| Finished match: Fulham vs Chelsea, 2026-08-24 | Correctly resolved `2:3`, winner `2` | `RESOLVED`, `2:3`, winner `2` |
| BBC source returns 404 | `WEBPAGE_LOAD_FAILED`, transaction error | `SOURCE_UNAVAILABLE`, transaction succeeds |
| Existing BBC page but nonexistent matchup | `winner = -1`, remains unresolved | `UNRESOLVED`, `winner = -1` |
| Simple instruction-like text in team input | Did not override resolution instructions | Additional input and prompt hardening applied |

More detailed test evidence is documented in `TEST_RESULTS.md`.

## Example Successful Resolution

```json
{
  "status": "RESOLVED",
  "score": "2:3",
  "winner": 2
}
```

## Example Source Failure

```json
{
  "status": "SOURCE_UNAVAILABLE",
  "score": "-",
  "winner": -1
}
```

Unlike the original failure case, the contract does not permanently resolve the market and does not abort the transaction.

## Contract

The Intelligent Contract is available at:

`contracts/hardened_prediction_market.py`

It uses GenLayer Python SDK v0.2.16 and GenLayer's Equivalence Principle for consensus over normalized resolution results.

## Scope

This is a robustness experiment and reference implementation, not a claim that the original GenLayer example contains a security vulnerability.

The goal is to demonstrate defensive patterns for Intelligent Contracts that depend on:

- external webpages;
- nondeterministic AI outputs;
- validator consensus;
- potentially unavailable or incomplete evidence.

## Future Work

Potential extensions include:

- multi-source resolution instead of relying on a single webpage;
- automated GenLayer test-suite coverage;
- adversarial benchmark cases for conflicting sources;
- postponed/cancelled match handling;
- custom equivalence validation for semantically equivalent model outputs.

## License

MIT
