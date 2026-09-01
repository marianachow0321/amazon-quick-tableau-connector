# amazon-quick-tableau-connector

Connect **Amazon Quick Suite** to **Tableau Cloud** using Tableau's hosted MCP
server, with no proxy, no stored credentials, and per-user Tableau identity.

This repo contains one file that matters: `tableau.json`, an OAuth **Client ID
Metadata Document**. It is served over GitHub Pages and its URL is used
directly as the Client ID in Quick's connector settings.

## Why a JSON file is the client ID

Vendors let a client obtain a client ID in one of three ways:

| Mechanism | Example | What you do |
|---|---|---|
| Dynamic Client Registration (RFC 7591) | Trello, Netlify | Nothing — the client self-registers |
| Console-registered app | Jira, Zendesk, Figma | Create an app, copy client ID + secret |
| **Client ID Metadata Document** | **Tableau Cloud** | **Host a JSON file; its URL is the client ID** |

Tableau Cloud's authorization server advertises:

```
$ curl -s https://sso.online.tableau.com/.well-known/oauth-authorization-server \
    | jq '{client_id_metadata_document_supported, registration_endpoint}'
{
  "client_id_metadata_document_supported": true,
  "registration_endpoint": null
}
```

No registration endpoint, so there is nothing to register and no console app to
create. Instead the client ID is an HTTPS URL, and the authorization server
fetches it at request time to learn who the client is.

Quick Suite's MCP connector form has a Client ID field but no way to obtain one
from Tableau — so the document has to be hosted publicly. That is all this repo
does.

## What happens at sign-in

```
1. User clicks Connect in Quick
2. Quick → https://sso.online.tableau.com/oauth2/authorize
              ?client_id=https://marianachow0321.github.io/amazon-quick-tableau-connector/tableau.json
              &code_challenge=...  (PKCE, no client secret)
3. Tableau FETCHES that URL
4. Tableau reads:
      client_name                 → shown on the consent screen
      redirect_uris               → validates Quick's callback
      token_endpoint_auth_method  → "none", so PKCE and no secret
5. User signs in to Tableau, consents
6. Quick exchanges the code at /oauth2/token
7. Tableau MCP makes REST API calls AS THAT USER
```

Step 7 is the important one: the user's own Tableau permissions are enforced by
Tableau, not approximated by us. Nothing here holds a credential.

## Quick Suite connector settings

| Field | Value |
|---|---|
| MCP server endpoint | `https://mcp.tableau.com` |
| Authentication method | User authentication |
| Auth configuration | Custom user based OAuth |
| Client ID | `https://marianachow0321.github.io/amazon-quick-tableau-connector/tableau.json` |
| Public OAuth client | **checked** |
| Client secret | *(leave blank)* |
| Authorization URL | `https://sso.online.tableau.com/oauth2/authorize` |
| Token URL | `https://sso.online.tableau.com/oauth2/token` |

The endpoint is the bare host. `https://mcp.tableau.com/mcp` returns 404.

## Constraints that will bite you

- **`client_id` in the JSON must be byte-identical to the URL it is served
  from.** The document is self-referential: that binding is what stops someone
  else hosting a document claiming your client ID. A trailing slash breaks it.
- **The repo must be public.** Tableau's authorization server fetches the
  document server-side and unauthenticated. A private repo fails with an opaque
  error.
- **`token_endpoint_auth_method` is `none`.** Tableau advertises only `none`,
  i.e. a public client using PKCE. Quick's "Public OAuth client" box must be
  checked and the secret left empty.
- **Quick's callback is region-scoped, not account-scoped.** Every Quick tenant
  in a region shares one callback URL, which is why several are listed and why
  this document is reusable by other accounts. Extra entries cost nothing —
  Tableau only checks that the incoming URI is in the list.

## Reusability

Because the callbacks are regional, any Quick tenant in a listed region can use
this same client ID URL unchanged. The trade-off is that they all appear to
Tableau as one OAuth client named "Amazon Quick Suite" — Tableau's audit log
records which *user* signed in, but not which tenant. An organisation wanting
its own audit identity should host its own copy of this file.

## No secrets here

The document contains only public client metadata: a name, redirect URIs, grant
types, and requested scopes. Publishing it is how the mechanism works, not an
oversight. Never add a client secret to this repo.

## Setup

1. Push this repo to GitHub as **public**.
2. Settings → Pages → Source `main`, folder `/ (root)`.
3. Verify before configuring Quick — Pages takes a minute to publish:

   ```bash
   curl -i https://marianachow0321.github.io/amazon-quick-tableau-connector/tableau.json
   ```

   Expect `HTTP 200` and `content-type: application/json`.

## Scopes requested

| Scope | Grants |
|---|---|
| `tableau:mcp:content:read` | Browse site content |
| `tableau:mcp:datasource:read` | Read data source metadata |
| `tableau:mcp:workbook:read` | Read workbooks |
| `tableau:mcp:view:read` | Read views |
| `tableau:content:read` | General content read |
| `tableau:viz_data_service:read` | Query data via the VizQL Data Service |

All read-only. Tableau's full scope list also includes `view:download`,
`insight:create`, and `insight_brief:create`; add them if you need them, but
note some tools require entitlements (Pulse Insight Briefs need Tableau+, the
full Metadata API needs Data Management) and will error at call time without
them.

## Alternatives, if this does not work

| Approach | Effort | Identity | Holds a credential |
|---|---|---|---|
| This document | ~15 min | Per-user, Tableau-enforced | No |
| Quick Desktop → Local MCP, `npx @tableau/mcp-server` with a PAT | ~5 min | Per-user | User's own env |
| Self-hosted Tableau MCP + Direct Trust connected app | Days | Per-user, asserted | Yes, one site-wide secret |
| Lambda proxy acting as its own OAuth server | Days | Either | Yes |

## Known unknown

It is not confirmed whether Tableau requires the client ID URL's origin to match
the redirect URI's origin. It does not here — the document is on `github.io`,
the callback is on `amazonaws.com`. The IETF draft does not require same-origin;
some implementations of the pattern do. If authorization fails with an
invalid-client or redirect-mismatch error, that constraint is the likely cause,
and the workaround is to serve this document from a host that can also receive
the callback — which in practice means a proxy, and the effort table above
applies.

## References

- Tableau MCP docs — https://tableau.github.io/tableau-mcp/
- Hosted Tableau MCP — https://tableau.github.io/tableau-mcp/docs/hosted-tableau-mcp
- `EXCLUDE_TOOLS` site setting, for admins wanting to disable tool groups
  server-side — see Admin controls on that page
