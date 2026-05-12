# hyperlane-deploy

Hyperlane core deployment for **Hydration** — chain ID `222222`, Frontier EVM on a Polkadot parachain.

This repo holds:

- `chains/hydration/` — Hyperlane registry entry (copy into `~/.hyperlane/chains/` in step 2)
- `configs/core-config.yaml` — owner / ISM / hook config consumed by `hyperlane core deploy`

For the full production roadmap (validators, relayer, ISM hardening, registry PR) see [PRODUCTION.md](./PRODUCTION.md).

---

## Status

| | |
|---|---|
| `chains/hydration/metadata.yaml` | done |
| `configs/core-config.yaml` | done — edit `owner` fields in step 3 if you're not the original deployer |
| `chains/hydration/addresses.yaml` | written by `hyperlane core deploy` in step 6 |
| Smoke test | step 7 — Hydration ↔ Ethereum round trip |
| Canonical registry PR | see PRODUCTION.md phase 5 |

---

## Metadata + config decisions baked in

| Field | Value | Why |
|---|---|---|
| `nativeToken` | `WETH / 18` | Hydration's `pallet_evm::Config::Currency = WethCurrency`. `eth_getBalance` and `msg.value` are denominated in WETH, not HDX. |
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
- **≥ 0.01 WETH** on the deployer EOA at the Hydration EVM. Bridge from Ethereum via Snowbridge or transfer from a Hydration account holding asset id `20`.
- A small amount of **ETH on Ethereum mainnet** for the smoke test (the ephemeral relayer pays delivery gas there)

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
cp -R chains/hydration ~/.hyperlane/chains/hydration
```

Verify the chain is now discoverable:

```bash
hyperlane registry list | grep hydration   # should print: hydration | 'Hydration' | 222222
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
curl -s https://hydration.dotters.network -H 'content-type: application/json' \
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
hyperlane core deploy --chain hydration
```

Takes 2-5 minutes. Deploys ~12 contracts in sequence:

- Mailbox + proxy
- `staticMerkleRootMultisigIsmFactory`, `staticMessageIdMultisigIsmFactory`, `staticAggregationIsmFactory`
- `staticAggregationHookFactory`, `domainRoutingIsmFactory`, `proxyAdmin`
- `interchainAccountRouter`, `interchainAccountIsm`
- `validatorAnnounce`, `testRecipient`

On success, addresses land in `~/.hyperlane/chains/hydration/addresses.yaml`. Copy them back into the repo before committing:

```bash
cp ~/.hyperlane/chains/hydration/addresses.yaml chains/hydration/addresses.yaml
```

Inspect:

```bash
hyperlane core read --chain hydration
cat chains/hydration/addresses.yaml
```

Owner, ISM, hooks, and beneficiary should match what you set in step 3.

### 7. Smoke test

Send a message Hydration → Ethereum with an ephemeral relayer:

```bash
hyperlane send message --origin hydration --destination ethereum --relay
```

Expect: dispatched tx hash on Hydration, delivered tx hash on Ethereum, message ID printed.

Check status separately if needed:

```bash
hyperlane status --origin hydration --destination ethereum --id <messageId>
```

### 8. Commit deploy artifacts

```bash
git add chains/hydration/addresses.yaml configs/core-config.yaml
git commit -m "Deploy Hyperlane core on Hydration mainnet"
git push
```

---

## After a successful deploy

Phase 1 + 2 are done. To make the bridge production-grade, continue with [PRODUCTION.md](./PRODUCTION.md):

- Run your own validators (4a)
- Run your own relayer (4b)
- Switch the default ISM from trusted-relayer to multisig (4c)
- Transfer ownership to a Gnosis Safe (4d)
- Submit Hydration to the canonical hyperlane-registry (phase 5)

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `No chain metadata set for hydration` | Step 2 wasn't run, or `~/.hyperlane/chains/hydration/` is a symlink instead of a real directory. The CLI doesn't follow symlinks at that level. Re-run step 2 with `cp -R`. |
| `insufficient funds for gas` | Top up the deployer EOA with more WETH on Hydration. |
| Deploy hangs at a contract | RPC flake. CLI rotates through `rpcUrls` in order — reorder them in `chains/hydration/metadata.yaml` if a specific endpoint is unstable. |
| Smoke test message stays pending | Ephemeral relayer needs gas on **both** chains. Confirm the deployer holds ETH on Ethereum, not just WETH on Hydration. |
| YAML schema validation error | A required field is missing in `metadata.yaml`. Run `hyperlane registry list -r .` from the repo root to surface the failing field path. |
