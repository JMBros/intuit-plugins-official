# Platform Session OAuth and Access

## Two separate OAuth relationships

1. **Platform-session OAuth** authenticates Claude Code or Codex to Intuit
   Developer Platform. Host and platform handle it before a tool runs.
2. **QuickBooks app OAuth** authorizes a developer-created app to access a
   QuickBooks company. `build-quickbooks-app` orchestrates that later with the
   app's own `client_id` and `client_secret`.

Never combine their credentials, callbacks, scopes, or troubleshooting.

## Browser login flow

The plugin's `.mcp.json` declares one remote HTTP MCP server. It intentionally
contains no API key, bearer token, client secret, or custom callback command.

1. Claude Code or Codex initializes the remote server without a user token.
2. Server returns HTTP `401` with `WWW-Authenticate` pointing to
   `/.well-known/oauth-protected-resource`.
3. Host reads protected-resource metadata, discovers Intuit's authorization
   server, and performs OAuth with PKCE using the OAuth client Intuit
   registered for that AI platform. Dynamic client registration (DCR) is not
   supported.
4. Host opens Intuit browser sign-in (url given by PRM)
5. User signs in and approves.
6. Host receives callback, exchanges code, stores credential in host-managed
   secure storage, and retries initialization/tool call with a bearer token.
7. `Intuit_developer_connectivity_status()` confirms actual connectivity.

Intuit Developer is a remote (web) MCP server. In Claude Code it
appears as `claude.ai Intuit Developer`, not as a locally configured server,
so there is no `claude mcp login` command for it. If the automatic flow fails,
run `/mcp` in Claude Code, select `claude.ai Intuit Developer`, and
authenticate from there. Do not invent a login tool.

## Authentication recovery

- If connectivity status fails, stop later workflow calls. Tell developer to
  reconnect the MCP server and retry.
- **Partial failure (some tools work, others don't) is a distinct case from
  full disconnection.** If `Intuit_developer_connectivity_status()` succeeds
  and workspace/app reads work, but a different call (e.g. sandbox) fails,
  the session is authenticated but likely stale — it is not the same as never
  being connected. Do not jump straight to escalation here: have the developer
  reconnect the connector and retry the failed call. Only escalate if the
  same call still fails identically after reconnecting.
- If browser login never opens, confirm server is enabled and approved, inspect
  host MCP status, then reconnect `claude.ai Intuit Developer` from `/mcp`.
- If OAuth reports `No matching partner configuration found for
  resourceAssetId: ...` and this reproduces even after reconnecting, the AI
  platform's OAuth client registration with Intuit is missing or
  misconfigured. OAuth clients are registered per AI platform, and DCR is not
  supported: plugin content, static tokens, custom client credentials, or
  further retries cannot repair it. Escalate the client registration to the
  platform owner and preserve the host-managed browser OAuth flow.
- If client sees only `WWW-Authenticate: Bearer realm="Intuit"`, edge gateway
  stripped protected-resource metadata. Server/gateway must preserve
  `resource_metadata`; plugin cannot add it from `.mcp.json`.
- If identity resolves but a workspace/app call reports permission failure,
  confirm developer used correct Intuit account and resource ID.
- Resume at failed step after reconnection; do not restart completed work.

## Developer sign-up and Terms of Service

Most live tools require a developer profile and accepted third-party Terms of
Service. This gate is separate from successful platform-session OAuth.

Canonical response:

```json
{"message":"Sign up or Sign in at developer.intuit.com, and sign the Terms of Service."}
```

Direct developer to `developer.intuit.com`, ask them to sign up/sign in and
accept Terms of Service, then retry same call. No MCP tool accepts TOS.

## Authorization errors

Canonical shape:

```json
{
  "error": "You don't have access to this resource. Verify you have permission and are referencing a workspace or app that belongs to you.",
  "type": "AUTHORIZATION_ERROR"
}
```

Treat `AUTHORIZATION_ERROR` as stable permission response, not transient
gateway failure. Re-read workspaces/apps and verify selected IDs. Never mutate
IDs or retry unrelated privileged calls hoping one succeeds.

For an authorization error on a specific workspace or app ID, lead with the ID
and signed-in-account checks. Do not introduce the identity-mapping caveat
unless reads return no resources that the developer knows they own.

Use this response order: say it is not an outage; re-read the selected
workspace/app ID; verify the currently signed-in developer account; then, and
only if expected resources are absent from the reads, describe identity mapping
as a possible platform-side mismatch.

## Identity mismatch caveat

If `Intuit_developer_workspace_read()` returns zero but developer knows they
own workspaces, platform-session identity may not map to same account used at
`developer.intuit.com`. Present this as known unresolved possibility, not proof
that no workspace exists. Ask developer to verify signed-in account.

## Privacy

Report effects only: authenticated, reconnect required, permission denied.
Never repeat internal proxies, sidecars, service topology, forwarded headers,
IAM tickets, internal account IDs, transaction/trace IDs, backend hostnames,
GraphQL operation names, raw error payloads, stack traces, internal credentials,
or other internal implementation details. Describe user-facing effects in plain
language. Paraphrase `AUTHORIZATION_ERROR` and similar stable types without
quoting raw internal values.
