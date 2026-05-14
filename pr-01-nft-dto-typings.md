# PR 1 — fix(nft): sync DTO typings with documented image and mint options

**Package:** `@chaingpt/nft@0.1.2`
**Type:** Type-safety bug
**Impact:** TypeScript users get type errors on documented SDK usage

## Issue summary

The shipped DTO for image/NFT generation only exposes `walletAddress`, a single-string `prompt`, and a small subset of generation fields:

- `dist/dto/createImage.dto.d.ts` (lines 1–11)

The README examples use additional fields that are not typed at all:

- **`style`** — README lines 28–36, 225–231, 281–283, 335–345, 401–411
- **`traits`** — README lines 232–249, 336–345, 402–411
- **`chainId` and `amount`** — README lines 333–345, 367–376, 399–411
- **Array `prompt`** for `generateMultipleImages()` — README lines 271–284

The shipped mint DTO only exposes `collectionId`, `name`, and `description`:

- `dist/dto/mintNft.dto.d.ts` (lines 1–4)

The README `mintNft()` example also uses **`ids`** and **`symbol`**, which are missing from the type:

- README lines 529–535

**Secondary mismatch:** `walletAddress` is required on the shared `createImageDto`, but the README's image-only examples omit it entirely, which makes the published typings reject the documented `generateImage()` usage:

- README lines 28–36, 56–64, 85–92, 141–147, 167–174, 193–201, 225–231

## Proposed fix

Update the source DTOs, then rebuild `dist`.

**`src/dto/createImage.dto.ts`:**

```typescript
export interface TraitOptionDto {
  value: string;
  ratio: number;
}

export interface TraitDto {
  trait_type: string;
  value: TraitOptionDto[];
}

export interface createImageDto {
  walletAddress?: string;
  prompt: string | string[];
  model: 'velogen' | 'Dale3' | 'nebula_forge_xl' | 'VisionaryForge';
  enhance?: '1x' | '2x' | 'original';
  steps?: number;
  height?: number;
  width?: number;
  upscale?: boolean;
  image?: string;
  isCharacterPreserve?: boolean;
  style?: string;
  traits?: TraitDto[];
  chainId?: number;
  amount?: number;
}
```

**`src/dto/mintNft.dto.ts`:**

```typescript
export interface mintImageDto {
  collectionId: string;
  ids: number[];
  name: string;
  description: string;
  symbol: string;
}
```

### Notes for maintainers

This package would be even more accurate with separate DTOs for `generateImage`, `generateMultipleImages`, `generateNft`, and `generateNftWithQueue`, because the current shared `createImageDto` has to be looser than any one method. The patch above keeps the change minimal and backwards-compatible while making the published declarations match the README. A future cleanup PR could split these into per-method DTOs for stricter typing.

## Test plan

1. Run `npm run build` in the SDK package so `dist/dto/*.d.ts` regenerates
2. Copy the README examples for:
   - `generateImage()`
   - `generateMultipleImages()`
   - `generateNft()`
   - `generateNftWithQueue()`
   - `mintNft()`
3. Paste them into a small TypeScript smoke test and run `tsc --noEmit`
4. Verify that:
   - `style`, `traits`, `chainId`, `amount`, `ids`, and `symbol` no longer produce type errors
   - `prompt` accepts `string[]` for `generateMultipleImages()`
   - image-only examples no longer require `walletAddress`
5. Confirm this PR changes typings only and produces no runtime diff besides regenerated declaration files

## Pre-formatted PR description

```markdown
## Summary

This PR syncs the published `@chaingpt/nft` TypeScript DTOs with the SDK's own README examples.

Today the shipped declarations are missing several documented request fields:

- `style`
- `traits`
- `chainId`
- `amount`
- `ids`
- `symbol`
- array `prompt` for `generateMultipleImages()`

There is also a typing mismatch where `walletAddress` is required on the shared `createImageDto`, even though the documented `generateImage()` examples do not send it.

## What changed

- widened `createImageDto.prompt` from `string` to `string | string[]`
- made `walletAddress` optional on the shared DTO
- added `style`, `traits`, `chainId`, and `amount` to `createImageDto`
- added `ids` and `symbol` to `mintImageDto`

## Why

Right now a TypeScript user following the official README gets type errors against valid documented SDK usage. This PR brings the published declarations back in sync with the docs without changing runtime behavior.

## Verification

- rebuilt the package declarations
- checked the README examples for `generateImage()`, `generateMultipleImages()`, `generateNft()`, `generateNftWithQueue()`, and `mintNft()` against the updated DTOs

## Notes

The current SDK uses one shared DTO across several methods with slightly different request shapes. This PR keeps the fix intentionally minimal and backwards-compatible. A later cleanup could split these into per-method DTOs for stricter typing.
```
