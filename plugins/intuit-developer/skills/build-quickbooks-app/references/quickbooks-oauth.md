# QuickBooks OAuth: scaffold or integrate

If an application already exists, follow its framework, configuration,
callback, and token-storage conventions; the examples below are only for a
new local scaffold.

Use this guide when building an app that connects QuickBooks customers.
OAuth token exchange is performed by the app you are scaffolding. Do not look
for an OAuth exchange or refresh MCP tool.

## 1. Use the app credentials

Read the selected app with `Intuit_developer_app_read` before writing the
scaffold. Use the returned `client_id` to configure the local sandbox app.
`client_secret` is NEVER read or returned by this MCP — the response instead
carries a `portal_url` deeplink to the app's Keys page in the Intuit
Developer Platform. Tell the developer to open it, sign in, and copy the
secret themselves; it goes directly into their own local app's environment,
saved securely there.

For a greenfield local app, configure the app with:

```text
CLIENT_ID=<client_id from Intuit_developer_app_read>
CLIENT_SECRET=<retrieved by the developer from portal_url, set securely>
AUTHORIZE_URL=<authorize URL from the connected MCP>
TOKEN_URL=<token URL from the connected MCP>
REDIRECT_URI=<exact callback URI used by the local app>
```

Call `Intuit_developer_oauth_details` and replace `AUTHORIZE_URL` and `TOKEN_URL`
with its returned values. Do not use remembered URLs or the authorize
resource's server-generated state for the application.

For an existing app, first read its configured client ID, redirect URI,
environment-variable names, and callback route. Preserve those conventions
and map them to the selected platform app. Do not introduce `CLIENT_ID`,
`CLIENT_SECRET`, or `REDIRECT_URI` under new names when the project already
has an established configuration pattern.

For a development tryout, the developer copies their Development client secret
directly from the `portal_url` Keys page into the local `.env` file — the MCP
never returns `client_secret`, so the agent never receives or writes it. Ensure
`.env` is listed in `.gitignore`, never print the secret, and never commit or
place it in source, `.env.example`, logs, or shell history.
Write only the non-secret values to `.env.example`. Never read, cat, grep, or
otherwise open `.env` or any other file holding the secret. Before starting the
app, ask the developer to confirm that `CLIENT_ID` and `CLIENT_SECRET` are set
in their own environment; the app itself fails fast at startup if either is
missing, without printing the values.

### Making `.env` easy to reach in the AI coding tool

Don't wait for the developer to ask "where do I put this?" — as soon as the
scaffold needs `CLIENT_SECRET`, create the `.env` file yourself with every
non-secret value filled in and a clearly-fake placeholder line for the
secret, e.g. `CLIENT_SECRET=paste-your-client-secret-here`, and tell the
developer the exact path you wrote it to.

When no project folder is selected (a scratch/session workspace), say this
explicitly, because it changes how they'll reach the file:

- Dotfiles like `.env` may not appear in the Files pane, especially with no
  folder selected — this is normal, not an error.
- Give them a way to actually open it: the literal absolute path for their own
  editor or terminal, or the option to move the session to a real project
  folder (after which the whole directory, `.env` included, is reachable from
  the Files pane going forward).

Regardless of environment, never accept the client secret as chat text and
write it into `.env` yourself, even if the developer offers to paste it —
a chat transcript is exactly the kind of log this value must never land in.
Point them to the file and portal `Keys` page and have them edit it directly,
then wait for them to confirm it's done.

Once they confirm, always tell them to restart the local app process before
testing again. Most runtimes (Flask, Node, etc.) read environment variables
once at process startup; a server left running from before the edit will
keep using the old (missing or placeholder) secret until it is restarted.

For a greenfield local app, choose a local port that is available for your
callback listener. For an existing app, use its current callback route and
port unless the developer explicitly approves changing them. In either case,
register the exact callback URL in the app's settings.
The listener must be bound to that port when OAuth redirects back, and the
same URL must be used as the registered callback URL, `redirect_uri`, and
token-exchange `redirect_uri`. For example:

```text
http://localhost:9753/oauth/callback
```

Read existing `redirectUrls` first. If the callback is missing, preserve the
existing list when calling `Intuit_developer_app_update`. The listener address,
authorize `redirect_uri`, and token `redirect_uri` must match exactly.

## 2. Build the authorize URL

For a local application, use the authorize endpoint returned by
`Intuit_developer_oauth_details`:

```text
AUTHORIZE_URL=<authorize URL from the connected MCP>
```

