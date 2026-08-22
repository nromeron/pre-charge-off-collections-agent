# Eval Execution Status

**Status: definitions valid, execution blocked.**

---

## What works

The five eval definitions in [`evals/`](evals/) validate and persist correctly. The dashboard parses them and reports the expected turn counts — for example, `A1 happy path PTP` registers as 3 eval turns across 7 total turns, matching the definition.

## What does not

Every run stalls immediately:

- Status remains `Running...` indefinitely
- Duration `< 1s`
- **Total Turns: 0, Eval Turns: 0** — no turn is ever generated
- **Zero credit consumption**
- Runs are labeled `Transient Eval` rather than by their saved name

---

## Diagnosis

Bisected rather than guessed at:

**1 · Reduced to a minimal case.** A two-turn eval — one user message, one assistant turn with an `exact` judge and no expected content, no AI judge, no tool calls — reproduces the identical behavior. This rules out judge configuration, prompt complexity, and tool availability as causes.

**2 · Checked the API log.** Only `GET` requests returning `200`. **No `POST` creating a run appears at all.** The client never issues the request that would start execution.

**3 · Checked call logs.** Empty, as expected — evals are chat-type, not call-type, so absence there is not evidence either way.

**4 · Ruled out cost.** Credit balance is unchanged across runs, consistent with execution never beginning.

**Conclusion:** the failure is upstream of the eval definitions. The dashboard does not dispatch the run request. Non-ASCII characters in eval names were eliminated as a variable; behavior is unchanged.

**Not yet attempted:** submitting the same definitions via the REST API directly, bypassing the dashboard client. That would confirm whether the executor works when reached another way.

---

## Fallback

The [test matrix](test_matrix.md) was validated through structured review against the prompt rather than automated execution. That review surfaced five real defects, all corrected:

| Finding | Nature |
|---|---|
| Rule precedence conflict between the authority list and instruction-shaped input handling | Prompt defect — fixed with an explicit precedence rule |
| Disclosure turn exceeded the stated sentence limit with no documented exception | Prompt defect — fixed by merging the disclosure and discovery turns with a declared exception |
| Absolute prohibition on unlisted figures conflicted with split-payment authority | Prompt defect — fixed by permitting arithmetic within authorized options |
| Tool parameter lists in the prompt did not match the tool schemas | Consistency defect — aligned |
| One test case fed an unreachable second turn | Test design defect — reordered |

Two additional "failures" reported by the review were traced to incorrectly written assertions rather than prompt defects, which is itself a reason not to treat a review pass as equivalent to execution.

## Known limits of this approach

A structured review predicts behavior; it does not observe it. Specifically it cannot measure:

- **Non-determinism.** A case that passes twice and fails once is a failing case, and single-pass review cannot detect that.
- **Drift over length.** All cases are short. Instruction drift on a long prompt appears around turn fifteen, not turn three.
- **Real ASR degradation.** Text-based review assumes perfect transcription.
- **Interruption timing.** Barge-in latency requires audio.

One finding did emerge from live voice testing that no review caught: the ASR transcribed the consumer's name incorrectly, and the agent correctly declined to proceed on the mis-transcription and asked for explicit confirmation before disclosing anything.
