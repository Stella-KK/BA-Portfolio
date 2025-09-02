# Business Requirements Document (BRD)
## Posting Rules → Journal Entries (JE) → General Ledger (GL)

> All examples are anonymized and synthetic for demonstration only.

### 1) Executive Summary / Background
Manual or semi-automated journal entries lead to longer period close, higher error rates, and weak auditability.  
This initiative maps financial events (e.g., **Contribution / Withdrawal / Fee**) to **Posting Rules** to automatically generate **balanced double-entry JEs** with full status tracking and audit trails, improving close speed and accuracy.

### 2) Business Objectives & KPIs
- **Automation:** ≥ **90%** of in-scope events auto-generate JEs  
- **Accuracy:** Unbalanced JE defect rate < **0.5%**; **100%** of missing-mapping cases are detected and alerted  
- **Efficiency:** Period close shortened by **≥ 0.5 day**; manual JEs reduced by **30%**  
- **Timeliness:** **T+1** completion of event-level reconciliation & exception reports  
- **Auditability:** End-to-end event → rule → JE traceability (with version and operator)

### 3) Scope
**In scope**
- Event ingestion (type/amount/currency/accountId/timestamp/attributes)
- Posting Rules lifecycle & versioning; match debit/credit accounts by event attributes
- JE generation (**DR == CR**), status transitions: `Pending / Posted / Error`
- Exceptions & alerts (missing rule, invalid account, invalid amounts, etc.)
- Daily reconciliation (trial balance, event-level matching)
- Audit logs & operations dashboards (automation rate, exception distribution, top missing rules)

**Out of scope**
- Final posting to external ERP (mock/interface only in this portfolio)
- Real production data or sensitive configurations (synthetic data only)
- Complex FX/hedging/valuation engines (basic multi-currency only for demo)

### 4) High-Level Target Process
`Financial Event → Rule Match → JE Build → Validate → Posted / Error → Reconciliation → Reporting`

- Events are matched by type and attributes (product, channel, tax type, etc.) to a single effective Posting Rule (by priority).
- System builds JE lines (debit/credit) and runs balance/validity checks.
- Success → `Posted`; failure (e.g., invalid account/missing rule) → `Error` with reason.
- Daily trial balance and event-level reconciliation feed Finance Ops review.

> A BPMN diagram is provided as `bpmn.png` in the case folder.

### 5) Key Business Rules (non-technical phrasing)
> Examples only; replace account names with your GL catalog in real projects.

| Rule ID | Event Type           | Condition (example)         | Debit (DR)              | Credit (CR)                 | Note                                   |
|--------:|----------------------|-----------------------------|-------------------------|-----------------------------|----------------------------------------|
| R-001   | Contribution         | amount > 0                  | Cash/Bank               | Contribution Payable        | Cash in; recognize payable to investor |
| R-002   | Contribution-Post    | period end / settlement     | Contribution Payable    | Investor Account/Unit       | Move payable to investor equity        |
| R-011   | Withdrawal           | amount > 0                  | Investor Account/Unit   | Cash/Bank                   | Investor redemption                     |
| R-021   | Fee                  | feeType = Admin             | Fee Expense             | Cash/Bank                   | Policy-dependent revenue/expense side   |
| R-031   | FX Revaluation       | currency ≠ book currency    | FX Loss                 | FX Gain                     | Period-end reval (demo)                 |

**Common validations**
- Non-negative amounts; a line cannot have both debit and credit
- **Balance:** Σ(debit) = Σ(credit)
- Only **effective, active** GL accounts may be posted
- FX: if event currency ≠ book currency, convert using day-T rate with standard rounding
- Errors (missing rule/invalid GL) set status to `Error` with reason codes & details

### 6) Business Capabilities (high-level)
- **Accounts (Chart of Accounts):** Maintain usable GL accounts/hierarchy, control manual posting, effective dating; JE foundation.  
- **Posting Rules:** For each event type + attributes, match exactly one **Active** rule (by priority) that outputs DR/CR templates; rules are versioned and approved.  
- **Financial Events:** System-generated business events (not user-created), e.g., Contribution/Withdrawal/Fee, with minimal attributes to drive matching and audit trail.  
- **Journal Entries:** Auto/manual JEs; status `Posted/Error`; **Reversal** supported to offset erroneous JEs; provide period trial balance & exception lists.

> Field-level definitions, matching expressions, status machine, and error codes are documented in the **FRD/Design** (not in this BRD).

### 7) Reporting & Reconciliation (business outcomes)
- **Daily Trial Balance** (Σ DR/CR by account; difference = 0)  
- **Event-Level Reconciliation** (event amount vs. JE net amount)  
- **Exception Lists:** Missing mapping, invalid GL, unbalanced, missing FX, etc.  
- **Ops Dashboard:** Automation rate, defect rate, close duration, top missing/failed rules

### 8) Compliance & Controls
- **Privacy & Retention:** PIPEDA-aligned minimization/purpose limitation; audit trail retained **≥ 7 years**  
- **Segregation of Duties:** Rule maintenance vs. approval; four-eyes principle for production changes  
- **Change Traceability:** Capture operator, timestamps, versions for rules/accounts/FX sources

### 9) Stakeholders & RACI (example)
| Role              | Responsibility                          | RACI |
|-------------------|------------------------------------------|:---:|
| Finance Director  | Business goals & final sign-off          |  A  |
| Product / BA      | Requirements, rule design, reconciliation|  R  |
| Engineering Lead  | Technical solution & delivery            |  R  |
| QA                | Test strategy & defect management        |  C  |
| Accounting Ops    | UAT, daily reconciliation & operations   | C/I |

### 10) Business Acceptance (UAT)
- Pass rate ≥ **95%**, all Sev-1/2 defects closed; KPIs met  
- Scenario coverage: Contribution / Withdrawal / Fee / FX; reversals; missing rule; invalid account; unbalanced; missing FX

```gherkin
Feature: Automated Journal Entries (Business)
  Scenario: Contribution 100 CAD produces a balanced JE
    Given a valid Contribution event
    When the system processes the event
    Then a posted JE exists with total debit = total credit
    And the event→rule→JE audit trail is available
```

### 11) Assumptions & Dependencies
- GL chart of accounts is maintained by Finance master-data and synchronized daily.
- Events meet minimum data completeness (see **FRD**).
- Trusted FX rate source available on day T (latency ≤ 15 minutes).

### 12) Risks & Mitigations
| Risk                     | Impact                       | Mitigation                                                    |
|--------------------------|------------------------------|---------------------------------------------------------------|
| Misconfigured rules      | Unbalanced/incorrect JEs     | Pre-deployment rule checks; approval workflow; rollback plan  |
| Poor input data quality  | Batch failures                | Inbound validation; isolation; replay mechanism               |
| Inconsistent FX handling | Reconciliation differences    | Single FX source; unified rounding; periodic audits           |
| Period-end load spikes   | SLA degradation               | Pre-generation; batch windows; performance baseline & scale   |

### 13) Change Control
- Any change affecting Posting Rules/GL requires a change request with impact assessment and rollback plan, approved by **Finance + Product + Engineering**.
- Post-change monitoring and targeted sampling within **24 hours**.

### 14) Glossary
- **Posting Rule**: Business rule mapping an event to accounting entries.  
- **JE (Journal Entry)**: Accounting document composed of debit/credit lines.  
- **Trial Balance**: Account-level aggregation of DR/CR to validate zero difference.  
- **Audit Trail**: Traceability from event to rule version and resulting JE.


