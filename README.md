# Dotsama Multichain Indexer

A [Subsquid](https://docs.subsquid.io) ETL pipeline that indexes the **Dotsama NFT Marketplace** across **Astar** and **Moonbeam** into one Postgres database and serves it over one GraphQL API.

The marketplace is a LooksRare-style exchange: makers sign orders off-chain and takers settle on-chain. This indexer turns the raw settlement logs into queryable trade history, royalty flows and order-cancellation state for the marketplace frontend and analytics.

## Why it's interesting

Astar and Moonbeam are **Substrate parachains with EVM compatibility (Frontier)**. The exchange contract emits ordinary EVM logs, but on these chains each log arrives wrapped inside a Substrate `EVM.Log` event. So the indexer:

1. Consumes **Substrate** archives (`SubstrateBatchProcessor`), not EVM archives.
2. Unwraps each `EVM.Log` via `@subsquid/frontier`, filters by contract address and `topic0`, and decodes it with typegen-generated ABI bindings.
3. Tags each row with a `chainId` and writes both chains into **one schema**, while keeping separate processor state (`astar_processor` / `moonbeam_processor`). Each chain can then progress, reorg and restart independently.

```
          Astar archive + RPC                   Moonbeam archive + RPC
                 │                                       │
   SubstrateBatchProcessor (EVM.Log)       SubstrateBatchProcessor (EVM.Log)
                 │  frontier unwrap + ABI decode         │
                 └──────────────┬────────────────────────┘
                                ▼
                 Postgres (shared entities, chainId-tagged,
                          per-chain processor state)
                                ▼
                       GraphQL API (:4350)
```

## Indexed events

| Entity | Source event | Captures |
|---|---|---|
| `TakerBid` / `TakerAsk` | Order settlement | order hash & nonce, maker, taker, strategy, currency, collection, tokenId, amount, price, block, tx |
| `RoyaltyPayment` | Royalty transfer on sale | collection, tokenId, recipient, currency, amount |
| `CancelAllOrders` | Nonce floor raised | user, new min nonce |
| `CancelMultipleOrders` | Specific nonces cancelled | user, order nonces |

Events are batched per block range and written in bulk, and hot (unfinalized) blocks are supported, so the API stays near chain head and still survives reorgs.

## Run it

Requirements: Node.js 18+, Docker, `@subsquid/cli` (`npm i -g @subsquid/cli`).

```bash
git clone https://github.com/yasinadil/dotsama-multichain-indexer
cd dotsama-multichain-indexer
npm ci

cp .env.example .env          # set RPC_ENDPOINT_ASTAR and RPC_ENDPOINT_MOONBEAM

sqd up                        # start Postgres
sqd migration:apply
sqd build
sqd run .                     # both processors + GraphQL server
```

GraphiQL is available at <http://localhost:4350/graphql>. To run services individually:

```bash
sqd process:astar
sqd process:moonbeam
sqd serve
```

Example query: latest sales across both chains:

```graphql
query {
  takerBids(orderBy: timestamp_DESC, limit: 10) {
    chainId collection tokenId price taker maker
  }
}
```

## Stack

TypeScript · Subsquid (ArrowSquid) · `@subsquid/substrate-processor` · `@subsquid/frontier` · TypeORM · PostgreSQL · GraphQL · Docker
