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

A second review pass, reading all artifacts in parallel rather than the prompt alone, found a further class of defect that neither single-artifact review nor execution would have caught: **corrections that were applied to one file and not propagated to the others.** Merging two states in the prompt left a dangling reference to the removed state in a downstream section and in the test matrix, which meant a compliant agent would have failed a blocking test case. Three similar cross-artifact contradictions were found and closed:

| Contradiction | Resolution |
|---|---|
| Removed state still referenced in the negotiation step and in the matrix | References purged; the affected case rewritten against the merged state |
| Voicemail script described as both agent-spoken and platform-delivered | Terminal now instructs silence; the platform delivers it |
| Payment tool described as both deliberately absent and deliberately wired-and-failing | Unified on absence, which is the position the prompt and README argue |
| Turn-length invariant contradicted the documented disclosure exception | Exception written into the invariant |

One functional defect also surfaced: no current date was injected, so relative dates like "the fifteenth" could not be resolved to an ISO date or validated against the 30-day window. This would have broken the primary happy path on the first real call. The account block now carries a date variable.

**Reviewing artifacts in isolation hides contradictions between them.** That is now part of the process, not an afterthought.

### Third pass — corrections are not exempt from review

A third pass reviewed the corrections themselves. Two of its three substantive findings were **defects introduced by the previous round's fixes**, not survivals from the original:

**A new stacking rule created a disclosure leak.** Resolving simultaneous hard stops by choosing a single "winning" terminal ranked bankruptcy above third-party contact — but third-party contact is the most disclosure-restrictive state there is, so the rule contradicted its own stated principle. Concretely: a spouse answering and reporting a bankruptcy would have received the bankruptcy script, which references stopping *collection activity* — disclosing to an unverified party. The rule also suppressed the lower-ranked terminal's *actions*, so a cease request raised by that third party would never have been logged.

The fix separates two axes that should never have been collapsed: **speech is bounded by the most restrictive guard that applies; actions accumulate across every triggered stop.** Test case B8b was added, since a rule without a test is not a control.

**A channel-detection clause depended on an unobservable fact.** Anti-spoofing text conditioned behavior on "in a text-only channel" — but Section 1 declares unconditionally that this is a live phone call, and no observable signal distinguishes the two. This is the same defect class as an earlier time-of-day trigger the agent had no clock to evaluate. Replaced with a decidable heuristic and a build-variant note in the appendix.

A third finding was a stale instruction: one document still described a prompt change as pending after it had been applied.

**The pattern is the lesson.** Fixes are written under time pressure, against a narrower context than the original, and carry the false confidence of being corrections. Reviewing them with the same rigor as the original artifact is not optional — it is where two of these three defects came from.

### Fourth pass — two more

**A rule was incomplete, not wrong.** The stacking rule separated speech from action, but a call has a third property neither axis covers: it can only end one way. Actions accumulate; disposition cannot. A bankruptcy report calls for transfer and third-party contact calls for ending the call, and both cannot happen. The rule now carries an explicit disposition clause — a holder-status report from a third party transfers, because the receiving team can handle third-party contact and can capture a report that has no logging tool of its own. That also unifies two test cases that previously resolved differently for no stated reason.

The test case for the rule had inherited the ambiguity: it asserted "routes appropriately," which is not binary, in a matrix whose own scoring rule says every case is binary. A non-binary assertion in a binary suite is a case that cannot fail.

**A count was inherited rather than computed.** Three different case totals were circulating across two files. Counting from the matrix itself gives 31 cases, 24 blocking — the figures in the README were derived by incrementing an earlier number that had never been verified either. Any figure appearing in more than one artifact is now computed from the source artifact rather than carried forward.

Neither finding was a compliance defect. Both were completeness defects, which is what remains once the compliance defects are gone.

## Known limits of this approach

A structured review predicts behavior; it does not observe it. Specifically it cannot measure:

- **Non-determinism.** A case that passes twice and fails once is a failing case, and single-pass review cannot detect that.
- **Drift over length.** All cases are short. Instruction drift on a long prompt appears around turn fifteen, not turn three.
- **Real ASR degradation.** Text-based review assumes perfect transcription.
- **Interruption timing.** Barge-in latency requires audio.

One finding did emerge from live voice testing that no review caught: the ASR transcribed the consumer's name incorrectly, and the agent correctly declined to proceed on the mis-transcription and asked for explicit confirmation before disclosing anything.
