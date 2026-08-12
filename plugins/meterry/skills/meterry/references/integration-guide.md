# Meterry Integration Guide

Use this reference after the `meterry` skill triggers. For deployed systems, first fetch `/llms.txt`; for repository work, inspect `frontend/apps/web/public/llms-full.txt` and the endpoint docs under `frontend/apps/docs/src/content/docs/api/`.

## Capability Map

- Rules: `POST /v1/projects/:project_id/extractor-rule-sets`, `POST /v1/projects/:project_id/pricing-rule-sets`, `POST /v1/projects/:project_id/rules/test`, publish endpoints.
- Usage: `POST /v1/projects/:project_id/extractor-rule-sets/:id/events/ingest`, `POST /v1/projects/:project_id/events/batch-ingest`.
- Accounts and routes: `POST /v1/projects/:project_id/accounts`, `POST /v1/projects/:project_id/account-subjects`, `GET /v1/projects/:project_id/account-subjects`.
- Wallets and funds: `POST /v1/projects/:project_id/wallets`, `/wallets/credit`, `/wallets/credit-limit`, voucher grant endpoints, realtime amount and limit reads.
- Usage control: `POST /v1/projects/:project_id/usage-control-rules`, usage-limit plans, resets, and `POST /v1/projects/:project_id/budget-leases`.
- Webhooks: webhook endpoint CRUD, test delivery, event list, and redelivery endpoints.
- Query: usage event list/details, aggregate usage, bill query, wallet ledger entries.

## Recommended Setup Order

1. Create or select a project and server-side API key.
2. Create extractor rule set draft from a representative raw event.
3. Test extraction and attached pricing with `POST /rules/test`.
4. Publish extractor rule set.
5. Create account for the payer.
6. Create wallet for the account and currency; optionally credit it or grant vouchers.
7. Bind each usage producer to the payer account through a subject route.
8. Add `limits[]` when a user, API key, workspace, or agent needs a realtime window.
9. Ingest usage with stable idempotency keys.
10. Handle signed webhooks and query processed usage or realtime state for verification.

## Rule Design

Extractor rule sets contain `subject`, `dimensions`, and `charge_item_extractors`.

- Use `subject` for the usage producer that should resolve to a payer route.
- Use dimensions for provider, model, plan, region, API key ID, organization ID, or other query labels.
- Use charge item metrics such as `input_tokens`, `output_tokens`, `requests`, `tool_calls`, `agent_minutes`, or `workflow_runs`.
- Prefer attached pricing expressions for stable hot-path metrics.
- Use `x-billing.items` for sparse dynamic metrics known only when a request runs.
- If `x-billing.items` reports the same metric as an extractor output, the `x-billing` item is authoritative.

Minimal LLM extractor shape:

```json
{
  "id": "ers_llm_gateway_openai",
  "subject": {
    "type_path": "$.meta.subject_type",
    "id_path": "$.meta.subject_id",
    "default_type": "user",
    "required": true
  },
  "dimensions": [
    {"name": "provider", "path": "$.provider", "type": "string", "required": true},
    {"name": "model", "path": "$.model", "type": "string", "required": true}
  ],
  "charge_item_extractors": [
    {
      "name": "input-tokens",
      "metric": "input_tokens",
      "path": "$.usage.input_tokens",
      "type": "number",
      "unit": "token",
      "skip_if_missing": true,
      "currency": "USD",
      "unit_price_expr": "dec('0')",
      "amount_expr": "dec_mul(dec_div(dec(item['quantity']), dec(item['pricing_unit'])), dec(item['unit_price']))"
    }
  ]
}
```

## Account, Wallet, Route

The account is who pays. The wallet is funds for one account and currency. A subject route maps usage producer to payer account.

```text
project + subject_type + subject_id -> billing_account_id
project + billing_account_id + currency -> wallet
```

Organization-billed user limits should use one organization account and wallet, then one route and optional limit per user. Keep organization ID as a dimension unless the organization itself is the usage producer.

## Usage Limits

Create usage-control rules with:

- `billing_account_id`: payer account.
- `subject_type`, `subject_id`: usage producer.
- `currency`: default currency.
- `limits[]`: range limits.

Limit fields:

- `metric_scope="amount"` sums rated charge item amounts.
- Other metric scopes sum matching charge item metric or unit quantities.
- `period`: `day`, `week`, `month`, `lifetime`; aliases `daily`, `weekly`, `monthly`.
- `timezone`: calendar window timezone.
- `action`: customer-controlled semantics such as `block`, `warn`, or `notify`.
- `webhook_event`: optional business event name for webhook selection.

Use `UsageEvent.OccurredAt` as the business-time source. Late events stay in their historical windows.

## Budget Lease

Use a Budget Lease for one agent run:

1. Ensure the durable subject route already exists.
2. Create `POST /v1/projects/:project_id/budget-leases` with never-reused `run_id`.
3. Attach returned `budget_lease_id` to each event under `x-usage-control`.
4. Stop scheduling task work when `budget_lease.exhausted` arrives.

Do not create a new subject route per task.

## Ingest Event Shape

```json
{
  "source": "llm-gateway",
  "external_event_id": "resp_...",
  "idempotency_key": "openai:resp_...",
  "occurred_at": 1783008000,
  "raw_json": {
    "provider": "openai",
    "model": "gpt-4.1-mini",
    "usage": {"input_tokens": 2400, "output_tokens": 520},
    "meta": {"subject_type": "user", "subject_id": "user_123"}
  },
  "x-billing": {
    "pricing_hints": {
      "input_tokens": {"unit_price": "0.75", "pricing_unit": "1000000", "currency": "USD"}
    }
  },
  "x-usage-control": {
    "budget_lease_id": "ble_..."
  }
}
```

Use batch ingest when sending multiple events, especially when each event may use a different extractor rule set.

## Webhook Handling

Delivery headers include `X-Billing-Webhook-ID`, `X-Billing-Webhook-Delivery-ID`, `X-Billing-Webhook-Timestamp`, and `X-Billing-Webhook-Signature: v1=<hex>`.

Verification workflow:

1. Read raw body before JSON parsing.
2. Build `<timestamp>.<raw body>`.
3. Compute HMAC-SHA256 with endpoint signing secret.
4. Constant-time compare with signature after `v1=`.
5. Reject stale timestamps.
6. Dedupe by event ID.
7. Apply customer-side state changes durably.
8. Return `2xx`.

Events to handle:

- `usage_limit.exhausted`: durable route limit crossed. Block or warn the subject based on payload action.
- `budget_lease.exhausted`: per-run budget crossed. Block/cancel the task run.
- `wallet.available_balance.threshold_crossed`: threshold edge crossed for configured account/currency.
- `wallet.insufficient_balance` and `wallet.balance_changed`: account-level wallet state changes.

## Debugging

Wallet not debiting:

- Published extractor rule set ID is wrong or not used in ingest URL.
- Extracted subject does not match any route in the same project.
- Charge items lack amount or currency.
- Account has no wallet for that currency.

Limit not advancing:

- `occurred_at` is outside expected window.
- `metric_scope` does not match amount, metric, or unit.
- Realtime read used wrong subject, currency, or dimension scope.

Webhook not acting:

- Signature verification used parsed JSON instead of raw body.
- Endpoint subscribed to the wrong standard event type or business event name.
- Handler returns `2xx` before durable state transition.
