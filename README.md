# hyperlane-deploy

Hyperlane core deployment for **Lark** — Hydration's testnet parachain (chain ID `222222`, Frontier EVM). This deployment is a dress rehearsal before deploying the same setup to Hydration mainnet.

This repo holds:

- `chains/lark/` — Hyperlane registry entry (copy into `~/.hyperlane/chains/` in step 2)
- `configs/core-config.yaml` — owner / ISM / hook config consumed by `hyperlane core deploy`

For the full production roadmap (validators, relayer, ISM hardening, registry PR) see [PRODUCTION.md](./PRODUCTION.md).

---

## Status

| | |
|---|---|
| `chains/lark/metadata.yaml` | done |
| `configs/core-config.yaml` | done — edit `owner` fields in step 3 if you're not the original deployer |
| `chains/lark/addresses.yaml` | written by `hyperlane core deploy` in step 6 |
| Smoke test | step 7 — Lark ↔ Ethereum round trip |
| Canonical registry PR | see PRODUCTION.md phase 5 |

---

## Metadata + config decisions baked in

| Field | Value | Why |
|---|---|---|
| `nativeToken` | `WETH / 18` | Lark's `pallet_evm::Config::Currency = WethCurrency`. `eth_getBalance` and `msg.value` are denominated in WETH, not HDX. |
| `technicalStack` | `polkadotsubstrate` | Same flag Moonbeam / Astar / Acala use for Frontier-based chains. |
| `blocks.reorgPeriod` | `finalized` | Use the relay-chain finality tag, not a block count. |
| `blockExplorers` | omitted | Subscan is Cloudflare-gated and not Etherscan-compatible; required `apiUrl` field can't be filled. Block explorer can be added post-deploy. |
| Default ISM | `trustedRelayerIsm` | Simplest starting point. Upgrade to `messageIdMultisigIsm` after validators are running — see PRODUCTION.md phase 4c. |
| Default hook | `merkleTreeHook` | Required for messages to be provable on the destination. |
| Required hook | `protocolFee` with `protocolFee = 0` | Zero fee for now. |

---

## Prerequisites

