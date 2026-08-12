---
name: meterry
description: Build Meterry customer integrations for LLM API usage metering and agent SaaS billing. Use when implementing or debugging extractor rule sets, usage event ingestion, x-billing dynamic charge items, billing accounts, wallets, voucher grants, subject routes, usage limits, budget leases, usage queries, or signed Meterry webhook receivers.
---

# Meterry

## Overview

Use this skill to integrate a customer backend or agent runtime with Meterry. Meterry normalizes raw usage events into charge items, rates them, debits account wallets, advances subject limits, and emits signed webhooks for balance and limit state changes.

## Workflow

1. Identify the customer's model: LLM gateway, organization-billed SaaS users, per-task agent budgets, analytics-only metering, or a hybrid.
2. Read `references/integration-guide.md` before writing integration code, rule JSON, webhook handlers, or setup scripts.
3. Use server-side API keys only. Never put Meterry project API keys in browser, mobile, or customer-exposed code.
4. Create and test extractor rule sets before ingesting production traffic. Prefer extractor-attached pricing for stable hot-path metrics; use fallback pricing rule sets only when direct pricing is not enough.
5. Create payer state in this order: account, wallet, optional wallet credit or voucher, subject route, optional limits.
6. Ingest usage with stable idempotency keys and business `occurred_at` timestamps.
7. Verify webhooks from the raw request body before parsing JSON, dedupe by event ID, and make customer-side blocking or cancellation durable before returning `2xx`.

## Resource Loading

- Read `references/integration-guide.md` for end-to-end workflows, endpoint order, JSON examples, and webhook semantics.
- When working inside the Meterry repository, also inspect `frontend/apps/web/public/llms.txt`, `frontend/apps/web/public/llms-full.txt`, and the endpoint docs under `frontend/apps/docs/src/content/docs/api/`.
- When working against a deployed customer Portal, fetch `/llms.txt` first, then `/llms-full.txt` or the linked `/docs/api/...` page needed for the endpoint.

## Implementation Rules

- Treat `Account` as the payer and `Wallet` as account-currency funds. Do not use the usage subject itself as the payer unless the customer's product model truly bills that subject directly.
- Bind `subject_type + subject_id` to an account with a subject route before expecting realtime wallet debit or subject limits to work.
- Use `UsageEvent.OccurredAt` semantics for customer limit windows and late reports. Do not use processing time as the business usage time.
- Use decimal strings for money and quota values in request bodies.
- Use idempotency keys for wallet credits and every ingest retry path.
- For task budgets, create one Budget Lease per run and attach its ID through `x-usage-control.budget_lease_id`; do not create a new subject route per task.
- For `x-billing.items`, remember that reported dynamic items override extractor-generated charge items for the same metric.
- For webhook receivers, verify HMAC-SHA256 over `<timestamp>.<raw body>`, reject stale timestamps, dedupe event IDs, and durably update the customer system before acknowledging.

## Validation Checklist

- Rule test returns the expected subject, dimensions, charge items, currency, and amounts.
- Published extractor rule set ID is used in ingest URLs.
- Events include stable idempotency keys and correct `occurred_at`.
- Account, wallet, subject route, and limits are in the same project.
- Realtime wallet amount and realtime limits endpoints return expected state after a test event is processed.
- Webhook receiver passes a signature test and handles duplicate deliveries safely.
