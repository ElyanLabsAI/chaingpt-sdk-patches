# PR 3 — fix(generalchat): align createChatBlob with documented /chat/stream endpoint

**Package:** `@chaingpt/generalchat@0.0.17`
**Type:** Doc/SDK behavioral mismatch
**Impact:** External integrators get contradictory guidance — public API docs say one endpoint, SDK uses another

## Recommendation

**Preferred (a):** Update the SDK so `createChatBlob()` posts to `/chat/stream`, because the current public docs consistently present `/chat/stream` as the single supported chat endpoint for both buffered and streaming responses.

**Also recommended (c):** Mention the endpoint alignment in release notes / CHANGELOG, since some users may know `/chat/blob` from older package behavior or private examples.

**Use (b) instead** only if ChainGPT confirms internally that `/chat/blob` is still the intended canonical backend contract and the public docs are wrong. In that case, the correct fix is a docs PR against the public docs, not an SDK change.

## Issue summary

The SDK source posts `createChatStream()` to `/chat/stream`:

- `src/index.ts` (lines 25–76)

The same SDK posts `createChatBlob()` to `/chat/blob`:

- `src/index.ts` (lines 129–165)
- `dist/index.js` (lines 137–175)

The public API reference says the service uses a single endpoint and that all chat requests should go to `POST https://api.chaingpt.org/chat/stream`, with response mode determined client-side:

- Single-endpoint statement: docs lines 99–103
- Endpoint section: docs lines 124–131
- `sdkUniqueId` remains optional: docs lines 140–144
- URL: https://docs.chaingpt.org/dev-docs-b2b-saas-api-and-sdk/web3-ai-chatbot-and-llm-api-and-sdk/javascript/api-reference

The public QuickStart says the same thing: "There is no separate `/chat` endpoint — all chat traffic goes to `/chat/stream`."

- URL: https://docs.chaingpt.org/dev-docs-b2b-saas-api-and-sdk/web3-ai-chatbot-and-llm-api-and-sdk/javascript/quickstart-guide

So today there is a public inconsistency between the published SDK/runtime and the published REST docs. Either the SDK is using a legacy endpoint that should be migrated, or the docs are wrong. Both can't be authoritative.

## Proposed fix (option a — change SDK)

If the docs are authoritative, change `createChatBlob()` to use the same endpoint as streaming mode and rely on the absence of `responseType: 'stream'` to keep returning buffered JSON.

**`src/index.ts`:**

```typescript
async createChatBlob(createChatDto: CreateChatDto): Promise<any> {
  const obj: any = createChatDto;
  Object.keys(createChatDto).forEach((key) => {
    const typedKey = key as keyof CreateChatDto;
    if (createChatDto[typedKey] === undefined) {
      delete obj[typedKey];
    }
  });

  obj['model'] = 'general_assistant';

  const instance: AxiosInstance = axios.create({
    baseURL: SERVER_URL,
    timeout: TIMEOUT,
    headers: { 'Authorization': `Bearer ${this.apiKey}` }
  });

  try {
    if (
      createChatDto?.useCustomContext === false &&
      createChatDto?.contextInjection &&
      Object.keys(createChatDto.contextInjection).length > 0
    ) {
      throw new contextInjectionError();
    }

    if (
      createChatDto?.contextInjection &&
      createChatDto.contextInjection?.cryptoToken === false &&
      createChatDto.contextInjection?.tokenInformation
    ) {
      throw new Error('Token Information Is Not Allowed Due To Crypto Token False');
    }

    const resp = await instance.post(`/chat/stream`, obj);
    return resp.data;
  } catch (error: any) {
    if (axios.isAxiosError(error)) {
      switch (error.response?.status) {
        case 403:
          throw new InvalidApiKeyError();
        case 429:
          throw new RateLimitExceededError();
      }
    }

    throw new GeneralChatError(
      error.response?.data?.message ? error.response.data.message : error
    );
  }
}
```

(The single substantive change is `instance.post('/chat/stream', obj)` instead of `instance.post('/chat/blob', obj)`.)

If ChainGPT confirms `/chat/blob` is still the intentional backend route, **do not merge the code change above**. In that case, the correct PR is a docs correction explaining that:

- `/chat/stream` is for streaming
- `/chat/blob` is for buffered JSON
- the current single-endpoint language is inaccurate

## Risks

- **Main risk:** if `/chat/blob` and `/chat/stream` are not true server-side equivalents for buffered mode, switching the SDK could break existing `createChatBlob()` consumers even though the public method name stays the same.
- **Safest migration path:**
  1. Validate against the live API that a non-streaming POST to `/chat/stream` returns the same JSON payload shape currently returned by `/chat/blob`
  2. Ship the endpoint swap in a patch release if payload parity is confirmed
  3. Mention the internal endpoint alignment in release notes
  4. If the backend still supports `/chat/blob`, consider keeping it temporarily as a fallback in case `/chat/stream` returns an unexpected non-200 or non-JSON response
- If parity is not confirmed quickly, prefer a docs PR first so external developers stop getting contradictory guidance.

## Test plan

1. With a valid API key, call `createChatBlob()` before and after the patch using the same prompt
2. Confirm the patched SDK still returns the same buffered JSON shape, especially `response.data.bot`
3. Confirm `createChatStream()` behavior is unchanged
4. Compare raw REST behavior manually:
   - `POST /chat/blob`
   - `POST /chat/stream` without stream handling
5. Verify error handling is unchanged for:
   - invalid API key
   - rate limit
   - context injection validation
6. If maintainers prefer the docs-fix route instead, verify the docs examples and prose consistently describe whichever endpoint contract the backend actually supports

## Pre-formatted PR description (option a)

```markdown
## Summary

This PR aligns `createChatBlob()` with the public ChainGPT chat API docs by sending buffered chat requests to `/chat/stream` instead of `/chat/blob`.

## Why

The currently published SDK and the currently published docs disagree:

- `createChatStream()` posts to `/chat/stream`
- `createChatBlob()` posts to `/chat/blob`
- the public API reference says `POST /chat/stream` is the single chat endpoint for both buffered and streaming responses, with response mode determined client-side

That inconsistency makes it unclear which contract external integrators should trust.

## What changed

- updated `createChatBlob()` to post to `/chat/stream`
- kept the public SDK method name and response handling unchanged

## Expected behavior

- `createChatBlob()` should still return the same buffered JSON response
- `createChatStream()` should continue returning a stream
- the difference remains client-side rather than endpoint-based, matching the public docs

## Verification

- compared buffered responses before/after the change
- confirmed `response.data.bot` remains accessible in blob mode
- confirmed streaming behavior is unchanged

## Risk / rollback

If the backend still treats `/chat/blob` and `/chat/stream` differently for non-streaming requests, this should not merge as-is. In that case the right fix is a docs correction instead of an SDK change.
```

## Bonus — there's also a separate error-handler bug nearby

While reviewing this file we also noticed the error handler at `dist/index.js` line ~84 reads `error.response.data.message` without a null-guard on `error.response`. When the API errors in certain ways (e.g., "Insufficient credits" returned with no `response` property), this throws `TypeError: Cannot read properties of undefined (reading 'data')` and masks the real cause from the developer.

Suggested patch (separate from this PR's scope, but trivial — happy to file as PR 4 if helpful):

```typescript
// before
throw new GeneralChatError(
  error.response?.data?.message ? error.response.data.message : error
);

// after — collapse to a single null-safe expression
throw new GeneralChatError(error?.response?.data?.message ?? error?.message ?? error);
```

This makes the SDK return a useful error message in all failure modes instead of crashing the error handler itself.
