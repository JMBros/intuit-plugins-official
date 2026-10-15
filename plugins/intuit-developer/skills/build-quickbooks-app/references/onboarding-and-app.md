# Onboarding, Workspace, and App

## Ground rules

Follow prerequisite chain:

`identity → workspace → app → scopes → credentials → access token → sandbox → request`

When a prerequisite is missing, name first missing link, explain why it blocks
later work, and resume there. Do not fabricate placeholders.

Before every create or update:

1. State exact proposed change and target.
2. Ask standalone confirmation question.
3. Wait for explicit approval.
4. If declined, do not call tool, re-ask, or work around refusal. Explain what
   later work is blocked.

Never treat silence, vague agreement, or agent recommendation as consent.

When a workspace, app, scope, or operation reference is ambiguous, show
matches and ask. Never silently choose first, newest, closest name, or broadest
scope.

## Step 1: connectivity

Call `Intuit_developer_connectivity_status()`. Report only whether developer is
authenticated and platform reachable. Never expose how identity was resolved.
If call fails, use `connect-to-intuit-developer-platform` and stop this flow.

Workspace or app IDs can contain internal namespace prefix before `:`. Pass
full returned ID back to tools unchanged. When displaying it, show only portion
after final colon.

## Step 2: workspace

Call `Intuit_developer_workspace_read()`.

- **Zero:** workspace creation is not available through this plugin or its MCP
  tools. Direct the developer to create a workspace at `developer.intuit.com`,
  then re-run `Intuit_developer_workspace_read()` and resume here. Do not call a
  workspace-create tool or invent one. Total portal-owned workspace cap is 10.
- **One:** reuse it and show name plus display-safe ID.
- **Multiple:** show each workspace by name and display-safe ID, plus a
  user-friendly role from live membership details in the read response. If the
  response indicates the caller owns the workspace, you may recommend it but
  must not select it automatically. If it indicates limited or shared access,
  label it accordingly and do not treat it as caller-owned. Do not infer
  ownership from tier, organization visibility, badges, or permissions;
  permissions describe capabilities, not ownership. Always present the choices
  and ask the developer to select a workspace. Use the current MCP schema and
  returned data—do not depend on specific field paths or enum values.

Workspace deletion is unsupported here and not self-service in portal. Suggest
reuse.

## Step 3: app and scopes

Call `Intuit_developer_app_read(workspace_id)`.

- **Zero:** present only real QuickBooks scopes:
  - `com.intuit.quickbooks.accounting` — read/write invoices, bills,
    customers, vendors, and accounts.
  - `com.intuit.quickbooks.payment` — create/manage payments, charges, and
    refunds. Only US sandbox companies are allowed for this scope.
- At least one scope is required. Do not offer none/skip/later. If developer
  cannot choose, ask what they intend to build.
- Pass exactly selected scopes to
  `Intuit_developer_app_create(workspace_id, app_name, scopes)`. Creation
  accepts comma-separated scope string, not list. Suggest `My QuickBooks App`
  only as optional name. Ask explicit approval with workspace, name, and
  scopes. Cap is 5 MCP-created apps per workspace.
- **One:** reuse by name.
- **Multiple:** show list by name and display-safe ID, ask, and wait.
- Before creating any app, recommend reusing a suitable existing app. For a
  same/similar name, show the existing app and ask reuse versus genuinely new
  app.
- If the local project already contains a client ID, portal app ID, app name,
  or registered redirect URI, use that context to match a platform app after
  reading the available apps. Ask about any mismatch; do not create a new app
  merely because the project uses different configuration names.
- If an existing project identifies an app but no matching live platform app
  can be found, show the mismatch and stop before creating or replacing
  resources.

App deletion is unsupported by tools. Direct developer to
`https://developer.intuit.com`.

## Step 4: read credentials

Call `Intuit_developer_app_read(workspace_id, app_id)` for full app detail.
Read the selected app's current details and use the returned update handle,
scopes, credentials, portal link, and redirect configuration as required by
the next operation. Never infer or hardcode those values.

For an existing-project integration, compare the live read with the
project's configuration before writing code. Preserve the project's callback
route and token-storage convention when they are valid. If a redirect must be
added, send the complete existing-plus-new list, obtain explicit approval,
and re-read the app after the update.

There is no `get_credentials` tool and no client-credentials exchange tool.
`client_secret` is never read or returned by this MCP.

Show `client_id` freely; it is not a secret. For `client_secret`, give the
developer the `portal_url` deeplink and tell them to open it, sign in, and
copy the secret themselves from the Keys page. Use it together with
`client_id` to securely exchange the OAuth authorization code for an access
token — store it in the app's own environment, never in generated source,
config, README, logs, shared channels, or shell history, and do not repeat
it back once shown.

If `client_id` is absent, re-verify app ID and app creation completion. Never
invent placeholder credentials.

## Platform boundaries

Tools cannot delete workspace/app, issue production keys, or promote an app.
Production readiness, review, verification, and keys happen in Intuit
Developer Platform. This plugin and current MCP endpoint target development work.
