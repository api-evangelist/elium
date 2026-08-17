---
name: Connect an AI assistant to Elium over MCP
description: Stand up an OAuth Connected App and point ChatGPT, Claude, or your own agent at a tenant's Elium MCP server — and know exactly which three tools you get.
api: mcp/elium-mcp.yml
generated: '2026-08-17'
method: generated
source:
  - https://help.elium.com/en/articles/12767574-mcp
  - https://elium.com/product/connect-your-ai
  - https://learn.elium.com/en/api/getting_started/authentication
operations:
  - FindRelevantArticles
  - FindSpecificArticles
  - ReadFullArticle
  - connectedAppCreate
---

# Connect an AI assistant to Elium over MCP

Elium shipped a remote MCP server on 3 June 2026. This is the only agent-native surface Elium publishes —
there is no A2A agent card, no `/.well-known/mcp.json`, and no OAuth discovery document.

## What you are pointing at

```
https://{platform}.elium.com/services/mcp
```

**The endpoint is genuinely per customer.** Elium's own wording: "each Elium instance exposes its own secure
MCP connection". `{platform}` is the customer's Elium subdomain. There is no shared vendor URL — if you do
not know the tenant name, you cannot proceed, and you must not guess one.

Transport is HTTPS. Authorization is OAuth 2.0. The API and MCP surface are **Enterprise-plan only**.

## Step 1 — create the Connected App (administrator)

A tenant administrator, in Elium's **Security** tab of the administration panel, creates a Connected App
with:

- a **name**
- a **redirect URL** — this must match the client you are wiring:
  - ChatGPT: `https://chatgpt.com/connector_platform_oauth_redirect`
  - Claude: `https://claude.ai/api/mcp/auth_callback`
- optionally the **trusted** flag, which skips the end-user approval screen for pre-approved corporate apps

That yields a **Client ID** and **Client Secret**. Dynamic Client Registration (DCR) is available from
Elium on request if you would rather not hand-provision.

The equivalent GraphQL operations are `connectedAppCreate`, `connectedAppEdit`, `connectedAppDelete`.

## Step 2 — wire the client

**ChatGPT:** enable Developer Mode under Connectors, add a custom connector, set the MCP URL above, choose
**OAuth** from the authentication dropdown, accept the connection.

**Claude:** add the custom connector with the MCP URL, supply the Client ID and Client Secret from the
Elium Connected App, then enable it in the conversation.

**Your own agent:** standard OAuth 2.0 against the tenant's endpoints —
`https://{platform}.elium.com/oauth/authorize` and `/oauth/token`, scope `apiv1`, `Authorization: Bearer`,
access tokens valid 3600 seconds, refresh supported, `/oauth/revoke` to kill a token pair. Note that Elium
serves **no** RFC 8414 or OIDC discovery document, so these endpoints must be configured, not discovered.

## Step 3 — know your tool budget: three, all read-only

| Tool | What it does |
|---|---|
| `FindRelevantArticles` | Semantic (meaning-based) search of the knowledge base from a natural-language question |
| `FindSpecificArticles` | Filtered search: keyword, template, date range, custom sort order |
| `ReadFullArticle` | Retrieves complete article content with key attributes |

That is the whole MCP surface. Plan around two consequences:

1. **No writes.** Elium's GraphQL API has 206 mutations and *none* of them is exposed over MCP. An agent
   that must create, edit, tag, or approve content needs a GraphQL client, not an MCP client.
2. **No events.** MCP has no subscription surface here; the 18 GraphQL subscriptions are unreachable from
   MCP. An agent cannot be notified that a story changed — it must poll.

Tool `inputSchema`s are not published: `tools/list` requires an authenticated session on a real tenant, so
discover them at runtime after connecting rather than hard-coding argument names.

## Step 4 — reason correctly about permissions

Elium is explicit: "Every tool runs with the permissions of the authenticated user, so an assistant can only
ever access content that user is already allowed to see in Elium."

Practical consequences for an agent:

- Results are **not** the whole knowledge base. They are one user's slice of it.
- An empty result may mean "does not exist" **or** "you may not see it". Never report the second as the
  first, and never escalate by trying another token.
- Two users asking the same question will legitimately get different answers. Do not cache across users.

## Step 5 — grounding and citation

Elium's product position is governed knowledge with cited, permission-aware answers. Match it: when you
answer from `FindRelevantArticles`, follow up with `ReadFullArticle` and cite the story you actually read.
Do not answer from a search snippet alone, and do not blend Elium content with untraced sources without
saying which is which.

## Falling back to GraphQL

Anything outside those three tools means the GraphQL API at `https://{platform}.elium.com/graphql`, same
OAuth token and scope. See `skills/elium-search-knowledge-base.md` for reads and
`skills/elium-publish-a-story.md` for writes — and note that Elium has no idempotency mechanism, so writes
are not safe to blind-retry.
