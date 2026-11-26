# Ascend Retail — Returns Concierge

An Agentforce Service Agent for **Ascend Retail** (the online home-and-lifestyle store in the Ascend Group) that helps shoppers check whether an item can be returned and, when eligible, open a single return request. The agent is deliberately narrow: it **verifies the shopper**, **reads shipment/return-eligibility data**, and **creates exactly one return Case** per eligible shipment. It **never issues, calculates, approves, or promises a refund** — refunds are handled by a separate team.

This is a Salesforce DX project. The agent lives in an `aiAuthoringBundle` and delegates all data work to three autolaunched Flows.

---

## Architecture at a glance

```
Shopper ──▶ Agent Router ──┬─▶ Shopper Verification ──▶ flow: AWT_Verify_Shopper_By_Order_and_Email
                           ├─▶ Return Eligibility Lookup ─▶ flow: AWT_Check_Return_Eligibility
                           ├─▶ Create Return Request ─────▶ flow: AWT_Create_Return_Case
                           ├─▶ General FAQ ──────────────▶ Knowledge (AnswerQuestionsWithKnowledge)
                           ├─▶ Escalation ───────────────▶ Omni-Channel flow → human agent
                           └─▶ Ambiguous Question (asks one clarifying question)
```

- **Agent definition:** [Ascend_Retail_Returns_Concierge_2.agent](force-app/main/default/aiAuthoringBundles/Ascend_Retail_Returns_Concierge_2/Ascend_Retail_Returns_Concierge_2.agent)
- **Model:** `sfdc_ai__DefaultGPT41`
- **Type:** `AgentforceServiceAgent` (shopper-facing, messaging)

---

## The agent

### Routing logic

The **Agent Router** runs a strict, ordered check every time it receives control:

1. **Out-of-scope check.** Damaged / broken / defective / missing items, or payment / charge / chargeback disputes, and any explicit request for a human (or signs of anger/distress) route straight to **Escalation**. These must never reach Create Return Request.
2. **Verification check.** Anything that reads shipment data or creates a return requires `isVerified == True`. If not verified, route to **Shopper Verification** first. Purely informational policy questions do not require verification.
3. **Route by intent:**
   - **General FAQ** — informational questions about the returns policy, refund timelines, or final-sale items.
   - **Return Eligibility Lookup** — "can I return this order/item?" (requires verification).
   - **Create Return Request** — "start / submit a return" (requires verification + a known eligible shipment).
   - **Ambiguous Question** — intent unclear; asks one clarifying question.

### Subagents

| Subagent | Purpose | Backing action |
|---|---|---|
| **Shopper Verification** | Verifies identity by **order number + the email on the order** (no OTP). Locks out after 3 failed attempts → Escalation. | `AWT_Verify_Shopper_By_Order_and_Email` |
| **Return Eligibility Lookup** | For a verified shopper, reports per-shipment status, delivery date, and eligibility. Handles split orders — explains which shipment is eligible and why the others aren't. | `AWT_Check_Return_Eligibility` |
| **Create Return Request** | Creates **exactly one** return Case for the single identified eligible shipment. Re-validates all rules server-side. | `AWT_Create_Return_Case` |
| **General FAQ** | Answers policy questions strictly from approved Knowledge articles, with citations. No verification required. | `AnswerQuestionsWithKnowledge` |
| **Escalation** | Hands off to a human specialist via an Omni-Channel flow for out-of-scope requests. | `@utils.escalate` |
| **Ambiguous Question** | Asks a single clarifying question, then lets the router re-route. | — |

### Guardrails

- **No refunds, ever.** The agent must never issue, approve, calculate, estimate, or promise a refund amount or date. Refund questions are answered only as general policy (from the Refund timelines article) and refund timing is governed by the published policy.
- **One verified shopper per session.** Once verification starts, the agent never substitutes a different shopper, and never reveals whether an order number or email exists.
- **One return per shipment.** Only the single identified eligible shipment may be returned, and never more than one.
- **Server is the source of truth.** Every rule is re-validated inside the Flow before a Case is created — the agent's view may be stale.

### Key session variables

`isVerified`, `orderNumber`, `email`, `contactId`, `contactName`, `eligibleShipmentId`, `eligibilitySummary`, `ineligibleReason`, `returnCaseNumber`, `verificationMessage`, `verificationFailures` (locks out at 3). Linked variables (`ContactId`, `EndUserId`, `EndUserLanguage`, `RoutableId`, `VoiceCallId`) come from the messaging session.

---

## The flows

All three are **autolaunched flows** (API `66.0`) invoked as agent actions.

