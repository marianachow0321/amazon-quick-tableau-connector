# quick-mcp-clients

OAuth **Client ID Metadata Documents** for connecting Amazon Quick Suite to MCP
servers whose authorization servers support them.

## Why this exists

Most vendors let a client obtain a client ID one of two ways:

| Mechanism | Example | What you do |
|---|---|---|
| Dynamic Client Registration (RFC 7591) | Trello, Netlify | Nothing — the client self-registers |
| Console-registered app | Jira, Zendesk | Create an app, copy client ID + secret |

Tableau Cloud uses a third mechanism. Its authorization server advertises:

```
client_id_metadata_document_supported: true
```

which means **the client ID is itself an HTTPS URL**. The authorization server
fetches that URL and reads the client's metadata from the JSON document it
returns. There is no registration endpoint and no console app to create.

Quick Suite's MCP connector form has a Client ID field but no way to register
one with Tableau, so the document has to be hosted somewhere public. This repo
hosts it via GitHub Pages.

## Contents

| File | Client ID URL |
|---|---|
| `tableau.json` | `https://marianachow0321.github.io/quick-mcp-clients/tableau.json` |

## Important constraints

- **`client_id` must exactly equal the URL the document is served from.** The
  document is self-referential; a mismatch is rejected.
- **`redirect_uris` must contain Quick's callback**,
  `https://us-east-1.quicksight.aws.amazon.com/sn/oauthcallback`. This is
  region-specific — change it if your Quick account is not in `us-east-1`.
- **`token_endpoint_auth_method` is `none`.** Tableau's authorization server
  only advertises `none`, i.e. a public client using PKCE. There is no secret,
  so Quick's "Public OAuth client" checkbox must be **checked**.
- The document must be served over HTTPS and be **anonymously fetchable** —
  the authorization server retrieves it server-side, unauthenticated.

## No secrets here

These documents contain only public client metadata: a name, redirect URIs,
grant types, and requested scopes. Publishing them is inherent to how the
mechanism works, not an oversight. Never add a client secret to this repo.

## Quick Suite connector settings

| Field | Value |
|---|---|
| MCP server endpoint | `https://mcp.tableau.com` (no path — `/mcp` returns 404) |
| Authentication method | User authentication |
| Auth configuration | Custom user based OAuth |
| Client ID | the Pages URL of the relevant file above |
| Public OAuth client | checked |
| Client secret | *(blank)* |
| Authorization URL | `https://sso.online.tableau.com/oauth2/authorize` |
| Token URL | `https://sso.online.tableau.com/oauth2/token` |

## Setup

1. Push this repo to GitHub as **public** — the document must be anonymously
   fetchable.
2. Settings → Pages → Source `main`, folder `/ (root)`.
3. Wait for the deployment, then verify before configuring Quick:

   ```bash
   curl -i https://marianachow0321.github.io/quick-mcp-clients/tableau.json
   ```

   Expect `HTTP 200` and `content-type: application/json`. If this 404s, Quick
   will fail with an opaque error.

## Known unknown

It is not confirmed whether Tableau requires the client ID URL's origin to
match the redirect URI's origin. It does not here — the document is on
`github.io`, the callback is on `amazonaws.com`. Some implementations of this
pattern enforce same-origin; the IETF draft does not require it. If
authorization fails with an invalid-client or invalid-redirect error, that
constraint is the most likely cause, and the fallback is to serve this document
from a host you control that can also receive the callback.
