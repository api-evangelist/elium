---
name: Publish and update an Elium story
description: Create a knowledge article (story) in Elium, attach a file, revise it as a new version, and move it through approval — without double-posting.
api: graphql/elium-graphql-operations.yml
generated: '2026-08-17'
method: generated
source:
  - https://learn.elium.com/reference/
  - https://learn.elium.com/en/api/how_to/post_a_story
  - https://learn.elium.com/en/api/how_to/post_a_story_with_a_file
  - https://learn.elium.com/en/api/how_to/edit_a_story
operations:
  - storyPost
  - storyEdit
  - storySpace
  - storyApprovalRequest
  - storyApprovalStatus
  - storyExpire
  - storyLock
  - storyUnlock
  - story
---

# Publish and update an Elium story

Writes go over GraphQL only — Elium's MCP server has no write tools.

## Before you start

- Endpoint `https://{platform}.elium.com/graphql`; OAuth 2.0 bearer, scope `apiv1`; Enterprise plan only.
- Every mutation takes a single `input` argument of its own input type: `storyPost(input: StoryPostInput!)`,
  `storyEdit(input: StoryEditInput!)`, and so on. There are 244 input types — read the one you need in the
  reference rather than guessing field names.
- Writes execute **as the token's user**. That user's space membership and role decide what you can create
  and where.

## READ THIS FIRST — there is no safe retry

**Elium publishes no idempotency mechanism.** No `Idempotency-Key`, no request key, nothing in any of the
206 mutations. `storyPost` called twice creates two stories.

So, on any timeout or transport error during a write:

1. Do **not** retry blind.
2. Query for the effect first — `me { search(query: {text: "<title>", sort: ..., limit: 10, start: 0}) }`
   or read the space's recent stories.
3. Retry only if you can prove the story does not exist.

## Step 1 — create the story

```graphql
mutation storyPost($input: StoryPostInput!) {
  storyPost(input: $input) {
    story { slug version { title } status isPublished }
    errors { __typename path }
  }
}
```

The returned `story.slug` is an **`Int`** — Elium's identifier for a story is an integer it calls a slug.
Capture it; every later operation keys on it.

For a story with an attachment, follow the provider's `post_a_story_with_a_file` guide — the asset is
uploaded and then referenced from the story input. Do not invent an upload endpoint.

If the story needs a template, set the story **schema** (Elium's term for a content template). Look the
schema up with `storySchema(slug: Int!)` first; converting later means `storySchemaConvert`, which is a
heavier operation.

## Step 2 — place it and set visibility

`storySpace` — "Change Space and visibility of a story". Space placement drives who can see the story, so
treat it as part of publishing, not an afterthought.

Related: `storyGrant` (explicit grants), `storyPublicShare` (share publicly / edit public-share settings),
`storyInviteContributors`.

## Step 3 — revise it

`storyEdit` does not patch in place: it **creates a new version** of the story. Version history is
first-class — `Story.versions`, `Story.editVersion`, `Story.editVersionUUID`, plus
`storyVersionRestore` and `storyVersionDelete`.

When several actors may edit concurrently, take a lock: `storyLock`, keep it alive with `storyLockRefresh`,
release with `storyUnlock`. `LockedContentError` and `StoryLockError` tell you someone else holds it.
`storyUpdated` and `storyLocked` subscriptions let you watch this live.

`storyNoModificationsError` (`StoryNoModificationsError`) means your edit changed nothing — treat it as a
no-op, not a failure.

## Step 4 — approval and freshness

- `storyApprovalRequest` — request approval on a story.
- `storyApprovalAssign`, `storyApprovalStatus`, `storyApprovalCancel` — move it through the flow.
  (`storyApprovalEdit` is deprecated in the published reference — do not use it.)
- `storyExpire` — set an expiration date; passing null clears it. This drives Elium's expiration-management
  and verification-reminder behaviour, so set it for anything that goes stale.

## Step 5 — check the errors array, always

HTTP 200 is not success. Read `data.storyPost.errors[]` and switch on `__typename`:

| `__typename` | What it means for you |
|---|---|
| `PermissionError` | The token's user may not write here. Stop; do not retry. |
| `ObjectNotFoundError` | A referenced id in `path` does not exist. |
| `InvalidInputError` / `InvalidFormatError` / `MissingTypeInputError` | Fix the input at `path`. |
| `LockedContentError` / `StoryLockError` | Another editor holds the lock. Wait or take it. |
| `LimitsViolationError` / `InvalidPlanLimitsError` | A contractual plan limit was hit — the actual numbers are not published. |
| `ObjectAlreadyExistsError` | Likely your own earlier attempt succeeded. Verify before retrying. |
| `StoryNoModificationsError` | No-op edit. |

Full catalogue of all 31 error types: `errors/elium-error-codes.yml`.

## Cleanup and bulk work

`storyArchive`, `storyDelete`, `storyUndelete`; bulk equivalents `bulkStoryArchive`, `bulkStoryDelete`,
`bulkStoryMove`, `bulkStorySchemaConvert`. `storyMigrate` is deprecated — avoid it.
