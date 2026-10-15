# Sandbox, API Discovery, and Safety

## Step 6: sandbox first

First real API call should target QuickBooks sandbox, not live company.

Call `Intuit_developer_sandbox_list()` first. It returns existing sandboxes,
`count`, `allowed` (10 per user), and `remaining`.

**First decide which of these two you are doing — they need different
handling of `realm_id`:**

- **Mode A — scaffolding or integrating a generic OAuth-connecting app** (the
  normal case: the developer wants an app that any QuickBooks customer, including
  themselves first, will connect to). Use `sandbox_list` only to confirm at
  least one sandbox exists so a first connection is possible. Only US sandbox
  companies are allowed for `com.intuit.quickbooks.payment`. Do not ask the
  developer to pick a realm here — do not turn this into a "which sandbox
  should the app use?" question. The app's `/connect` route sends the
  developer through the real authorize screen; they pick the company there,
    and the resulting OAuth callback's `realmId` is the realm the app stores
    and operates on for that connection. Proceed straight to Step 7/8 (API
    discovery, then wiring the new or existing integration) once readiness is
    confirmed.
- **Mode B — an immediate one-off call you make yourself** (not through the
  scaffolded app) or a destructive sandbox operation (create/reset/delete).
  Here you do need a confirmed target:
  - If sandboxes exist, show them and explicitly offer to reuse one instead of
    creating a new sandbox; ask which to use only if the developer wants a new
    one anyway.
  - If creating, explain sandbox is isolated test company with disposable data.
    First confirm region and SKU as separate choices, then state the exact
    creation and get a separate explicit approval. Do not treat "create another"
    as approval to write.
  - Regions: `US`, `CA`, `AU`, `IN`, `GB`.
  - SKUs: `plus`, `advanced`, `ies_se`, `ies_me`; all except `plus` are US-only.
  - Only `US` sandbox companies are allowed for
    `com.intuit.quickbooks.payment`.
  - At cap, suggest reuse/reset. Deletion is blocked for four hours after create.
  - Reset destroys all test data and is limited once per ten minutes. Confirm
    explicitly and use `sandbox_id` from list, never user-typed company name.
  - Delete is permanent; deleting IES Multi-Entity sandbox also removes child
    companies. Confirm exact target and impact.
  - If developer requests production data first, explain risk and recommend
    sandbox. Proceed only after explicit override.
  - Before the first real API call, confirm the target sandbox's `realm_id` from
    the list/create result (never assume or guess it). The first authenticated
    call must use that confirmed `realm_id`, made only after the sandbox is
    verified to exist.

## Step 7: discover operation and build request

1. Call `Intuit_quickbooks_api_search(query, top_k?)`.
2. Select result based on developer intent. Ask if ambiguous.
3. Call `Intuit_quickbooks_api_describe(id)`.
4. Use returned request/response schema, required `scope`, `base_url`, and
   example cURL.
5. Fill real access token, the confirmed sandbox `realm_id` from Step 6, and
   required parameters. Explain each.

Use `base_url` from describe result. Do not use generic OpenAPI `servers` host;
token from this platform may 401 there.

## Catalog provenance

Treat any pre-supplied catalog response as
mock/static planning data rather than live product metadata. Label it clearly
as non-live. Before a real request, obtain the operation, required scope, and
`base_url` through `Intuit_quickbooks_api_search` and
`Intuit_quickbooks_api_describe`; never fill those values from memory or
invented catalog entries.

When asked to explain discovery without calling tools, name the required
search/describe reads but do not include illustrative endpoint, scope, base
URL, or cURL values. Those values are unknown until returned by the reads.

Treat status/enum-like response fields as advisory. Make one real authenticated
call and inspect raw response before filtering/branching on a field that can be
null or absent.

## Scope check

Never guess scope. Read `Intuit_quickbooks_api_describe(id).scope`.

If developer only asks which scope and has no app, answer directly with minimum
scope and what it grants. Do not force app creation.

If app exists:

1. Read the selected app's current details and compare the operation's required
   scope with the scopes returned by that live read.
2. If present, continue.
3. If missing, ask standalone yes/no to add exact scope.
4. On yes:

```text
Intuit_developer_app_update(
  app_id,
  workspace_id,
  properties_id,
  updates={"scopes": ["<exact full required scope>"]}
)
```

Scope update is additive. Send missing scope only; do not combine with
`redirectUris`, `name`, or `purpose`. If result says `already_configured`,
report it was already present.

If declined, do not call update. Explain operation will fail unauthorized.
Warn against all/broader scopes and recommend minimum; proceed only on explicit
override. Token minted before scope addition lacks it, so repeat QuickBooks app
OAuth afterward.

## Workspace and app discovery

For a request to show available resources, call
`Intuit_developer_workspace_read()` first, then
`Intuit_developer_app_read(workspace_id)` for the workspace the developer
selects. Present only returned names and IDs. If there is more than one
plausible resource, ask the developer to choose; do not add unrelated setup
steps to this response.

## Step 8: run and diagnose

Never resend identical failed request without diagnosis.

Before a local invoice tryout, call `Intuit_developer_sandbox_list` and
confirm that at least one sandbox company exists — this is Step 6 Mode A
above: for a generic OAuth app, do not ask the developer to choose a realm,
each customer (including the developer's own first test connection) supplies
a `realmId` through the OAuth callback. Ask for a sandbox choice only when
making an immediate test call against one specific company yourself or
performing a destructive operation (Step 6 Mode B).

- **401:** check in order: token expired (~1 hour; refresh in the developer's
  application), missing scope
  (describe/compare; reauthorize after addition), wrong environment/company
  for base URL/realm.
- **403:** entitlement/permission issue; retries do not fix. Explain and stop.
- **400:** compare request field-by-field with describe schema: types, required
  fields, minor version, realm ID, unresolved path placeholders.
- **404 token endpoint:** path wrong, not credentials. Use only documented
  `/oauth2/v1/tokens/bearer`.

State likely cause, evidence, corrected retry path, and what changed.

## No fabricated business data

Never substitute invented invoices, bills, customers, vendors, transactions,
amounts, or dates when real call is empty, failed, or not made.

For unexpected empty result:

1. Inspect raw real response.
2. Check date formats, timezone, off-by-one windows, overly strict filters, and
   nullable schema fields.
3. Only after query is verified, state result is genuinely empty or retrieval
   failed and why.

If developer explicitly asks for demo data, label every display unmistakably
as mock. Never let mock resemble claimed real company data.

