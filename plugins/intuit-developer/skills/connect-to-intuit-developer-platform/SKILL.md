---
name: connect-to-intuit-developer-platform
description: Connect Claude Code or Codex to Intuit Developer Platform and diagnose platform-session access. Use for first connection, browser MCP OAuth, missing login window, needs-auth, "No matching partner configuration" failures, missing resource_metadata, developer sign-up or Terms-of-Service gating, AUTHORIZATION_ERROR, wrong account, and connectivity checks. Keep QuickBooks app OAuth separate; hand build work to build-quickbooks-app.
---

# Connect to Intuit Developer Platform

Use this skill for platform-session authentication and access recovery. Intuit
Developer Plugin declares the remote MCP endpoint; Claude Code or Codex owns
OAuth discovery, browser launch, callback handling, token storage, and retry.
There is no platform login tool.

## Reference

Read `references/session-oauth-and-access.md` before explaining connection or
diagnosing authentication, sign-up, Terms-of-Service, or authorization issues.

## Required flow

1. Let the host initialize the plugin's remote MCP server.
2. If authentication is needed, the server returns `401` with
   `WWW-Authenticate` protected-resource metadata.
3. The host discovers Intuit's authorization server, opens browser sign-in,
   receives the local callback, stores the resulting credential, and retries.
4. Call `Intuit_developer_connectivity_status()`.
5. Say the developer is connected only after that call succeeds.
6. If the developer then wants to build an integration, hand off to
   `build-quickbooks-app`.

## Boundaries

- Call the product **Intuit Developer Platform** and this package **Intuit
  Developer Plugin**. Do not expose internal project names.
- For "plugin shows MCP but tools are missing," check RFC 9728
  `resource_metadata` preservation on the `401` challenge, and confirm there is
  no static-token/local-proxy/alternate-URL bypass.
- A bare `Bearer realm="Intuit"` challenge is an upstream edge/gateway defect.
- Do not ask for or add a static bearer token to avoid interactive OAuth.
- Do not conflate platform-session OAuth with the QuickBooks app OAuth flow.
- Do not expose proxy, service-mesh, IAM, forwarded-header, trace, or internal
  identity details.
- Do not retry sign-up/TOS (Terms of Service) or authorization failures with altered parameters
  to route around policy.
- For `AUTHORIZATION_ERROR`, say it is not an outage, then re-read the selected
  workspace/app ID and verify the signed-in account. Mention identity mapping
  only after those reads show no expected owned resource.
- Dynamic client registration (DCR) is not supported. OAuth clients are
  registered per AI platform by Intuit, not by the developer.
- For `No matching partner configuration`, explain that the AI platform's
  OAuth client registration is missing or misconfigured upstream; do not
  prescribe client registration, static tokens, or alternate credentials.
- This plugin targets the development environment. Do not claim production availability until verified.