- **Node.js ≥ 18** and the Hyperlane CLI: `npm i -g @hyperlane-xyz/cli`
- **Deployer EOA private key** (you'll paste it as `HYP_KEY` in step 5)
- **≥ 0.01 WETH** on the deployer EOA at the Lark EVM. Source WETH from the testnet faucet / Lark sudo or transfer from a Lark account holding asset id `20`.

---

## Deploy

### 1. Clone the repo

```bash
git clone <repo-url> hyperlane-deploy
cd hyperlane-deploy
```

### 2. Wire the CLI to this repo's chain config

Copy the chain folder into `~/.hyperlane/chains/` as a real directory. The CLI's filesystem-registry walker doesn't follow symlinks at the chain-folder level, so a copy is required.

```bash
mkdir -p ~/.hyperlane/chains
cp -R chains/lark ~/.hyperlane/chains/lark
```

Verify the chain is now discoverable:

```bash
hyperlane registry list | grep lark   # should print: lark | 'Lark' | 222222
```

### 3. Set the owner address in `core-config.yaml`

Open `configs/core-config.yaml`. Replace every occurrence of the owner address with **your** deployer EVM address (the address that corresponds to the private key you'll use in step 5).

```yaml
defaultIsm:
  relayer: "0xYourDeployerAddress"     # who can deliver inbound messages
  type: trustedRelayerIsm
owner: "0xYourDeployerAddress"          # Mailbox owner — can change ISM, hooks
proxyAdmin:
  owner: "0xYourDeployerAddress"        # most powerful: can upgrade Mailbox impl
requiredHook:
  beneficiary: "0xYourDeployerAddress"  # protocol fee recipient (fee is 0 currently)
  maxProtocolFee: "100000000000000000"
  owner: "0xYourDeployerAddress"        # owner of the protocolFee hook
  protocolFee: "0"
  type: protocolFee
```

Use the same address in all five places for the initial deploy. PRODUCTION.md phase 4 covers the post-deploy handoff to a multisig and a dedicated relayer key.

### 4. Verify gas balance

```bash
DEPLOYER=0xYourDeployerAddress
curl -s https://node3.lark.hydration.cloud -H 'content-type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"method\":\"eth_getBalance\",\"params\":[\"$DEPLOYER\",\"latest\"],\"id\":1}" \
  | python3 -c 'import sys,json; r=json.load(sys.stdin); print(int(r["result"],16)/1e18, "WETH")'
```

Expect ≥ 0.01 WETH. If lower, top up before continuing.

### 5. Export the deployer key

Use an ephemeral shell so the key doesn't end up in history:

```bash
HISTFILE=/dev/null
read -rs HYP_KEY        # paste private key, press enter (input is hidden)
export HYP_KEY
```

### 6. Deploy core contracts

```bash
hyperlane core deploy --chain lark
```

Takes 2-5 minutes. Deploys ~12 contracts in sequence:

- Mailbox + proxy
- `staticMerkleRootMultisigIsmFactory`, `staticMessageIdMultisigIsmFactory`, `staticAggregationIsmFactory`
- `staticAggregationHookFactory`, `domainRoutingIsmFactory`, `proxyAdmin`
- `interchainAccountRouter`, `interchainAccountIsm`
- `validatorAnnounce`, `testRecipient`

On success, addresses land in `~/.hyperlane/chains/lark/addresses.yaml`. Copy them back into the repo before committing:

```bash
cp ~/.hyperlane/chains/lark/addresses.yaml chains/lark/addresses.yaml
```

Inspect:

```bash
hyperlane core read --chain lark
cat chains/lark/addresses.yaml
```

Owner, ISM, hooks, and beneficiary should match what you set in step 3.

### 7. Smoke test (Lark → Lark loopback)

Hyperlane's canonical relayer infrastructure doesn't know about Lark, so we can't send a message to Ethereum or any other public chain from a Lark deploy without running our own agents. The simplest smoke test is a self-addressed message — Lark dispatches, Lark's own testRecipient receives, ephemeral relayer delivers locally.

```bash
hyperlane send message --origin lark --destination lark --relay
```

Expect: dispatched tx hash + delivered tx hash, both on Lark, plus a message ID.

Check status separately if needed:

```bash
hyperlane status --origin lark --destination lark --id <messageId>
```

For a full inter-chain test (Lark ↔ another chain with continuous delivery) you'd run local validator + relayer agents — see the [deploy-hyperlane-with-local-agents](https://docs.hyperlane.xyz/docs/guides/chains/deploy-hyperlane-with-local-agents) guide.

### 8. Commit deploy artifacts

```bash
git add chains/lark/addresses.yaml configs/core-config.yaml
git commit -m "Deploy Hyperlane core on Lark testnet"
git push
```

---

## Going to production

The Lark deploy above is intentionally insecure: single EOA owner, single trusted relayer. The path from there to a hardened mainnet (Hydration) bridge is ten ordered steps. Full detail in [PRODUCTION.md](./PRODUCTION.md); summary below.

**Critical ordering:** steps 4 → 5 → 6 → 7 must run in that order. You can't switch to a multisig ISM before validators exist, and you don't want to hand ownership to a Safe until you've confirmed `core apply` flows work with the deployer EOA — once the Safe owns the Mailbox, every config change becomes a multi-sig ceremony.

### Step 0 — Prep mainnet inputs

- Create `chains/hydration/` as a mirror of [chains/lark/](./chains/lark/). Adjust `chainId`, `domainId` (must be globally unique — testnet reuses 222222, mainnet needs its own), and RPC URLs.
- Fund the deployer EOA with ≥ 0.01 WETH on Hydration EVM.
- Copy [configs/core-config.yaml](./configs/core-config.yaml). Keep `trustedRelayerIsm` for the initial deploy — you swap it in step 6.

### Step 1 — Core deploy on Hydration

Same flow as Lark (steps 1–6 above):

```bash
cp -R chains/hydration ~/.hyperlane/chains/hydration
export HYP_KEY=...
hyperlane core deploy --chain hydration
cp ~/.hyperlane/chains/hydration/addresses.yaml chains/hydration/addresses.yaml
```

Outcome: Mailbox + factories + ICA router + ValidatorAnnounce on Hydration mainnet, owned by the deployer EOA, with `trustedRelayerIsm` as default.

### Step 2 — Smoke test

```bash
hyperlane send message --origin hydration --destination ethereum --relay
```

`--relay` runs an ephemeral relayer using `HYP_KEY`. Confirms the deploy works end-to-end before real assets are bridged.

### Step 3 — Warp route(s) deploy (optional)

Only if you're bridging a specific token (HOLLAR, HDX-as-ERC20, USDC, …). Same pattern as the existing [configs/warp-routes/WETH-lark-basesepolia.yaml](./configs/warp-routes/WETH-lark-basesepolia.yaml):

```bash
hyperlane warp init       # picks native/collateral/synthetic per chain
hyperlane warp deploy
hyperlane warp send       # small test transfer
```

### Step 4 — Run your own validators

Validators sign Merkle-root checkpoints of the Hydration Mailbox so other chains can verify Hydration-origin messages.

1. Provision a validator signing key — **AWS KMS** (not hex) for production.
2. Provision an S3 bucket for checkpoint storage.
3. Provision a **dedicated private RPC** to Hydration. Never public RPCs (rate limits + finality risk).
4. Launch the validator agent in Docker, pointed at KMS + S3 + private RPC.
5. Fund the validator EOA once for the `ValidatorAnnounce.announce()` tx.
6. **Repeat on independent infrastructure** for N validators — separate keys, hosts, RPC providers. This is the whole point of multisig: an attacker has to compromise multiple operators.

Wait until checkpoints are appearing in S3 before moving on.

### Step 5 — Run your own relayer

The local [hyperlane_db_relayer/](./hyperlane_db_relayer/) state is from a dev relayer. For production:

1. Provision a relayer signing key (separate from any validator key) — AWS KMS.
2. Fund the relayer EOA on **every destination chain** it will deliver to.
3. Launch the relayer agent in Docker with `--relayChains hydration,ethereum,base,...` and **private RPCs per chain**.
4. Run a hot-spare on independent infrastructure — the Mailbox is idempotent, so duplicate delivery is safe.

### Step 6 — Switch ISM trustedRelayer → messageIdMultisigIsm

**Only after step 4 validators are producing checkpoints.** This is the actual security upgrade — until now anyone with the relayer key could forge messages.

Edit [configs/core-config.yaml](./configs/core-config.yaml):

```yaml
defaultIsm:
  type: messageIdMultisigIsm
  threshold: 2
  validators:
    - "0x<validator-1>"
    - "0x<validator-2>"
    - "0x<validator-3>"
```

Apply and verify:

```bash
hyperlane core apply --chain hydration
hyperlane core read --chain hydration
```

Pick `threshold` so one validator outage can't halt the bridge (e.g., 2-of-3, 3-of-5).

### Step 7 — Transfer ownership to a Gnosis Safe

While the deployer EOA still owns Mailbox / ProxyAdmin / hooks, a single key compromise = full bridge takeover.

1. Deploy or pick a Safe on Hydration EVM. Safe Transaction Service isn't live there, so coordinate signatures off-chain.
2. In [configs/core-config.yaml](./configs/core-config.yaml), change every `owner:` field (top-level, `proxyAdmin.owner`, `requiredHook.owner`) to the Safe address.
3. `hyperlane core apply --chain hydration`.
4. Verify with `hyperlane core read --chain hydration`. From this point, ISM / hook changes require Safe approvals.

### Step 8 — Same hardening for warp routes

If you ran step 3:

- `hyperlane warp read` → strip `trustedRelayerIsm` from warp config → `hyperlane warp apply`.
- Mirror step 7's ownership transfer for each warp route's `owner` and `proxyAdmin.owner`.

### Step 9 — Canonical registry PR

Only after the bridge has been running stably end-to-end.

- Add `deployer:` + `gasCurrencyCoinGeckoId: weth` + `logo.svg` to `chains/hydration/metadata.yaml`.
- Fork `hyperlane-xyz/hyperlane-registry`, copy `metadata.yaml` + `addresses.yaml` + `logo.svg` into `chains/hydration/`, run `yarn changeset add`, open PR.
- After merge, Hydration appears in the canonical registry — other deployers can route to/from it without local overrides.

### Step 10 — Ongoing operations

Track these continuously:

| Signal | What it tells you |
|---|---|
| Validator S3 checkpoint last-modified | Validator alive + producing |
| Relayer delivery latency (dispatch → process) | Relayer / RPC health |
| `Mailbox.dispatched - processed` per chain | Backlog forming |
| Validator + relayer EOA gas balances per chain | Days until they go dark |
| RPC error rate | Need to add or swap providers |

Scrape the agents' Prometheus endpoints into your existing monitoring stack.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `No chain metadata set for lark` | Step 2 wasn't run, or `~/.hyperlane/chains/lark/` is a symlink instead of a real directory. The CLI doesn't follow symlinks at that level. Re-run step 2 with `cp -R`. |
| `insufficient funds for gas` | Top up the deployer EOA with more WETH on Lark. |
| Deploy hangs at a contract | RPC flake. CLI rotates through `rpcUrls` in order — reorder them in `chains/lark/metadata.yaml` if a specific endpoint is unstable. |
| Smoke test message stays pending | Ephemeral relayer needs gas on **both** chains. Confirm the deployer holds ETH on Ethereum, not just WETH on Lark. |
| YAML schema validation error | A required field is missing in `metadata.yaml`. Run `hyperlane registry list -r .` from the repo root to surface the failing field path. |
