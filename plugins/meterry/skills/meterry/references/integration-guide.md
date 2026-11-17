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

Minimal LLM extractor creation body for `POST /v1/projects/:project_id/extractor-rule-sets`:

```json
{
  "name": "LLM gateway OpenAI",
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

The create endpoint assigns the rule set ID. Use the returned `id` as `extractor_rule_set_id` when testing with `POST /rules/test`, and in the publish and single-event ingest URLs. Do not supply a custom `id` in the creation body.

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

- `measure`: `amount` for rated monetary amounts or `quantity` for usage quantities.
- `selector_type`: `all`, `metric`, or `unit`. Set `selector_value` for `metric` or `unit` (for example `input_tokens` or `token`). `all` requires `measure="amount"`; `unit` requires `measure="quantity"`. `metric` supports either measure.
- `currency`: required for amount limits, resolved from the limit or the rule's default currency. Amount limits count rated amounts in that currency.
- `limit_amount`: a positive decimal string; `init_amount` is an optional nonnegative starting usage value.
- `metric_scope`: legacy compatibility field. Prefer explicit `measure` and selector fields in new integrations rather than relying on metric-or-unit inference.
- `period`: `hour`, `day`, `week`, `month`, `lifetime`; aliases `daily`, `weekly`, `monthly`. Custom periods include `5h`, `5 day`, `2 week`, and `5 month`.
- `period_type`: defaults to `current`. Direct current windows anchor to rule creation or reset; Plan current windows retain the original subject-route cycle across resets. Set `calendar` explicitly for natural hour/day/week/month boundaries; calendar weeks start on Monday.
- `timezone`: timezone for calendar boundaries. Lifetime limits have no timezone or periodic reset.
- `action`: customer-controlled semantics such as `block`, `warn`, or `notify`.
- `webhook_event`: optional business event name for webhook selection.

Example body for `POST /v1/projects/:project_id/usage-control-rules`, with a calendar month amount limit and a calendar day token limit:

```json
{
  "billing_account_id": "acct_...",
  "subject_type": "user",
  "subject_id": "user_123",
  "currency": "USD",
  "limits": [
    {
      "measure": "amount",
      "selector_type": "all",
      "period": "month",
      "period_type": "calendar",
      "timezone": "Asia/Singapore",
      "limit_amount": "100",
      "action": "block"
    },
    {
      "measure": "quantity",
      "selector_type": "metric",
      "selector_value": "input_tokens",
      "period": "day",
      "period_type": "calendar",
      "timezone": "Asia/Singapore",
      "limit_amount": "1000000",
      "action": "warn"
    }
  ]
}
```

Use `UsageEvent.OccurredAt` as the business-time source. Late reports use their event-time windows; creation and reset counting boundaries still apply.

## Budget Lease

Use a Budget Lease for one agent run:

1. Ensure the durable subject route already exists.
2. Create `POST /v1/projects/:project_id/budget-leases` with never-reused `run_id`.
3. Attach returned `budget_lease_id` to each event under `x-usage-control`.
4. Stop scheduling task work when `budget_lease.exhausted` arrives.

Create the lease with the existing route's payer and subject, an explicit currency, and a positive decimal-string budget:

```json
{
  "run_id": "run_123",
  "billing_account_id": "acct_...",
  "subject_type": "user",
  "subject_id": "user_123",
  "currency": "USD",
  "limit_amount": "5.00",
  "action": "block"
}
```

The response's `id` is the `budget_lease_id` used by ingest. `action` supports `block` (the default) or `notify`; `webhook_event` is optional. Use a new `run_id` for each new run; retry creation for the same run with the same account, subject, currency, and budget to retrieve the existing lease. Do not create a new subject route per task.

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
      "input_tokens": {"unit_price": 0.75, "pricing_unit": 1000000, "currency": "USD"}
    }
  },
  "x-usage-control": {
    "budget_lease_id": "ble_..."
  }
}
```

`x-billing.items[].quantity`, `pricing_hints.*.unit_price`, `pricing_hints.*.pricing_unit`, and `discount_hints.*.value` are JSON numbers. Wallet and limit amounts use decimal strings as shown above.

Use batch ingest when sending multiple events, especially when each event may use a different extractor rule set. Send `POST /v1/projects/:project_id/events/batch-ingest` an outer `events` array; each event needs its own returned `extractor_rule_set_id` and stable `idempotency_key`:

```json
{
  "source": "llm-gateway",
  "events": [
    {
      "extractor_rule_set_id": "ers_...",
      "external_event_id": "resp_batch_1",
      "idempotency_key": "openai:resp_batch_1",
      "occurred_at": 1783008000,
      "raw_json": {
        "provider": "openai",
        "model": "gpt-4.1-mini",
        "usage": {"input_tokens": 2400, "output_tokens": 520},
        "meta": {"subject_type": "user", "subject_id": "user_123"}
      },
      "x-billing": {
        "pricing_hints": {
          "input_tokens": {"unit_price": 0.75, "pricing_unit": 1000000, "currency": "USD"}
        }
      }
    }
  ]
}
```

Each batch event can also carry its own `x-usage-control` lease binding. Replace illustrative IDs with IDs returned by setup APIs.

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

- `occurred_at` is outside the expected window or before the applicable creation/reset counting boundary; check `period_type` and the window anchor.
- `measure`, `selector_type`, or `selector_value` does not match the charge items; amount limits also require matching currency.
- Realtime read used wrong subject, currency, or dimension scope.

Webhook not acting:

- Signature verification used parsed JSON instead of raw body.
- Endpoint subscribed to the wrong standard event type or business event name.
- Handler returns `2xx` before durable state transition.
