# Bulk SMS Campaign Sales Automation Plan

## Current state and goal
You currently sell bulk SMS capacity for political campaigns through Twilio and Bandwidth, and you use a paper workflow to track incoming payments and allocate funds to the right campaign and user.

The goal is to move to a web-based system that:
- sells texting packages online,
- captures payment digitally,
- automatically maps each payment to campaign + user,
- enforces compliance and approval workflow,
- creates an auditable ledger for finance/reconciliation.

---

## Recommended target workflow (regardless of commerce platform)
1. **Campaign/user setup**
   - Campaign admin is onboarded.
   - Users are tied to campaigns with role-based permissions.
   - Approved sender pools (Twilio/Bandwidth numbers or short codes) are assigned.

2. **Package purchase**
   - Buyer selects a package (e.g., 25k, 100k, 500k texts).
   - Buyer chooses campaign and optionally user/cost center.
   - Checkout collects billing details, legal attestations, and consent.

3. **Payment + event handling**
   - Payment processor confirms payment.
   - Webhook creates an immutable order/payment ledger event.
   - Credits are added to campaign wallet and scoped to package rules.

4. **Usage and burn-down**
   - Sending service debits wallet as messages are sent.
   - Delivery receipts update utilization and cost analytics.
   - Low-credit alerts and optional auto-recharge trigger.

5. **Finance and reconciliation**
   - Daily reconciliation against payment processor payouts.
   - Carrier cost import from Twilio/Bandwidth usage records.
   - Margin reporting by campaign, user, and time window.

6. **Compliance and controls**
   - Required political messaging disclosures.
   - Opt-out handling and suppression list enforcement.
   - Audit trail for who purchased, approved, sent, and edited.

---

## Data model essentials
- **Account** (organization), **Campaign**, **User**, **Role**
- **ProductPackage** (SMS credits, expiration, restrictions)
- **Order** (created, paid, failed, refunded)
- **PaymentTransaction** (processor IDs, fees, net)
- **CampaignWallet** (current credits/funds)
- **WalletLedgerEntry** (credit/debit immutable journal)
- **MessageBatch** and **MessageUsage**
- **CarrierCostRecord** (Twilio/Bandwidth imports)
- **Refund/Adjustment** entities

Design around **event-sourcing-like append-only ledger entries** to eliminate ambiguous “paper corrections.”

---

## Option A: Host sales on Shopify (with app integration)

### Best when
- You want fastest go-live for storefront/checkout.
- Non-technical staff need easy product/catalog/promo management.
- You can tolerate integrating Shopify order events into your C# backend.

### Architecture sketch
- Shopify handles storefront, checkout UX, tax, receipts.
- Stripe (through Shopify Payments or external flow) processes payment.
- Shopify webhooks call your C# API:
  - `orders/create`, `orders/paid`, `refunds/create`.
- Your C# backend maps order metadata → campaign/user and credits wallet.
- Twilio/Bandwidth usage debits wallet in your platform.

### Pros
- Faster implementation and polished commerce UX.
- Built-in cart, discount codes, invoices, receipts.
- Lower initial engineering risk for checkout edge cases.

### Cons
- Metadata mapping complexity (campaign/user attribution must be airtight).
- Less flexible checkout logic for complex B2B political billing rules.
- Platform/app fees can be higher at scale.
- Two-system mental model (Shopify for orders, your app for ledger/usage).

### Risks and mitigations
- **Risk**: Wrong campaign allocation due to missing checkout metadata.
  - **Mitigation**: hard-required campaign ID + signed token validation.
- **Risk**: webhook delays/duplicates.
  - **Mitigation**: idempotency keys and replay-safe handlers.

---

## Option B: Build your own C# web app with Stripe directly

### Best when
- You need tight control over campaign attribution and approval flows.
- You already have C# engineering capacity.
- You want one system of record for order + ledger + messaging.

### Architecture sketch
- ASP.NET Core web app serves catalog + account-aware checkout.
- Stripe Checkout or Payment Intents for card/ACH.
- Stripe webhooks to C#:
  - `checkout.session.completed`, `payment_intent.succeeded`, `charge.refunded`.
- Ledger service writes payment events and credits campaign wallet.
- Messaging service debits wallet per Twilio/Bandwidth send outcome.

### Pros
- End-to-end control of business rules and political compliance.
- Strong attribution (campaign/user context throughout checkout).
- Lower long-term platform constraints and better extensibility.

### Cons
- More engineering effort for storefront + billing admin features.
- You own reliability/security posture of commerce surfaces.
- Must implement receipt, tax, and refund workflows thoughtfully.

### Risks and mitigations
- **Risk**: engineering backlog delays launch.
  - **Mitigation**: phase rollout with Stripe Checkout-hosted pages first.
- **Risk**: reconciliation gaps.
  - **Mitigation**: daily automated payout/fee reconciliation jobs.

---

## Shopify vs C# + Stripe comparison matrix

| Category | Shopify-centric | Custom C# + Stripe |
|---|---|---|
| Time to launch | **Fastest** | Moderate |
| Initial build cost | Lower | Higher |
| Long-term flexibility | Medium | **High** |
| Campaign/user attribution control | Medium | **High** |
| Complex approval workflows | Medium | **High** |
| Operational ownership | Lower | Higher |
| Platform fees | Often higher | Often lower (depends on volume) |
| Unified system-of-record | No (split) | **Yes** |

---

## Practical recommendation
If your top pain is **allocation accuracy + auditability** (paper replacement), and you already run C# services, prefer **custom C# + Stripe** for the core ledger and attribution system.

A hybrid path can reduce risk:
1. Use **Stripe Checkout** (hosted UI) with your C# backend for ledger and wallet automation.
2. Add an internal admin portal for finance reconciliation and campaign adjustments.
3. Only consider Shopify if marketing/ecommerce complexity dominates your roadmap.

---

## Suggested phased delivery roadmap

### Phase 0 (1–2 weeks): Foundation
- Define data model and ledger invariants.
- Establish campaign/user identity and RBAC.
- Decide wallet unit (currency vs message credits).

### Phase 1 (2–4 weeks): Payments + wallet automation
- Implement package catalog and checkout flow.
- Stripe webhook ingestion with idempotency.
- Auto-credit campaign wallet on successful payment.

### Phase 2 (2–3 weeks): Messaging integration and burn-down
- Integrate Twilio/Bandwidth send pipeline with wallet debits.
- Delivery status ingestion and usage dashboards.
- Alerts for low credits and payment failures.

### Phase 3 (2–4 weeks): Finance & compliance
- Reconciliation reports (gross, fees, net, carrier cost, margin).
- Refunds/adjustments workflow with approval and audit trail.
- Compliance artifacts, opt-out enforcement, policy logs.

### Phase 4 (ongoing): Scale and controls
- Multi-tenant hardening.
- Fraud/risk checks for large purchases.
- SLA monitoring and incident playbooks.

---

## Technical controls checklist
- Webhook signature validation (Stripe + Shopify if used)
- Idempotent event processing
- Immutable ledger + reversible adjustments (never destructive edits)
- Row-level tenancy guards on campaign data
- PII and secrets encryption at rest
- Full audit logging (actor, action, before/after)
- Retry queues + dead-letter handling
- Monthly restore drill and reconciliation drill

---

## Decision guide (simple)
- Choose **Shopify** if you need speed and standard ecommerce quickly with minimal engineering.
- Choose **Custom C# + Stripe** if correctness of campaign allocation, compliance, and reporting is the strategic core.

For most political bulk-SMS providers with strict attribution needs, **custom C# + Stripe** is the better long-term architecture.
