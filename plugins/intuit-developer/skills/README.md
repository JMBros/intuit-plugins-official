# Intuit Developer Plugin Skills

## Available skills

- `connect-to-intuit-developer-platform` — platform-session browser OAuth,
  sign-up/TOS gate, and access recovery.
- `build-quickbooks-app` — end-to-end QuickBooks workspace, app, OAuth,
  sandbox, API request, and troubleshooting workflow.

Skills are plugin-first: `SKILL.md` handles routing and core constraints;
skill-local references carry detailed workflows. Remote MCP remains canonical
for live schemas, parameters, authenticated data, mutations, dynamic resources,
environment URLs, and OpenAPI-derived operation details.

## Runtime tools

Use the current tool names exposed by the connected MCP:

- `Intuit_developer_connectivity_status`
- `Intuit_developer_workspace_read`
- `Intuit_developer_app_read`
- `Intuit_developer_app_create`
- `Intuit_developer_app_update`
- `Intuit_developer_oauth_details`
- `Intuit_developer_sandbox_list`
- `Intuit_developer_sandbox_create`
- `Intuit_developer_sandbox_reset`
- `Intuit_developer_sandbox_delete`
- `Intuit_quickbooks_api_search`
- `Intuit_quickbooks_api_describe`

Use MCP discovery for current tool names and parameters, and live MCP reads for
current IDs, URLs, scopes, and other data. Skill-local references explain
workflow order and safety; they are not authoritative API documentation.
