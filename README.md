[![BCOS Certified](https://img.shields.io/badge/BCOS-Certified-brightgreen?style=flat)](BCOS.md)

# ChainGPT SDK Patches — drafted by Elyan Labs (May 2026)

Three drop-in patches for ChainGPT's published npm SDKs, surfaced during our 2-week PoC integration.

We can't open these as PRs publicly because `@chaingpt/generalchat` and `@chaingpt/nft` source repos are not on public GitHub. This repo is the handoff format — ChainGPT's engineering team can apply the patches manually OR open the SDK source repos and we'll re-file as proper PRs.

## Patches

| File | Subject | Severity |
|---|---|---|
| [`pr-01-nft-dto-typings.md`](pr-01-nft-dto-typings.md) | `fix(nft): sync DTO typings with documented image and mint options` | Type-safety bug — TypeScript users get errors on documented usage |
| [`pr-02-findchatdto-optional.md`](pr-02-findchatdto-optional.md) | `fix(generalchat): make FindChatDto.sdkUniqueId optional` | Type-safety bug — required field is documented as optional |
| [`pr-03-chat-blob-stream.md`](pr-03-chat-blob-stream.md) | `fix(generalchat): align createChatBlob with documented /chat/stream endpoint` | Doc/SDK mismatch — runtime uses legacy endpoint |

Each patch file contains:
- Issue summary with file:line references
- Proposed source code (TypeScript, ready to paste)
- Test plan
- Risks (where applicable)
- Pre-formatted PR description (for when SDK repos open)

## How to apply

For each patch:

1. Open the relevant SDK package in your private source repo
2. Apply the changes from the "Proposed fix" section
3. Run `npm run build` to regenerate `dist/`
4. Run the test plan steps to verify
5. Publish a patch version (e.g., `0.0.18` for generalchat, `0.1.3` for nft)

If you'd rather we open these as proper PRs, point us to the SDK source repo and we'll re-file with the same content in the standard PR format.

## Context

These patches came out of the Elyan Labs ChainGPT engagement (April–May 2026). Full retrospective covering the engagement, lessons, and recommendations is at https://github.com/ElyanLabsAI/chaingpt-poc/blob/main/RETROSPECTIVE.md (forthcoming public repo).

## Contact

Scott Boudreaux — scott@elyanlabs.ai
Elyan Labs — https://elyanlabs.ai

## License

CC BY 4.0 (this documentation). The proposed code changes themselves are MIT-licensed and may be incorporated into ChainGPT's MIT-licensed SDKs without restriction.