### 1. `AWT_Verify_Shopper_By_Order_and_Email`
[AWT_Verify_Shopper_By_Order_and_Email.flow-meta.xml](force-app/main/default/flows/AWT_Verify_Shopper_By_Order_and_Email.flow-meta.xml)

Verifies a shopper by matching the **order number** to a `Shop_Order__c` shipment and comparing the order's `Email__c` to the supplied email (case-insensitive, trimmed).

- **Inputs:** `orderNumber`, `email`, `failedAttempts`
- **Outputs:** `success`, `contactId`, `contactName`, `attemptCount`, `errorMessage`
- **Lockout:** if `failedAttempts >= 3`, it refuses to look anything up and returns a lockout message (enforced in the flow so the agent can't skip it).
- On success it returns the matched Contact and resets the attempt count to 0; error messages never disclose whether an order or email exists.

### 2. `AWT_Check_Return_Eligibility`
[AWT_Check_Return_Eligibility.flow-meta.xml](force-app/main/default/flows/AWT_Check_Return_Eligibility.flow-meta.xml)

Loops every shipment on the order (scoped to the verified `contactId`, so another shopper's order reads as "not found") and classifies each one. **Eligibility rules** (first match wins):

- **Eligible** — status `Delivered`, **not** final sale, within the return window, and **no existing return Case**.
- **Not eligible** — final-sale item / not yet delivered / past the return window / a return already exists.

It builds a human-readable per-shipment `summary`, counts eligible shipments, and captures the single `eligibleShipmentId`. For **split orders with more than one eligible shipment**, it clears `eligibleShipmentId` and asks the shopper to choose; call the action again with `selectedItem` to pin the choice. It never silently picks one.

- **Inputs:** `orderNumber`, `contactId`, `selectedItem` (optional)
- **Outputs:** `summary`, `eligibleShipmentId`, `eligibleCount`, `ineligibleReason`
- Runs `SystemModeWithoutSharing`.

> **Note:** the eligibility window cutoff formula currently evaluates `TODAY() - 14` while the policy described everywhere else (and enforced by the create flow) is **30 days**. Reconcile these before go-live. This flow's `status` is also `Obsolete` in metadata — re-activate it if it is meant to run.

### 3. `AWT_Create_Return_Case`
[AWT_Create_Return_Case.flow-meta.xml](force-app/main/default/flows/AWT_Create_Return_Case.flow-meta.xml)

Creates one return-request **Case** (`Type = Return`, `Origin = Agentforce`, `Status = New`) for one eligible shipment, re-validating **all** rules server-side before writing anything:

- shipment exists and belongs to the shopper, not final sale, delivered, within 30 days (`TODAY() - 30`), and no existing return Case for the shipment.
- Each failing rule returns its own specific `errorMessage`. The one-return-per-shipment rule is enforced by querying existing return Cases.
- On success it returns the human-friendly `caseNumber`. **No refund field is ever written.**

- **Inputs:** `shopOrderId`, `contactId`, `reason` (optional)
- **Outputs:** `success`, `caseNumber`, `errorMessage`
- Runs `SystemModeWithoutSharing`.

---

## Data model

**`Shop_Order__c`** (a shipment within an order) — custom fields:
`Order_Number__c`, `Email__c`, `Contact__c` (lookup), `Item__c`, `Status__c`, `Delivered_Date__c`, `Final_Sale__c`.

**`Case`** — standard object with a custom `Shop_Order__c` lookup. Return requests are `Type = Return`, linked to the shopper `Contact` and the shipment.

---

## Project layout

```
force-app/main/default/
├── aiAuthoringBundles/Ascend_Retail_Returns_Concierge_2/   # the agent
├── flows/                                                   # the three AWT_* flows
├── genAiPlannerBundles/                                     # generated planner bundle
└── objects/                                                 # Shop_Order__c + Case fields
```

---

## Getting started

```bash
# Authorize a dev/scratch org
sf org login web --alias ascend-dev

# Deploy the metadata
sf project deploy start --source-dir force-app

# Run lint / unit tests
npm install
npm run lint
npm test
```

> **Windows note:** this project has deeply nested `genAiPlannerBundles` paths. If `git` or deploy commands fail with "Filename too long", enable long paths:
> ```bash
> git config core.longpaths true
> ```

---

## Useful links

- [Salesforce Extensions Documentation](https://developer.salesforce.com/tools/vscode/)
- [Salesforce CLI Setup Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_setup.meta/sfdx_setup/sfdx_setup_intro.htm)
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
- [Agentforce documentation](https://help.salesforce.com/s/articleView?id=sf.agentforce.htm)
