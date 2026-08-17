---
name: Search the Elium knowledge base and read an article
description: Run a semantic or filtered search across an Elium platform and retrieve the full content of the article you found, over GraphQL or over the MCP server.
api: graphql/elium-graphql-operations.yml
generated: '2026-08-17'
method: generated
source:
  - https://learn.elium.com/reference/
  - https://learn.elium.com/en/api/how_to/get_a_story
  - https://learn.elium.com/en/api/how_to/answer
  - https://help.elium.com/en/articles/12767574-mcp
operations:
  - me.answer.semanticSearch
  - me.search
  - story
  - searchFeedback
---

# Search the Elium knowledge base and read an article

Elium calls a unit of knowledge content a **Story**. Everything below is grounded in operations that
exist verbatim in Elium's published GraphQL reference.

## Before you start

- **The endpoint is per tenant.** There is no shared vendor host. You need the customer's own platform
  hostname: `https://{platform}.elium.com/graphql`.
- **The API is Enterprise-only.** Team and Corporate plans cannot call it at all. Confirm the tenant is on
  Enterprise before debugging auth.
- **Auth is OAuth 2.0**, single scope `apiv1`, sent as `Authorization: Bearer <access_token>`. Tokens last
  3600 seconds. Get a client id/secret by having an administrator create a Connected App in the Security
  tab. See `authentication/elium-authentication.yml`.
- **Results are permission-scoped to the token's user**, not to the app. Two tokens will legitimately return
  different results for the same query. Never report a permission-filtered result as "not found".

## Step 1 — choose the right search

| Intent | Operation | Input |
|---|---|---|
| Natural-language question, meaning-based | `me { answer { semanticSearch } }` | `SemanticSearchQuery`: `text` (String!), `limit` (Int!), `minScore` (Int), `method` (AnswerMethod) |
| Known filters: keyword, template, date, sort | `me { search }` | `StorySearchQuery`: `text`, `schemas`, `tags`, `hashtags`, `spaces`, `spaceGroups`, `langs`, `users`, `facets`, `dateRange`, `sort` (required), `limit` (required), `start` (required) |
| A direct answer rather than a result list | `me { answer { answer } }` | returns `Answer` |
| Scoped to one space | `space(slug: Int!) { search }` | same `StorySearchQuery` |
| Scoped to a curated corpus | `SmartAssistant.semanticSearch` | `SemanticSearchQuery` |

Use `semanticSearch` when the user asked a question. Use `me { search }` when the user gave you
constraints — Elium's word for a content template is **schema**, so "only onboarding-template articles"
means `schemas: [ID!]`, not a text match.

```graphql
query semantic($q: SemanticSearchQuery!) {
  me { answer { semanticSearch(query: $q) { ... } } }
}
```

`sort`, `limit` and `start` are **non-null** on `StorySearchQuery`. Omitting them is the most common
first-call failure; there is no default.

## Step 2 — read the full article

Search results identify a story by its **`slug`, which is an `Int`, not a string**. Pass it straight to the
root query:

```graphql
query story($slug: Int!) {
  story(slug: $slug) {
    slug
    status
    isPublished
    updated
    version { title }
    user { name }
    space { name }
    tags { name }
    schema { name }
  }
}
```

`Story` exposes 54 fields. Select only what you need — several fields (`comments`, `annotations`,
`followers`, `viewers`, `versions`) are cursor connections that will fan out your response.

Do **not** use `statusStory` — it is deprecated in the published reference.

## Step 3 — page through more results

Pagination is Relay cursor connections: pass `first`/`after` (or `last`/`before`) and read
`pageInfo.hasNextPage` plus `pageInfo.endCursor`. Never synthesise a cursor; only echo one Elium gave you.
`me { search }` itself pages with `start`/`limit` rather than cursors — the two styles coexist, so check
which one the field you are calling accepts.

## Step 4 — handle errors correctly

**HTTP 200 does not mean success.** Handled errors come back in-band:

- Query errors appear in the top-level GraphQL `errors` array.
- Mutation errors appear in `data.<mutation>.errors[]`, each with `__typename` (which error) and `path`
  (which input field).

Switch on `__typename`. The ones you will actually hit here are `ObjectNotFoundError` (bad slug),
`PermissionError` (the token's user cannot see it — say so, do not retry), `InvalidInputError` and
`InvalidFormatError`. Full list: `errors/elium-error-codes.yml`.

## Step 5 — close the loop when you find nothing

If the knowledge base could not answer, call `searchFeedback` — "Post a feedback about missing content in
search results for your administrators". That is the documented way to report a gap, and it feeds Elium's
gap-detection feature. `answerFeedback` gives feedback on a specific AI answer.

## Retry and idempotency

**Elium has no idempotency mechanism** — no idempotency key, anywhere in 206 mutations. Reads here are
safe to retry freely. Never blind-retry a write (see the publishing skill). No rate limits are published
and no rate-limit response headers exist, so back off on your own schedule and treat throttling as
unobservable. See `conventions/elium-conventions.yml`.

## Doing this over MCP instead

If the tenant has enabled the MCP server at `https://{platform}.elium.com/services/mcp`, the same three
jobs are exposed as tools, and only these three:

- `FindRelevantArticles` — semantic search from a natural-language question (backs onto `me.answer.semanticSearch`)
- `FindSpecificArticles` — filtered search: keyword, template, date range, sort order (backs onto `me.search`)
- `ReadFullArticle` — full article content with key attributes (backs onto `story`)

All three are read-only. There are **no MCP write tools** — if the task requires creating or editing
content, you must use GraphQL. Mapping detail: `mcp/elium-tool-crosswalk.yml`.
