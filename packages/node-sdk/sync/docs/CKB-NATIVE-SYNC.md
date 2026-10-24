# Native CKB sync integration

This adapter adds CKB as a parallel source in EffectStream's native synchronization service. It uses the existing protocol config registries, `genSyncProtocols`, `BaseDataFetcher`, `SyncState`, orchestration merge, primitive transition, and per-output resume marker. It does not start a second polling loop or publish a separate event queue.

## Configuration

The network schema accepts only `devnet`, `testnet`, or `mainnet` and requires `rpcUrl` and the expected 32-byte `genesisBlockHash`. A CKB parallel protocol requires a name, `startBlockHeight`, polling interval, confirmation depth, reorganization window, and a target script with an explicit `lock` or `type` role. `stepSize` cannot exceed `reorgWindow`. The current adapter supports exactly one configured primitive entry per CKB protocol.

Example protocol and primitive entries:

```ts
{
  name: "ckb-devnet",
  type: ConfigNetworkType.CKB,
  rpcUrl: "http://127.0.0.1:8114",
  network: "devnet",
  genesisBlockHash: "0x<64 hex characters>",
}

{
  name: "ckb-cell-created",
  type: ConfigSyncProtocolType.CKB_RPC_PARALLEL,
  startBlockHeight: 0,
  pollingInterval: 3_000,
  confirmationDepth: 6,
  reorgWindow: 24,
  stepSize: 24,
  targetScript: {
    role: "lock",
    script: { codeHash: "0x<64 hex characters>", hashType: "type", args: "0x1234" },
  },
}

{
  name: "cell-created",
  type: "CKB:CellCreated",
  startBlockHeight: 0,
  stateMachinePrefix: "cell-created",
}
```

The full v1 event is carried as the primitive output payload. Capacity and output index remain decimal strings, and transaction, block, script, and data hashes remain full `0x` values. The existing shared `contractAddress` metadata field is populated with the matched script hash as a source identifier; it is not an EVM address. The CKB event does not imply an application action or trustless proof.

EffectStream's page-height and block-number types are JavaScript numbers. The CKB adapter checks block heights and timestamps against `Number.MAX_SAFE_INTEGER` and rejects values it cannot represent without loss. It adds CKB-specific block-number and block-hash nominal types to `@effectstream/utils`. Since the upstream config has no registered CKB CAIP-2 namespace, its local namespace is `ckb:` plus a 32-hex prefix of the configured genesis hash.

## Cursor, failures, and canonical changes

The native CKB page contains `{ height, cursor }`. The page relation compares heights; the cursor contains the observer's bounded block checkpoints, retained event IDs, and pending invalidations. The fetcher asks the observer to scan through one height at a time, so each native output carries the cursor for that block. EffectStream persists that marker with the root block transaction. On restart, the restored page resumes at the next height. A failed block parse returns no last page, so the same range is retried.

Before waiting for another height and before reading each block, the adapter checks committed checkpoint hashes. A changed checkpoint fails closed, even when the replaced block had no matching cell. The native runtime does not have a rollback operation for already committed root or state-machine effects. Recovery after a detected committed reorganization is therefore an explicit operator task; the adapter will not silently continue on the replacement history. The confirmation depth and bounded window are risk controls, not finality guarantees.

## Package boundary and reproduction

The upstream change is based on `5604043213df01beccebe253fbb479e387a514bb` in branch `feat/ckb-native-sync`. The original pinned checkout is kept in a separate clean worktree. The integration depends on the connector package through Bun's `file:` dependency; for this local implementation, the sibling `ckb-multichain-connector` checkout must be at connector commit `b8353e53298190dd937e90dd47d2f0623a2bf775`.

Expected sibling layout:

```text
GKM/
├── ckb-multichain-connector/
└── effectstream-ckb-sync/
```

From the EffectStream worktree:

```sh
bun install
bun test ./packages/effectstream-sdk/config/test
bun test ./packages/node-sdk/sync/src/sync-protocols/ckb/fetcher.test.ts
bun test ./packages/node-sdk/runtime/test/config-snapshot.unit.test.ts
cd packages/effectstream-sdk/config && bun run typecheck
cd ../../..
bun test ./packages
```

The repository has no root lint script or sync package typecheck script. The config package's typecheck script filters two known baseline diagnostics and fails on new diagnostics. To export a review patch from the local worktree, use:

```sh
git format-patch --stdout 5604043213df01beccebe253fbb479e387a514bb..feat/ckb-native-sync
```

The local `file:` dependency works only when both repositories are present as siblings. Before applying the upstream patch in an unrelated checkout, publish or push the exact connector revision and replace the local dependency with that immutable package revision. No push or publish is part of this milestone.

## Verification status and next work

Unit fixtures exercise schema registration and rejection, factory selection, observer-to-primitive mapping, empty block outputs, exact large capacities and distinct outpoints, cursor persistence/restoration, retry after parse failure, and fail-closed canonical replacement. No CKB node was used, so live RPC compatibility remains unverified. The EVM E2E runner also could not start in this environment because `forge` was missing from `PATH`.

Milestone 4 will add the separate CKB-to-EVM demonstration and destination contract. This change does not add signing, transaction submission, or a demo contract.
