# Outbound Collections Voice Agent — Pre-Charge-Off

A compliance-first voice agent for first-party credit card collections, built on Vapi.

**Scenario.** Outbound call to James Carter. Chase credit card, 60 days past due, $189.00 past due against a $3,847.22 balance. Goal: resolve the delinquency before charge-off.

---

## The framing decision everything else follows from

Chase collecting its own credit card is the **original creditor**, not a debt collector. The FDCPA's own definition — 15 U.S.C. §1692a(6) — excludes a creditor collecting debts it originated, under its own name. So federally, **the FDCPA does not apply here.**

That is not a loophole to exploit. It changes which rules actually bind:

| Framework | Applies? |
|---|---|
| FDCPA / Reg F | **No** federally — adopted as an internal policy floor |
| UDAAP (Dodd-Frank §1031, §1036) | **Yes**, without exception |
| State statutes (e.g. CA Rosenthal Act) | **Yes** — several reach first-party creditors |
| TCPA | **Yes**, regardless of creditor status |
| FCRA furnisher duties | **Yes** — Chase reports to bureaus |
| GLBA | **Yes** — this is NPI |

The agent therefore does **not** recite the literal mini-Miranda ("this is a debt collector"), because that statement would be inaccurate. It delivers an equivalent purpose disclosure instead. See [D2](docs/design_decisions.md#d2--purpose-disclosure-rather-than-literal-mini-miranda).

---

## Four decisions worth reading

**1 · Anything legally required is removed from the model.**
The recording notice and AI disclosure are `firstMessage` — a static string that fires before the model generates a token. The voicemail script is `voicemailMessage`. An LLM told to say something verbatim will say it correctly *almost* always, and "almost" is not a compliance posture.

**2 · No debt information exists before identity is confirmed.**
The opening greeting, recording notice, and AI disclosure are all safe if a third party answers — none of them reveal that a debt exists. Everything that does is gated behind right party contact. This holds even when a third party volunteers that they already know about the account: anyone can say that, and familiarity is not verification.

**3 · The payment gate is structural, not instructional.**
`process_payment` requires a second identity factor. None is provisioned, so the payment path terminates in a transfer — and the tool is deliberately never created. No schema anywhere has a card number field, which makes PAN capture impossible by construction rather than by prohibition.

**4 · Success is defined as zero violations, not dollars collected.**
The evaluation rubric scores a call that collected nothing and violated nothing as a 10, and a call that collected the full balance but disclosed the debt to an unverified party as a 0. In a regulated vertical, an agent optimized for collection rate is an agent optimized to eventually produce a violation.

---

## Architecture

The prompt is a **state machine with interrupts**, not a linear script. Linear scripts break the moment the consumer leaves the rails.

```
S0_OPEN → S1_RPC → S2_DISCLOSE → S4_NEGOTIATE → S5_PAYMENT / S6_PTP → S7_CLOSE
```

Ten terminal states are reachable from **any** point in the flow: cease, dispute, attorney, bankruptcy, deceased, wrong party, third party, voicemail, transfer, bad time. These are evaluated *before* the collection objective on every turn.

Three block types, deliberately separated:

- **Verbatim** — legally required language, word for word, tested with exact-match assertions
- **Guidance** — tone and negotiation, freely paraphrased
- **Intent** — semantic triggers with non-exhaustive examples, never keyword lists

The asymmetry rule is written into the prompt explicitly: *when unsure whether something is a hard stop, treat it as a hard stop. A false positive costs one collection attempt; a false negative is a statutory violation.*

---

## Repository

| Path | Contents |
|---|---|
| [`prompt/system_prompt.md`](prompt/system_prompt.md) | The system prompt |
| [`docs/design_decisions.md`](docs/design_decisions.md) | Every decision, with rationale and trade-offs |
| [`docs/regulatory_notes.md`](docs/regulatory_notes.md) | Which framework governs what, and why |
| [`config/assistant_config.md`](config/assistant_config.md) | Vapi assistant configuration and parameter rationale |
| [`config/tools.md`](config/tools.md) | Tool definitions and schemas |
| [`testing/test_matrix.md`](testing/test_matrix.md) | 30 test cases, 20 blocking |
| [`testing/evals/`](testing/evals/) | Vapi eval definitions |
| [`testing/EVALS_STATUS.md`](testing/EVALS_STATUS.md) | Execution status and platform issue |

---

## Testing

Ten **global invariants** (G1–G10) are checked across all 30 cases rather than only where they are the target — G1, the disclosure guard, can break in a dispute case as easily as in a third-party case. Each maps 1:1 to a boolean in the post-call structured output schema, so the test matrix and production monitoring measure the same things.

Judge selection is itself a design decision: **verbatim blocks get exact-match judges, conversational behavior gets LLM judges.** Grading a legally required disclosure with an LLM means a model decides whether a mandated sentence was said correctly. That is not a control.

Release criterion: all 20 P0 cases pass. No exceptions.

Eval definitions validate and save correctly, but runs stall at zero turns without consuming credits. See [`testing/EVALS_STATUS.md`](testing/EVALS_STATUS.md) for the diagnosis.

---

## What this does not solve

Stated plainly, because a known gap is a design position and an unknown one is a defect.

**Identity verification is weak.** The L1 gate is name confirmation only. Asking for the last digits of the account number was considered and rejected: that number is printed on every statement mailed to the address, so it verifies mail access rather than identity. Production needs a secondary identifier — ZIP or SSN last four — checked against the customer file. The brief provided neither, and no data was invented to paper over it.

A related rule applies whatever the identifier is: **ask, never confirm.** The agent asks *"what's your ZIP code?"*, never *"is your ZIP code 80202?"* — confirming hands the value to whoever is on the line.

**The agent never takes a payment.** By design. Promise-to-pay completes end to end; payment terminates in transfer.

**Compliance logging is fire-and-forget.** Async is correct for latency — an audit write should never block the conversation — but a failed POST is lost silently. Production requires a durable queue with retries and delivery confirmation.

**Pre-call controls are out of scope.** TCPA prior express consent, DNC and reassigned-number scrubs, the Reg F 7-in-7 frequency cap, and consumer-local-time windowing are all dialer-layer controls. The agent assumes the call was lawfully placed; it has no way to verify that, and pretending otherwise inside the prompt would be theater.

**Voice-specific behavior is only partly verified.** Barge-in latency, ASR degradation, and silence handling need real audio. The prompt and configuration address them; text testing does not prove them.
