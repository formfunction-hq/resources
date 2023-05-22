# How Participation NFTs Work

## Introduction

In this post, we'll go over how the participation NFT feature works. Instead of providing a detailed description, we'll review the feature at a high level and give code pointers to the actual implementation.

## Overview

Here's a simple breakdown of how the feature works:

1. When listing a 1/1 NFT for auction, the creator can optionally attach a distinct participation NFT.
2. Upon listing, the participation NFT will be minted as a master edition NFT with unlimited supply and transferred to an account owned by the [Formfunction Gumdrop](https://github.com/formfunction-hq/formfunction-gumdrop) program.
3. When the auction ends, an account owned by the Formfunction Gumdrop program is updated with a Merkle root that will allow each bidder to claim a participation NFT. Then, each bidder (including the winning bidder) can mint exactly one edition of the participation NFT by passing the correct Merkle proof to the program.
5. After a few weeks, the master edition will be transferred back to the creator. After this point, bidders can no longer claim a participation NFT.

Check out this video for a more in-depth walkthrough.

[![](https://img.youtube.com/vi/WOBCTQ7Lfzw/maxresdefault.jpg)](https://www.youtube.com/watch?v=WOBCTQ7Lfzw)

## Code Pointers

### Program

The full program code is [here](https://github.com/formfunction-hq/formfunction-gumdrop).

- When listing, [`new_distributor`](https://github.com/formfunction-hq/formfunction-gumdrop/blob/main/programs/formfn-gumdrop/src/lib.rs#L64) is called. This simply initializes an account called `distributor`.
- After an auction is won, [`update_distributor`](https://github.com/formfunction-hq/formfunction-gumdrop/blob/main/programs/formfn-gumdrop/src/lib.rs#L90) is called in order to update the on-chain Merkle root.
- Then, within some timespan (which can be configured), bidders can claim their participation NFT by calling [`claim_edition`](https://github.com/formfunction-hq/formfunction-gumdrop/blob/main/programs/formfn-gumdrop/src/lib.rs#L212). This instruction takes a Merkle proof, which is validated against the Merkle root.
- After some timespan, the Formfunction backend calls [`close_distributor_token_account`](https://github.com/formfunction-hq/formfunction-gumdrop/blob/main/programs/formfn-gumdrop/src/lib.rs#L116), which returns the master edition to the creator, and [`close_distributor`](https://github.com/formfunction-hq/formfunction-gumdrop/blob/main/programs/formfn-gumdrop/src/lib.rs#L170) to reclaim rent.

### Backend

- When an auction is won, [a Merkle root is stored on-chain](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/src/rest/hasura/auctionWonUpdatePnftDrop.ts) so that all bidders of the auction can claim.
- The code to process finished participation NFTs and return them to their owners is [here](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/src/rest/intern/processFinishedPnftDrops.ts).
- Most information about participation NFTs is stored in the [`Claim` table](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/prisma/schema.prisma#L518-L532).

### Frontend

- The listing code is [here](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/components/listing/ListNftForAuctionSteps.tsx).
- Participation NFTs are displayed just like all other NFTs, using [`NftPage.tsx`](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/components/pages/common/nft/NftPage.tsx).
- The code that displays participation NFTs on user profiles is [here](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/components/pages/profile/NftsForAddress.tsx#L436).
