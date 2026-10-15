# Intuit Developer

Build QuickBooks integrations with the Intuit Developer Platform. The plugin helps you set up a developer workspace and app, complete browser OAuth, manage sandbox companies, and generate verified QuickBooks API calls.

## Skills

- `connect-to-intuit-developer-platform`: browser OAuth, sign-up and terms gate, and access recovery.
- `build-quickbooks-app`: end-to-end workspace, app, OAuth, sandbox, API request, and troubleshooting workflow.

## Data sent

The plugin connects to one remote MCP server, `https://ai-inc-developer.intuit.com/v1/mcp`, operated by Intuit. After you sign in through browser OAuth, it sends your session identity, workspace, app, and sandbox details, and the QuickBooks API requests you ask it to build or verify. It does not send data to any other destination. Sandbox create, reset, and delete are state-changing actions and run only when you request them.

See the [Intuit Privacy Statement](https://www.intuit.com/privacy/statement/).

## License

Apache-2.0
