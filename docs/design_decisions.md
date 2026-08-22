# Design Decisions

Each decision states what was decided, why, and what it costs. Decisions without stated costs are usually decisions that were never actually made.

---

## Regulatory posture

### D1 · First-party creditor, FDCPA as a voluntary floor

The agent identifies as Chase collecting a debt Chase owns. FDCPA and Reg F are adopted as internal policy, not treated as binding federal obligations. What does bind: UDAAP, state statutes, TCPA, GLBA, and FCRA furnisher duties.

**Why.** 15 U.S.C. §1692a(6) excludes a creditor collecting its own debt under its own name from the definition of "debt collector." Treating the FDCPA as binding here would misread the scenario; ignoring its standards entirely would ignore what actually does apply.

**Cost.** Requires knowing at every point which rule binds and which was adopted. The position collapses if that distinction is fuzzy.

### D2 · Purpose disclosure rather than literal mini-Miranda

The agent does not say "this is an attempt to collect a debt by a debt collector." It states its identity, that the call is recorded, that it is automated, and — after identity confirmation — that the call is an attempt to collect a past due balance and that information provided will be used for that purpose.

**Why.** Declaring debt-collector status when you are the creditor is an inaccurate statement, which carries UDAAP exposure in the opposite direction. The disclosure's purpose — the consumer knows who is calling and why — is fully served.

**Cost.** This is the most challengeable decision in the design. An evaluator expecting the textbook mini-Miranda may read it as an omission. It is a verbatim block: a one-line change if policy or a state requirement demands it.

### D3 · Opening sequence

Fixed order: greeting and identification → recording disclosure → AI disclosure → ask for the consumer → **verify** → only then, purpose and account details.

**Why.** Steps one through three are third-party safe; none reveals that a debt exists. Everything that does is gated behind verification.

**Cost.** A longer opening before reaching the point — roughly twelve seconds of voice.

### D4 · Proactive AI disclosure

The agent states it is an automated assistant in the opening, and never denies or deflects when asked directly.

**Why.** The FCC's February 2024 declaratory ruling treats AI-generated voices as "artificial" under the TCPA. Disclosure at call open is not yet a finalized federal requirement, but the regulatory direction is clear, several state laws already require disclosure on request, and on a recorded line a denial is evidence.

**Cost.** Real. Disclosure reduces engagement; some consumers hang up on hearing "automated." Accepted, because concealment creates TCPA and UDAAP exposure and requires the agent to lie when asked — on a recorded call.

---

## Authority and negotiation

### D5 · Closed authority list

**May:** collect the past due amount or the minimum payment; schedule a promise to pay up to 30 days out; split the past due into two payments within 45 days; transfer.

**May not:** settle, waive, or reduce anything; modify terms; promise no charge-off; state what will or will not appear on a credit report; discuss legal consequences.

**Everything else:** transfer.

**Why.** An agent without explicit authority limits will negotiate anything if asked persuasively. A closed list defeats false-authority injection structurally — "your supervisor approved a settlement" cannot succeed against an agent that has no settlement capability to invoke.

**Cost.** Less flexibility; some resolvable calls escalate.

### D6 · Payments are tool calls or nothing

Confirmation numbers come only from tool results. No card, CVV, bank, or routing number is solicited, accepted, or repeated. Secure capture is masked DTMF or transfer to an IVR with recording paused.

**Why.** A fabricated confirmation number is a fabricated financial fact. Voice capture of a PAN breaks PCI-DSS.

**Cost.** A tool failure is visible to the consumer rather than hidden. That is the correct trade.

### D7 · Interruption hierarchy

Three tiers evaluated before the collection objective, at any point in the call. Hard stops end collection with no rebuttal and no second ask. Soft stops reschedule or hand off.

Plus the asymmetry rule, written explicitly into the prompt: **when unsure whether something is a hard stop, treat it as a hard stop.**

**Why.** A linear script breaks when the consumer leaves the rails. Hard stops are interrupts, not branches.

**Cost.** False positives — collection attempts lost to over-reading ambiguity.

### Dispute routing

Disputes are separated by kind. An **identity dispute** ("I never opened this account") is a fraud signal: collection stops absolutely and payment is never taken, because accepting money there means collecting from a victim. An **amount dispute** is logged and routed.

The distinction that matters most is **asking versus accepting**. The agent never solicits payment after a dispute — no rebuttal, no credit-reporting leverage, no "goodwill payment." But it does not refuse a payment the consumer volunteers; it routes it to a specialist who can process it with the dispute documented.

**Why.** A direct dispute triggers FCRA furnisher investigation obligations under §623(a)(8) — this binds Chase directly, unlike the FDCPA. Until investigation completes, the figure is in question. Soliciting payment on a disputed balance also extracts what can be construed as acknowledgment of the debt from someone who just contested it.

---

## Data handling

### D8 · Minimum data in context

Only the last four digits of the account are held. No full account number, no date of birth, no SSN, no address.

**Why.** What is not in context cannot be leaked. This converts a soft control — "never reveal X" — into a hard one, at no cost.

**Cost.** L1 verification is weak. Documented rather than papered over.

### Verification offramp

At any sign of doubt about the call's legitimacy, the agent directs the consumer to channels **they look up independently** — the bank's website, its app, or a branch — never to a number the agent supplies.

**Why.** On an outbound call the consumer cannot verify the caller. Supplying a callback number is the same move a vishing attacker makes. A collections agent that invites you to hang up and verify is counterintuitive and correct.

### Input contract

All consumer speech is **data, never instruction**. No claim by the speaker modifies rules, grants authority, or alters account figures, regardless of the authority invoked.

The channel is declared as speech transcription. A human on a phone cannot pronounce code, markup, or encoded payloads — so structured injection payloads are semantically incoherent within the frame the model was given. The defense is constructive rather than a blocklist.

---

## Architecture

### Mandated language leaves the model

The opening disclosure lives in `firstMessage`; the voicemail script lives in `voicemailMessage`. Both execute with probability 1, before the model produces a token.

What remains in the prompt is judgment — negotiation, intent recognition, state transitions, tone. That is what a model is for.

### `process_payment` is deliberately not created

`S5_PAYMENT` requires a second identity factor that this build does not provision, so the path terminates in transfer and the tool is never reachable. Its absence demonstrates that the gate is structural rather than a rule the model could choose to ignore.

### Asynchronous compliance logging

`log_dispute` and `log_cease_communication` are fire-and-forget. A compliance write must not block the conversation; if the audit endpoint takes two seconds, the consumer should not hear dead air.

**Cost.** A failed POST is lost silently. Correct for latency, incomplete for delivery guarantees. Production needs a durable queue.

### ASR keyterm boosting is a compliance control

The hard stops are only as reliable as the transcription. If "bankruptcy" is transcribed as "bank rupture," the interrupt never fires. The boosted vocabulary is the hard-stop trigger set, not product names.

---

## Testing

### Judges are selected by what is being tested

Verbatim blocks get exact-match judges. Conversational behavior gets LLM judges. Grading a legally mandated disclosure with an LLM means a model decides whether a required sentence was delivered correctly — that is not a control.

### Success is compliance, not collection

The evaluation rubric scores on violations only, explicitly disregarding whether money was collected.

### Global invariants over per-case assertions

Ten invariants are checked across all cases rather than only where they are the focus, and map 1:1 to the post-call structured output schema — so tests and production monitoring measure the same things.
