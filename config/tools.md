# Tool Definitions
<!-- companion to prompt/system_prompt.md -->

---

## Logging endpoint

The three function tools post to a single logging endpoint, referred to below as `[LOG_ENDPOINT]`. They are distinguishable by the tool name Vapi sends in the payload.

In a production deployment this is an authenticated audit-log service. For this build it is a request sink.

---

## Tool 1 — `transfer_to_specialist`

Type: **Transfer Call** (built-in — no server, no schema)

- **Tool Name:** `transfer_to_specialist`
- **Description:**
  > Transfer the call to a human specialist. Call this whenever the consumer requests a human, when the request is outside your authority, when payment verification is required, or when a hard stop requires human handling.
- **Destination:** any placeholder number for the demo — it is never actually dialed in chat mode

---

## Tool 2 — `log_dispute`

Type: **Function** · **Async: ON**

- **Tool Name:** `log_dispute`
- **Description:**
  > Record a consumer dispute. Call this IMMEDIATELY when the consumer disputes the debt, the balance, or states the account is not theirs. Call before any other response. Collection stops after this call.

**Parameters** (Visual builder → Add Property):

| Property | Type | Required | Description |
|---|---|---|---|
| `type` | string | ✅ | Either "identity" if the consumer says the account is not theirs or mentions fraud, or "amount" if they dispute the balance or payment history. |
| `detail` | string | ✅ | What the consumer disputed, in their own words. |

**Server URL:** `[LOG_ENDPOINT]`
**Timeout:** 10 sec
**Lock schema:** ON

---

## Tool 3 — `log_cease_communication`

Type: **Function** · **Async: ON**

- **Tool Name:** `log_cease_communication`
- **Description:**
  > Record a request to stop contact. Call this IMMEDIATELY when the consumer asks not to be contacted again. This suppresses future outbound contact and records revocation of consent.

**Parameters:**

| Property | Type | Required | Description |
|---|---|---|---|
| `scope` | string | ✅ | One of "this_number", "all_phone", or "all_channels". Use "all_phone" when the consumer does not specify. |
| `verbatim` | string | ✅ | The consumer's request, quoted exactly as they said it. |

**Server URL:** `[LOG_ENDPOINT]`
**Timeout:** 10 sec
**Lock schema:** ON

---

## Tool 4 — `schedule_promise_to_pay`

Type: **Function** · **Async: ON** *(see note below)*

- **Tool Name:** `schedule_promise_to_pay`
- **Description:**
  > Record a promise to pay. Call ONLY after the consumer has stated BOTH a specific amount AND a specific calendar date, and you have read both back to them. Never call with an approximate or open-ended date.

**Parameters:**

| Property | Type | Required | Description |
|---|---|---|---|
| `amount` | number | ✅ | The committed amount in US dollars. |
| `date` | string | ✅ | The committed date in YYYY-MM-DD format. Must be within 30 days. |
| `verbatim` | string | ✅ | The consumer's own words committing to this, quoted exactly. |

**Server URL:** `[LOG_ENDPOINT]`
**Timeout:** 10 sec
**Lock schema:** ON

**Messages** → optional start message: *"Okay."*

> Note: the prompt forbids stall phrases like "one moment please," because the agent has no lookup step to stall for. A tool-execution message is a different thing — it is spoken by the platform while a real call is in flight — but keep it minimal so the distinction is not blurred.

---

## Tool 5 — `process_payment`

**Do not create this tool.** Its absence is the demonstration that the `VERIFY_L2` gate is structural rather than instructional. `S5_PAYMENT` terminates in transfer, so it is never reachable.

---

## Note on async and confirmation

Setting all three logging tools to async means none returns a value, so the prompt cannot instruct the agent to confirm a tool result. Section 9 of the system prompt reflects this: the agent confirms only the amount and date the consumer stated and it read back, and never invents a confirmation number for a promise to pay — none exists.

---

## Verification order

Create and test in this order — each one that works de-risks the next.

1. **`transfer_to_specialist`** — built-in, no external dependency. If this fails, the problem is the assistant, not the tools.
2. **`log_cease_communication`** — chat: *"Yes."* → *"Don't call me again."* Watch webhook.site for the POST.
3. **The other two** — replicate the pattern once it's proven.

**What a success looks like:** the POST appears in webhook.site with the tool name and your arguments, *and* the agent keeps talking without a pause. If the agent hangs waiting, Async didn't take effect — check the toggle.

---

## Notes

**Why async on the logging tools.** A compliance log must not block the conversation. If the audit endpoint takes two seconds, the consumer should not hear dead air. Fire-and-forget is correct for logging and wrong for anything the agent must confirm.

**The honest caveat, which you raise first.** Fire-and-forget means a failed POST is lost silently. For a cease request that is unacceptable in production — it needs a durable queue with retries and delivery confirmation. The async choice is right for latency and incomplete for guarantees.

**On the encryption panel.** Vapi's own example of a field to encrypt is `payment.cardNumber`. The platform assumes you will transmit card data and offers to protect it. That is the right layer for that problem — but none of these schemas has a card field at all. The safest data is the data you never collect.
