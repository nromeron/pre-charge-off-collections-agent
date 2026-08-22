# Regulatory Notes

Which framework governs what, and how each one shaped the design.

> Informational, not legal advice.

---

## The threshold question

Most collections guidance assumes a third-party debt collector. This scenario is not that.

**Chase collecting its own credit card, pre-charge-off, is the original creditor.** 15 U.S.C. §1692a(6) defines "debt collector" as one who collects debts owed to another, and excludes a creditor collecting debts it originated under its own name.

Getting this wrong in either direction is a failure. Assuming the FDCPA binds leads to reciting a mini-Miranda that is factually inaccurate. Assuming nothing binds ignores the frameworks that actually do.

---

## FDCPA — Fair Debt Collection Practices Act
**15 U.S.C. §1692 · 1977 · adopted here as policy, not obligation**

| Section | Rule | Design impact |
|---|---|---|
| §1692a(6) | Definition of "debt collector" | The threshold question above |
| §1692c(b) | No disclosure to third parties | Origin of invariant G1 and the `S0`/`S1` disclosure guard |
| §1692b | Third-party contact limited to location information, once | `T_THIRD_PARTY` behavior |
| §1692e(11) | Mini-Miranda | Replaced by an accurate purpose disclosure — see D2 |
| §1692g | Validation notice, 30-day dispute window | Informs dispute routing |

---

## Regulation F
**12 CFR Part 1006 · CFPB · effective November 2021 · adopted as policy**

Implements the FDCPA and converts principles into numeric rules.

- **§1006.14(b) — "7-in-7":** no more than seven call attempts per debt in seven consecutive days, and no call within seven days of a phone conversation. Dialer-layer control; noted so the agent never promises a callback cadence it cannot guarantee.
- **§1006.6(b)(1):** contact is presumed inconvenient before 8:00 a.m. or after 9:00 p.m. **in the consumer's local time**. Requires resolving local time from the customer file, not the area code alone.
- **§1006.2(j) — limited-content message:** a safe-harbor voicemail script that does not disclose the existence of a debt. The basis for the voicemail block.

---

## TCPA — Telephone Consumer Protection Act
**47 U.S.C. §227 · 1991 · binding**

Governs how you call, not what you say, and applies regardless of creditor status.

- Calls to wireless numbers using an artificial or prerecorded voice require **prior express consent**.
- **The FCC's February 2024 declaratory ruling treats AI-generated voices as "artificial" under the statute.** A voice agent is squarely within scope.
- Consent may be revoked by any reasonable means, and revocation must be honored.
- Statutory damages run $500–$1,500 **per call**, with no proof of actual harm required — which is why cease requests are treated as absolute rather than negotiable.

A cease request is therefore **two events**: an FDCPA-style cease of communication and a TCPA revocation of consent. They suppress different systems, which is why the logging tool carries a `scope` parameter.

Several states maintain their own mini-TCPA statutes with stricter requirements.

---

## FCRA — Fair Credit Reporting Act
**15 U.S.C. §1681 · binding**

Chase is a **furnisher** of credit information, and furnisher duties bind directly — this is not adopted policy.

- **§623(a)(8):** a direct dispute from the consumer triggers an investigation obligation.
- **§623(a)(3):** while unresolved, reporting must indicate the account is disputed.

This is the actual basis for dispute routing. A dispute is not a customer-service inconvenience — it is a data-quality event that puts the figure being collected in question. Collecting on a disputed balance before investigating means collecting money that may not be owed.

---

## UDAAP
**Dodd-Frank §1031 and §1036 · 12 U.S.C. §5531, §5536 · binding**

A general standard rather than an enumerated list, applying to original creditors without exception. It is the operative constraint wherever the FDCPA does not reach.

- **Unfair** — causes substantial, avoidable injury
- **Deceptive** — misleads a reasonable consumer
- **Abusive** — takes unreasonable advantage of a consumer's lack of understanding

*Abusive* is the most relevant prong here. Pressing someone who has just disputed a balance, or allowing a consumer to believe they are speaking with a human, both live in that territory.

---

## GLBA
**Gramm-Leach-Bliley Act · 1999 · binding**

Account details are **nonpublic personal information**. The Safeguards Rule requires technical and administrative controls; the Privacy Rule limits sharing. This is the regulatory basis for data minimization — holding only the last four digits rather than the full account number.

---

## Standards and state law

| Framework | Scope | Design impact |
|---|---|---|
| **PCI-DSS** | Card industry standard, not law | No voice capture of a PAN. No schema field for one. |
| **Illinois BIPA** — 740 ILCS 14 | Biometric identifiers | Voice authentication generates a voiceprint and triggers statutory damages of $1,000–$5,000 per violation with no actual harm required. Avoided by not using voice biometrics. |
| **CA Rosenthal Act** — Civil Code §1788 | Debt collection | Extends FDCPA-style duties to **original creditors** — the clearest reason the federal exemption does not mean the standards can be ignored. |
| **Bankruptcy Code** — 11 U.S.C. §362 | Automatic stay | A bankruptcy filing halts collection by operation of law. Basis for treating it as an absolute hard stop. |
| **SCRA** | Servicemember protections | Noted; not exercised in this scope. |
| **State AI disclosure laws** | Varies | Several require disclosure of automated agents on request. Reinforces D4. |

---

## Where each control lives

Compliance is not one layer, and claiming the prompt handles all of it would be inaccurate.

| Layer | Controls |
|---|---|
| **Pre-call (dialer)** | TCPA consent verification · DNC and internal suppression scrub · reassigned-number check · 7-in-7 frequency cap · local-time windowing |
| **Platform config** | Recording disclosure · AI disclosure · voicemail script · call duration bounds · barge-in responsiveness |
| **Prompt** | Disclosure guard · hard-stop routing · authority limits · input contract |
| **Tool schemas** | Payment gating · absence of any card-number field |
| **Post-generation** | Prohibited-phrase and PAN/SSN pattern filtering before TTS |
| **Post-call** | Structured compliance extraction · violation monitoring |

**The agent assumes the call was lawfully placed.** It cannot verify consent, suppression status, or local time, and simulating those checks inside the prompt would be theater rather than control.
