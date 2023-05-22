# How to Build a Solana NFT Marketplace

## Introduction

In this post, we'll go over some important factors to consider when building an NFT marketplace like Formfunction. For the most part, building an NFT marketplace is similar to building a web2 marketplace (e.g. there's a frontend that uses an API to read and write data stored in a database). However, there are some key difference, which we'll cover in more depth below.

This post is not meant to be a comprehensive guide—please refer to the [source code](https://github.com/formfunction-hq/formfunction-monorepo) if you'd like to dig in further or learn more about the specific tech stack Formfunction used.

## On-chain vs. Off-chain

When building an NFT marketplace, it is hard (although not impossible) to fully decentralize every aspect of the product. For example, Formfunction used a Postgres database and a private GraphQL API in order to serve information to the frontend. This approach is much more performant and flexible than directly querying on-chain data. That being said, there are certain things that should also be stored on-chain:

- Some [NFT metadata](https://docs.metaplex.com/programs/token-metadata/token-standard), like the NFT's name, should be stored directly on-chain.
- Other NFT metadata, like the NFT's asset (e.g. image/video/gif) should be stored off-chain, preferably using a decentralized solution like [Arweave](https://www.arweave.org/). Note that "off-chain" here means "not on Solana," and does NOT mean "not on any chain" (Arweave is also a blockchain).

Note that this information is typically *also* stored off-chain to make it easier and faster to query for. For example, Formfunction stored information about NFTs in [Postgres](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/prisma/schema.prisma#L218).

Unlike web2 marketplaces, where payments are typically faciliated by Stripe, transactions on web3 marketplaces go through smart contracts. For example, bidding on NFTs and buying editions went through [Formfunction's auction contract](https://github.com/formfunction-hq/formfunction-auction-house). The high-level flow for spinning up a smart contract on Solana is as follows:

1. Create a new [Anchor program](https://www.anchor-lang.com/).
2. Implement your program.
3. Create an SDK that your frontend and backend can use to interact with the program. Formfunction used a TypeScript SDK.
4. Deploy your program to devnet for testing.
5. Once you are confident your program is good to go, deploy it to mainnet.
6. When making changes to the program, make sure to keep [backwards compatibility](https://formfunction.medium.com/how-to-make-backwards-compatible-changes-to-a-solana-program-45015dd8ff82) in mind.

### Keeping Data in Sync

One of the trickier parts about building a web3 product is keeping the on-chain data in sync with your database. Here's how we approached this problem at Formfunction.

There are two main types of transactions to keep in mind here. The first type, transactions that are initiated via your frontend, are easier to handle. Whenever someone submitted a transaction via Formfunction's frontend, we first [sent the transaction](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/components/modal/BidModal.tsx#L221-L241). After the transaction was confirmed, we then sent a [request to our API](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/components/modal/BidModal.tsx#L249-L279) to update our database. After the request completes, the frontend immediately updated with any new data. Note that it's possible that the frontend sends the transaction and the transaction gets confirmed, but the request to our API never gets kicked off (e.g. if the user closes the window after sending the transaction but before the API request). These transactions will be handled alongside the second type of transactions.

The second type are transactions that do NOT go through your frontend, and instead hit the program directly. We had a [simple endpoint](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/src/rest/intern/syncAuctionTxs.ts) that we called every minute using [Airplane](https://www.airplane.dev/) to ensure these transactions were also synced to our database. This is important because otherwise, our frontend would not show the correct information (e.g. if a bid was placed by directly hitting our program, the frontend needs to reflect that). There are a number of different ways these transactions could be synced, but for us, periodically polling for missing transactions was simple and got the job done.

Lastly, keep in mind that an NFT marketplace that deals primarily with PFPs, like Magic Eden or Tensor, likely has different requirements and thus may handle data syncing in a much different manner.

## Authentication

The easiest way to add authentication to a Solana web3 dapp is to use the [wallet-adapter](https://github.com/solana-labs/wallet-adapter). Using a more web2 friendly approach like [Magic auth](https://magic.link/) is also a viable option.

Note that Formfunction didn't use either of these options; instead, we implemented [our own](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/components/modal/ConnectWalletModal.tsx) authentication flow so that we had more control over it.

If you have a database, it's likely that you'll also need to support [message signing](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/utils/solana/misc/signAuthMessage.ts). The basic idea here is that when a user makes an API request, the request headers will include their public key and a message signed by them (i.e. a signature). The server can then [verify](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/src/utils/auth/verifySignature.ts) that the message was indeed signed by the public key. For example, if there is an API endpoint called `modifyUserProfile`, then it should only succeed if the user's signature was signed by them; this prevents one user from modifying another user's profile.

Ideally, the message the user signs should have a nonce to prevent [replay attacks](https://en.wikipedia.org/wiki/Replay_attack). Formfunction never implemented this, so keep that in mind when referencing the source code.

## Bot Protection

In order to prevent hyped mints from being bought out by a handful of bots instead of real people, it is important to implement bot protection mechanisms. There are a few ways of doing this; see Candy Machine's [bot tax](https://docs.metaplex.com/programs/candy-machine/available-guards/bot-tax) and Strata's [dynamic pricing protocol](https://docs.strataprotocol.com/launchpad/dynamic-pricing-mint#:~:text=With%20a%20Strata%20Dynamic%20Pricing,matches%20the%20fall%20in%20price.) for a couple examples.

In this section, we'll on how Formfunction implemented bot protection for buying editions. This approach can also be applied to other minting mechanisms, e.g. a Candy Machine style mint.

The main idea is to [require some centralized keypair to sign transactions](https://github.com/formfunction-hq/formfunction-auction-house/blob/main/programs/formfn-auction-house/src/instructions/buy_edition_v2.rs#L183). This makes it impossible for bots to hit your program directly, and instead [funnels them through your API](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/src/rest/signVersionedTransactionWithAntiBotAuthority.ts). Once you have funneled bots through your API, you have many powerful options for bot management, such as [Cloudflare's bot management](https://www.cloudflare.com/products/bot-management/). For example, you can rate limit requests based on public key, IP address, and more.

Here's a more detailed breakdown of how Formfunction implemented this approach:

1. [The frontend hits the API in order to get a transaction signed by the Formfunction anti-bot authority.](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/frontend/src/utils/transactions/signVersionedTransactionWithAntiBotSigner.ts). The backend code is [here](https://github.com/formfunction-hq/formfunction-monorepo/blob/main/packages/server/src/rest/signVersionedTransactionWithAntiBotAuthority.ts).
2. The program [checks to make sure the anti-bot authority signed the transaction](https://github.com/formfunction-hq/formfunction-auction-house/blob/main/programs/formfn-auction-house/src/instructions/buy_edition_v2.rs#L183).
3. If they did not sign, we implement a [bot tax](https://github.com/formfunction-hq/formfunction-auction-house/blob/main/programs/formfn-auction-house/src/instructions/buy_edition_v2.rs#L198) to discourage bad behavior.

Note that Formfunction did not require this form of bot protection for all editions; users could turn bot protection on and off at their discretion.