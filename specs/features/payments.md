# Feature: Payments and Billing

```yaml
spec: features/payments
status: active
last_updated: 2026-06-14
owns:
  - src/billing/**
  - src/server/routes/billing.ts
  - src/server/routes/webhooks-stripe.ts
depends_on:
  - api/rest
  - api/errors
  - data/schema
  - features/notifications
```

## Summary

Each workspace has one Stripe-backed subscription; Stripe is the source of truth and our `subscriptions` table mirrors it via verified webhooks.

## Plans

| Plan | Price | Members | Projects | Storage |
|---|---|---|---|---|
| `free` | $0/mo | ≤ 5 | ≤ 3 | 1 GB |
| `pro` | $12/seat/mo | unlimited | unlimited | 50 GB |
| `enterprise` | custom | unlimited | unlimited | 1 TB+ |

## Invariants

- **INV-PAY-1:** Subscription state in our DB MUST be mutated ONLY by the verified Stripe webhook handler. No other code path writes `subscriptions`.
- **INV-PAY-2:** The webhook handler MUST verify the `Stripe-Signature` header against `STRIPE_WEBHOOK_SECRET` before processing; an invalid signature MUST return 400 and perform no write.
- **INV-PAY-3:** Webhook processing MUST be idempotent: processing the same Stripe `event.id` twice MUST yield identical DB state and side effects exactly once.
- **INV-PAY-4:** An action exceeding the workspace's plan limit MUST be rejected with 403 `PLAN_LIMIT_EXCEEDED` BEFORE any record is written.
- **INV-PAY-5:** Upgrades take effect immediately and are prorated; downgrades take effect at `current_period_end`.
- **INV-PAY-6:** A cancelled subscription MUST retain `pro` access until `current_period_end`, then revert to `free`.
- **INV-PAY-7:** After the final failed dunning attempt, the workspace MUST be downgraded to `free`.
- **INV-PAY-8:** Local subscription state MUST converge to Stripe within 60s of a Stripe event.
- **INV-PAY-9:** Billing endpoints MUST require the workspace `owner` role (else `FORBIDDEN`).

## Contract

`POST /webhooks/stripe` handler steps:
1. Verify signature (INV-PAY-2).
2. Dedupe on `event.id` (INV-PAY-3) using `processed_stripe_events`.
3. Look up workspace by `stripe_customer_id`.
4. Upsert the `subscriptions` row.
5. Enqueue notification/email jobs.
6. Return `200`. Any non-2xx triggers Stripe retry.

Handled events: `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`.

Dunning: 3 retries over 7 days (Stripe Billing); final failure → INV-PAY-7.

## Targets

| Invariant | File | Symbol |
|---|---|---|
| INV-PAY-1 | `src/billing/subscriptions.ts` | `applyStripeEvent` (sole writer) |
| INV-PAY-2 | `src/server/routes/webhooks-stripe.ts` | `verifyStripeSignature` |
| INV-PAY-3 | `src/billing/idempotency.ts` | `markProcessed`, table `processed_stripe_events` |
| INV-PAY-4 | `src/billing/limits.ts` | `assertWithinPlan` |
| INV-PAY-5,6 | `src/billing/transitions.ts` | `scheduleDowngrade`, `applyUpgrade` |
| INV-PAY-9 | `src/server/routes/billing.ts` | `requireOwner` |

## Acceptance

- **AC-PAY-1** (INV-PAY-4): GIVEN a `free` workspace with 3 projects WHEN creating a 4th THEN response is 403 `PLAN_LIMIT_EXCEEDED` and no project row is created.
- **AC-PAY-2** (INV-PAY-3): GIVEN a Stripe event WHEN delivered twice THEN `subscriptions` state is identical and only one notification is enqueued.
- **AC-PAY-3** (INV-PAY-2): GIVEN a webhook with an invalid signature THEN response is 400 and no DB write occurs.
- **AC-PAY-4** (INV-PAY-5): GIVEN a `free` workspace WHEN upgrading to `pro` THEN access changes immediately and the charge is prorated.
- **AC-PAY-5** (INV-PAY-6): GIVEN a cancelled subscription THEN the workspace keeps `pro` until `current_period_end`, then reverts to `free`.
- **AC-PAY-6** (INV-PAY-8): GIVEN `invoice.payment_failed` THEN `subscriptions.status = past_due` and a failure email is enqueued within 60s.

## Verify

```bash
npm test -- billing                         # INV-PAY-1,4,5,6,9
npm run test:webhooks -- stripe --replay    # INV-PAY-2,3 (signature + idempotency)
npm run test:int -- billing-convergence     # INV-PAY-8
```
