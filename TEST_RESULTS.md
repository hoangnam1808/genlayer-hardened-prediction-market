# GenLayer Hardened Prediction Market — Test Results

## Test Environment

The tests below were executed in **GenLayer Studio** using:

* Execution mode: `Normal (Full Consensus)`
* GenLayer Python SDK: `v0.2.16`
* External resolution source: BBC Sport football scores/fixtures
* Original contract: GenLayer `football_prediction_market.py` example
* Hardened contract: `contracts/hardened_prediction_market.py`

The objective was to compare normal resolution behavior and failure handling between the original example and the hardened implementation.

---

## Summary

| Case                                      | Original GenLayer example | Hardened resolver               |
| ----------------------------------------- | ------------------------- | ------------------------------- |
| Finished real match                       | ✅ Correct resolution      | ✅ Correct resolution            |
| Resolution source returns 404             | ❌ Transaction error       | ✅ Graceful `SOURCE_UNAVAILABLE` |
| Valid source page but nonexistent matchup | ✅ Remains unresolved      | ✅ Explicit `UNRESOLVED`         |

The hardened version preserves the successful behavior of the original example while handling an unavailable external source without aborting the resolution transaction.

---

## Test 1 — Finished Match Baseline

### Input

* `game_date`: `2026-08-24`
* `team1`: `Fulham`
* `team2`: `Chelsea`

Expected football result:

```json
{
  "score": "2:3",
  "winner": 2
}
```

### Original GenLayer Example

Result:

```json
{
  "score": "2:3",
  "winner": 2
}
```

Transaction status:

`FINALIZED / SUCCESS`

Explorer:

https://explorer-studio.genlayer.com/tx/0x409ce8a47d3d0596f6a9b8c701d0484ff2ddecf022b245ffd4cc6a752d7a19d2

### Hardened Resolver

Result:

```json
{
  "score": "2:3",
  "status": "RESOLVED",
  "winner": 2
}
```

Transaction status:

`ACCEPTED / SUCCESS`

Explorer:

https://explorer-studio.genlayer.com/tx/0xfcfe98ccd2ae50ee695ac8cdaf956a01550b12d4dd077ca189736353720b05de

### Observation

The hardened resolver preserves the expected happy-path behavior.

Adding validation and failure handling did not prevent normal market resolution.

---

## Test 2 — External Resolution Source Returns 404

### Input

* `game_date`: `2099-01-01`
* `team1`: `Fulham`
* `team2`: `Chelsea`

The generated BBC resolution URL does not provide a valid fixture page for this request.

### Original GenLayer Example

The call to:

```python
gl.nondet.web.render(market_resolution_url, mode="text")
```

raised a nondeterministic web exception.

Observed error:

`WEBPAGE_LOAD_FAILED`

The BBC response was:

`404 Not Found`

Transaction result:

`FINALIZED / ERROR`

Explorer:

https://explorer-studio.genlayer.com/tx/0xf0868b68135d88e6d022d4a8a1b219ac0033469214e438bd4b6af104667fa750

The transaction produced no successful resolution output.

### Hardened Resolver

The same logical test case returned:

```json
{
  "score": "-",
  "status": "SOURCE_UNAVAILABLE",
  "winner": -1
}
```

Transaction result:

`FINALIZED / SUCCESS`

Explorer:

https://explorer-studio.genlayer.com/tx/0xabf7099200be41302c61240822a1e78263b6ab16c31c07793b8481bb49db8363

The resulting contract state remained unresolved:

```json
{
  "has_resolved": false,
  "score": "",
  "status": "SOURCE_UNAVAILABLE",
  "winner": 0
}
```

### Observation

This is the main behavioral improvement demonstrated by the project.

Instead of propagating an external webpage failure and aborting `resolve()`, the hardened implementation converts the unavailable source into a deterministic application-level state.

The market is not permanently resolved and can be retried later.

---

## Test 3 — Existing Source Page, Nonexistent Matchup

### Input

* `game_date`: `2026-08-24`
* `team1`: `Crypto Lab FC`
* `team2`: `GenLayer United`

The BBC page itself is reachable, but the requested teams do not form a real matchup on that page.

### Original GenLayer Example

Result:

```json
{
  "score": "-",
  "winner": -1
}
```

Transaction result:

`FINALIZED / SUCCESS`

Explorer:

https://explorer-studio.genlayer.com/tx/0x6e52c04b12b6e41f3f13ddd18305de35ce7d57683e483b9810c00825b824ace5

The market remained unresolved.

### Hardened Resolver

Result:

```json
{
  "score": "-",
  "status": "UNRESOLVED",
  "winner": -1
}
```

Transaction result:

`ACCEPTED / SUCCESS`

Explorer:

https://explorer-studio.genlayer.com/tx/0xf68a698cd4dddc9111eb63d2cdeb70dc28c4c3c84b0082551735e0047edf20a9

### Observation

Both contracts avoid hallucinating a football result for a nonexistent matchup.

The hardened version additionally normalizes the outcome into the explicit `UNRESOLVED` state.

It also performs a deterministic source-presence check before asking an LLM to interpret the match, reducing unnecessary nondeterministic execution when the requested teams are absent from the source.

---

## Additional Adversarial Exploration

A simple instruction-like payload was also supplied through a team-name constructor input against the original example.

The model did not follow the injected instruction and correctly returned an unresolved result.

This was not treated as a discovered vulnerability.

The hardened implementation nevertheless reduces this attack surface by:

* validating constructor inputs;
* limiting team-name length;
* rejecting newline/control-style characters;
* explicitly identifying team names and webpage content as untrusted data in the LLM prompt;
* validating structured model output before changing contract state.

---

## Main Finding

The original GenLayer example performs correctly for the tested normal match and nonexistent-matchup cases.

The reproducible robustness issue identified during testing is narrower:

> An unavailable external resolution webpage can cause `gl.nondet.web.render()` to raise `WEBPAGE_LOAD_FAILED`, terminating the original example's `resolve()` transaction.

The hardened resolver demonstrates one possible defensive pattern:

```text
External source failure
        ↓
Catch nondeterministic exception
        ↓
Normalize result
        ↓
SOURCE_UNAVAILABLE
        ↓
Consensus
        ↓
Transaction succeeds
        ↓
Market remains unresolved
```

This project should therefore be interpreted as a **robustness and developer-experience improvement**, not as a claim of a security vulnerability in GenLayer.

## Future Testing

Potential follow-up work includes multi-source disagreement, postponed or cancelled fixtures, malformed AI outputs, semantic-equivalence testing across different validators, automated GenLayer test-suite coverage, and retry/recovery behavior after temporary source outages.
