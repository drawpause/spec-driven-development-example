# Feature: Payments and Billing

**Last Updated:** 2024-11-01
**Status:** Active

---

## Summary

Relay uses Stripe for subscription billing. Each workspace has a subscription tied to a plan. Stripe is the source of truth for subscription state; our database mirrors it via webhooks.

---

## Goals
- Offer three tiers with clear, enforced feature gates.
- Handle upgrades, downgrades, and cancellations gracefully.
- Automate invoicing and payment failure handling (dunning).

## Non-Goals
- One-time purchases or seat-level add-ons.
- Manual invoicing or offline payment.
- Currency conversion (USD only in v1).
- Per-project billing or usage-based pricing.

---

## Plans

| Plan | Price | Members | Projects | Storage |
|---|---|---|---|---|
| Free | $0/mo | Up to 5 | Up to 3 | 1 GB |
| Pro | $12/seat/mo | Unlimited | Unlimited | 50 GB |
| Enterprise | Custom | Unlimited | Unlimited | 1 TB+ |

---

## Requirements

### Functional
- [ ] Workspace owners can subscribe, upgrade, downgrade, or cancel.
- [ ] Upgrades are prorated and take effect immediately.
- [ ] Downgrades take effect at the end of the current billing period.
- [ ] Cancelled subscriptions retain Pro access until the period end, then revert to Free.
- [ ] Failed payments trigger a dunning sequence (3 retries over 7 days via Stripe Billing).
- [ ] After the final failed payment, the workspace is downgraded to Free.
- [ ] Owners receive email notifications for invoices, payment failures, and plan changes.

### Non-Functional
- Stripe webhooks must be handled idempotently.
- Subscription state in our DB must be consistent within 60 seconds of a Stripe event.

---

## Behavior

### Feature Gating

When a workspace action would exceed its plan limits, the API returns:
- HTTP `403 Forbidden`
- Error code `PLAN_LIMIT_EXCEEDED`
- Details: current plan, the limit hit, and what the next plan would allow

Example: A Free workspace attempting to create its 4th project receives the above response before any record is written.

### Webhook Handling

Stripe sends events to `POST /webhooks/stripe`. The handler:
1. Validates the `Stripe-Signature` header against `STRIPE_WEBHOOK_SECRET`.
2. Looks up the workspace by `stripe_customer_id`.
3. Updates the local `subscriptions` row.
4. Enqueues any relevant notification or email jobs.

Handled events:
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `invoice.payment_succeeded`
- `invoice.payment_failed`

The handler returns `200 OK` on success. Stripe will retry on any non-2xx response.

---

## Acceptance Criteria

- [ ] A Free workspace cannot create a 4th project; the API returns `PLAN_LIMIT_EXCEEDED`.
- [ ] Upgrading to Pro takes effect immediately and the charge is prorated for the remaining period.
- [ ] A Stripe `invoice.payment_failed` webhook updates local subscription status to `past_due` and sends a failure email within 5 minutes.
- [ ] Receiving the same Stripe webhook event twice produces identical DB state (idempotent).
- [ ] A cancelled subscription's workspace retains Pro access until `current_period_end`, then reverts to Free.
