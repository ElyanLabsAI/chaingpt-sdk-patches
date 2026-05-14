# PR 2 — fix(generalchat): make FindChatDto.sdkUniqueId optional

**Package:** `@chaingpt/generalchat@0.0.17`
**Type:** Type-safety bug
**Impact:** TypeScript users get type errors for documented `getChatHistory()` usage

## Issue summary

The published history-query DTO requires `sdkUniqueId`:

- `dist/dto/findChat.dto.d.ts` (lines 1–6)

The same package README documents `getChatHistory()` without `sdkUniqueId` when fetching all history for an API key:

- `README.md` (lines 409–428)

The README only requires `sdkUniqueId` when filtering down to a specific user/session:

- `README.md` (lines 432–447)

Public docs say the same thing — `sdkUniqueId` is optional when retrieving history and only needed to scope to a specific conversation/session:

- https://docs.chaingpt.org/dev-docs-b2b-saas-api-and-sdk/web3-ai-chatbot-and-llm-api-and-sdk/javascript/api-reference (lines 300–317)

**Internal consistency issue:** `CreateChatDto` already marks `sdkUniqueId` optional for create calls:

- `src/dto/createChat.dto.ts` (lines 112–118)

So the same field is consistently optional everywhere except `FindChatDto`, where it's required. That looks like a typing oversight, not an intentional contract.

## Proposed fix

Update the source DTO, then rebuild `dist`.

**`src/dto/findChat.dto.ts`:**

```typescript
export interface FindChatDto {
  sortOrder?: string;
  sortBy?: string;
  limit?: number;
  offset?: number;
  sdkUniqueId?: string;
}
```

(Single change: add `?` to `sdkUniqueId`.)

## Test plan

1. Run `npm run build` in the SDK package
2. Paste the README `getChatHistory()` example without `sdkUniqueId` (`README.md` lines 418–424) into a TypeScript test file and run `tsc --noEmit`
3. Paste the README example with `sdkUniqueId` (`README.md` lines 441–447) into the same test file
4. Verify both examples type-check
5. Optionally hit the runtime with both calls and confirm behavior is unchanged:
   - no `sdkUniqueId` → all available history for the API key
   - with `sdkUniqueId` → history filtered to that session

## Pre-formatted PR description

```markdown
## Summary

This PR makes `FindChatDto.sdkUniqueId` optional in `@chaingpt/generalchat`.

## Why

The current declaration requires `sdkUniqueId`, but the SDK README and public API docs both document `getChatHistory()` as working without it when you want all history for the API key.

That means a TypeScript consumer currently gets a type error for documented usage like:

\`\`\`ts
await generalchat.getChatHistory({
  limit: 10,
  offset: 0,
  sortBy: "createdAt",
  sortOrder: "ASC"
});
\`\`\`

`CreateChatDto.sdkUniqueId` is already optional, so this brings the two DTOs into consistent alignment with each other and with the docs.

## What changed

- changed `sdkUniqueId` from required to optional in `FindChatDto`

## Verification

- checked the documented `getChatHistory()` example without `sdkUniqueId`
- checked the documented session-filtered example with `sdkUniqueId`
- no runtime behavior changes; this is a typing fix only
```