The connected MCP supplies the correct environment host; never invent an
OAuth host. The app must append all query parameters and generate `state`
itself. Use the same app-generated state value for the authorize request and
callback validation.

Add these URL-encoded parameters:

```text
client_id=CLIENT_ID
redirect_uri=REDIRECT_URI
response_type=code
scope=MINIMUM_OPERATION_SCOPE
state=FRESH_RANDOM_STATE
```

Before building this URL, call `Intuit_quickbooks_api_describe(id)` for the
operation the app will call and use its exact `scope` value. Request only the
minimum scope required by that operation and already enabled for the selected
app; do not hardcode an accounting scope. For multiple operations, request
the minimum union of their exact described scopes. This also supports
payment-only apps.

Generate one new unpredictable `state` per connection attempt. Store that
exact value in the user's session, append it to the authorize URL, and reject
the callback if the returned state differs.

## 3. Add the local callback

For a Node.js app, use this shape. The same `state` generated here must be
the one sent in the authorize URL and checked in the callback:

```js
import crypto from "node:crypto";

const AUTHORIZE_URL = process.env.AUTHORIZE_URL;
const REDIRECT_URI = process.env.REDIRECT_URI;

function startOAuth(clientId, operationScope, session) {
  const state = crypto.randomBytes(32).toString("hex");
  session.oauthState = state;
  const query = new URLSearchParams({
    client_id: clientId,
    redirect_uri: REDIRECT_URI,
    response_type: "code",
    scope: operationScope,
    state,
  });
  return `${AUTHORIZE_URL}?${query}`;
}

function validateCallback(state, session) {
  if (!session.oauthState || state !== session.oauthState) {
    throw new Error("Invalid OAuth state. Start a new connection.");
  }
  delete session.oauthState;
}
```

Store the session state per browser/customer, not in one global variable.
After validation, exchange the code and store the returned tokens together
with the callback's `realmId`. Never generate a second state between the
authorize redirect and callback.

The callback receives `code`, `state`, and usually QuickBooks `realmId`.
Preserve `realmId` with the customer's tokens.

## 4. Exchange the code in the local app

Use the token endpoint supplied by the connected MCP. Never substitute a
remembered or guessed endpoint. The request is a form-encoded POST with HTTP
Basic auth for `CLIENT_ID` and `CLIENT_SECRET`:

```js
function requireHttpsTokenUrl(value) {
  const url = new URL(value);
  if (url.protocol !== "https:") {
    throw new Error("TOKEN_URL_FROM_MCP must use HTTPS");
  }
  return url;
}

async function exchangeCode({ code, clientId, clientSecret, redirectUri }) {
  const tokenUrl = requireHttpsTokenUrl(process.env.TOKEN_URL);
  const basicAuth = Buffer.from(`${clientId}:${clientSecret}`).toString("base64");
  const response = await fetch(tokenUrl, {
    method: "POST",
    headers: {
      Accept: "application/json",
      Authorization: `Basic ${basicAuth}`,
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      code,
      redirect_uri: redirectUri,
    }),
  });
  if (!response.ok) throw new Error(`Token exchange failed: ${response.status}`);
  return response.json();
}
```

Put this request in the developer's callback service, not in an MCP call.
Store the returned refresh token and access token using the app's secure
customer-token storage.

## 5. Refresh in the local app

When the access token expires, refresh it from the app:

```js
async function refreshToken({ refreshToken, clientId, clientSecret }) {
  const tokenUrl = requireHttpsTokenUrl(process.env.TOKEN_URL);
  const basicAuth = Buffer.from(`${clientId}:${clientSecret}`).toString("base64");
  const response = await fetch(tokenUrl, {
    method: "POST",
    headers: {
      Accept: "application/json",
      Authorization: `Basic ${basicAuth}`,
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: new URLSearchParams({
      grant_type: "refresh_token",
      refresh_token: refreshToken,
    }),
  });
  if (!response.ok) throw new Error(`Token refresh failed: ${response.status}`);
  return response.json();
}
```

Save a rotated refresh token when one is returned. Use the new access token
for QuickBooks API requests. Use `Intuit_quickbooks_api_describe` to obtain
the exact API path, scope, and request shape.

## Rules

- Use exact MCP-returned values for app IDs, credentials, scopes, and URLs.
- Keep `CLIENT_SECRET`, authorization codes, and tokens out of source control
  and logs.
- Never guess OAuth paths or perform token exchange or refresh through MCP.
- Keep the redirect URI identical in registration, authorize, and token calls.
