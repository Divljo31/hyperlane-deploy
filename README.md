# Hyperlane on Hydration — mainnet deployment

The single runbook for deploying, hardening, and registering Hyperlane on **Hydration mainnet**,
with a WETH warp route to **Ethereum** and **Base**. Everything else in this repo is config that
the phases below consume. Ordering is strict: **0 → 1 → 2 → (3) → 4a → 4b → 4c → 4d → (4e/4f) → 5 → 6**.

> History: this setup was dress-rehearsed on the Lark testnet, which was removed on 2026-07-02
> because it had reused mainnet's chainId/domainId `222222` (recover via `git log -- chains/lark`).
> Any future testnet must use a **different domainId** (e.g. `1222222`).

## Chain facts (live-verified 2026-07-02)

| Fact | Value | Verification |
|---|---|---|
| chainId = domainId | `222222` | `eth_chainId` = `0x3640e` on both RPCs below |
| RPCs (public, for registry metadata) | `https://rpc.hydradx.cloud`, `https://hydration-rpc.n.dwellir.com` | both answered `eth_chainId` |
| Native/gas token | WETH, 18 decimals | Frontier `pallet_evm` WethCurrency |
| Finality | `reorgPeriod: finalized` (relay-chain tag), ~6s blocks | — |
| Gas market | base fee ≈ 0.0041 gwei, priority fee 0 | `eth_gasPrice` / `eth_maxPriorityFeePerGas` |
| Explorer | `https://explorer.evm.hydration.cloud` (Blockscout) | ⚠️ backend `apiUrl` unconfirmed — `/api` 404s; ask Hydration infra before the Phase 5 PR |
| Canonical registry | no `hydration` entry; chainId/domainId 222222 unused | checked 2026-07-02 |

Design decisions baked into the configs: `technicalStack: polkadotsubstrate` (as Moonbeam/Astar),
`maxFeePerGas: 3 gwei` (SDK overrides are a hard cap with no escalation — with priority fee 0 only
base fee is ever paid, so headroom is free), initial ISM `trustedRelayerIsm` (hardened in 4c),
hooks `merkleTreeHook` + `protocolFee(0)` — **no IGP is deployed**.

## Repo layout

```
chains/hydration/            registry entry: metadata.yaml + logo.svg (+ addresses.yaml after Phase 1)
configs/
  agent-config.mainnet.json  agent chain config (zero-addresses filled after Phase 1)
  hydration/
    core-config.initial.yaml       Phase 1 input (EOA owner + trustedRelayerIsm)
    core-config.multisig-ism.yaml  Phase 4c ISM swap
    core-config.safe-owner.yaml    Phase 4d Safe handover (includes ICA-router owners)
    warp-WETH-hydration-ethereum-base.yaml  Phase 3 warp route (3-chain)
deploy/
  validator/   docker-compose + .env.example — 3 validators (hexKey + localStorage)
  relayer/     docker-compose + .env.example — relayer (whitelisted to Hydration traffic)
  monitoring/  prometheus.yml + alerts.yml
```

## Security must-dos (blocking)

- **New keys** for deployer / 3 validators / relayer — never reuse testnet keys. Deployer key only
  via ephemeral `HYP_KEY`. Agent keys live in `deploy/*/.env` (`chmod 600`, or Docker secrets/Vault).
  One independent key per validator.
- **≥2 private RPCs per chain** for the agents, set in `deploy/*/.env` (`customRpcUrls` overrides
  the JSON at runtime — never commit credentialed URLs; the public URLs in metadata/agent-config
  are correct for the registry and as unused fallbacks).
- **N-of-M multisig ISM** (4c) and a **Safe** (not EOA) as final owner (4d).
- Relayer DB is **fresh**; **one relayer per key** (warm standby only — two concurrent instances on
  one EOA race nonces and pay gas for already-delivered reverts).

## Phase 0 — prep

- [x] `chains/hydration/metadata.yaml` — complete & live-verified (chainId/domainId, RPCs, explorer,
      deployer `Hydration / hydration.net`, gas overrides, alphabetized) — 2026-07-02
- [x] `chains/hydration/logo.svg` — official mark, passes registry SVG validation
- [ ] Confirm explorer `apiUrl` (only remaining metadata TODO; gates Phase 5)
- [ ] Generate/secure deployer EOA, relayer EOA, 3 validator EOAs
- [ ] Fund deployer EOA with ≥ 0.01 WETH on Hydration
- [ ] Fill `<DEPLOYER_EOA>` / `<RELAYER_EOA>` in `configs/hydration/core-config.initial.yaml`
      (and decide `protocolFee` — currently 0)

## Phase 1 — core deploy

```bash
npm i -g @hyperlane-xyz/cli

# real directory, NOT a symlink — the CLI's registry walker doesn't follow symlinks
mkdir -p ~/.hyperlane/chains
cp -R chains/hydration ~/.hyperlane/chains/hydration

HISTFILE=/dev/null; read -rs HYP_KEY; export HYP_KEY     # deployer key, ephemeral shell
hyperlane core deploy --chain hydration
cp ~/.hyperlane/chains/hydration/addresses.yaml chains/hydration/addresses.yaml
```

Output: Mailbox, ISM/hook factories, ICA router, ValidatorAnnounce, testRecipient — EOA-owned,
`trustedRelayerIsm` (intentionally insecure until Phase 4).

Then backfill `configs/agent-config.mainnet.json`:

- [ ] `mailbox`, `merkleTreeHook`, `validatorAnnounce` ← `chains/hydration/addresses.yaml`
- [ ] `index.from` ← the Mailbox deploy block number
- [ ] `interchainGasPaymaster` **stays `0x0`** — the initial core config deploys no IGP
- [ ] Commit `addresses.yaml` (hard requirement for the Phase 5 registry PR)

## Phase 2 — smoke test (gates the registry PR)

```bash
hyperlane send message --origin hydration --destination ethereum --relay
```

`--relay` runs an ephemeral relayer with `HYP_KEY`; the deployer needs a little ETH on Ethereum for
delivery. Verify: tx hashes printed, message delivered, destination `Mailbox.processed(messageId)` true.

## Phase 3 — warp route (WETH → Ethereum + Base)

Config is ready at `configs/hydration/warp-WETH-hydration-ethereum-base.yaml` — one 3-chain route:
`native` on Hydration (WETH is the gas token), `synthetic` mint/burn on Ethereum and Base, canonical
remote mailboxes already filled.

> Product note: `synthetic` means users receive a route-minted ERC20, **not** canonical WETH, on the
> remotes. The alternative (`collateral` pointing at canonical WETH) gives users the real token but
> needs seeded vault liquidity + rebalancing. Decide before deploying.

- [ ] Fill `<HYDRATION_MAILBOX>` (from Phase 1) + owner/relayer EOAs in the warp config
- [ ] `hyperlane warp deploy`, then `hyperlane warp send` a small amount hydration→ethereum,
      hydration→base, and back
- [ ] If direct ethereum↔base synthetic transfers should be relayed: extend the relayer
      `--whitelist` in `deploy/relayer/docker-compose.yml` with the warp router addresses

For other tokens (HOLLAR, HDX-as-ERC20, USDC…): `hyperlane warp init` → `warp deploy`, same pattern.

## Phase 4 — production hardening

### 4a — validators

`deploy/validator/docker-compose.yml`: 3 validators, hexKey signing, localStorage checkpoints on the
shared `hyperlane_checkpoints` volume. Running all three on one host is an MVP — a host compromise
exposes all keys; split hosts for real HA (back the volume with NFS).

```bash
docker volume create hyperlane_checkpoints
cd deploy/validator && cp .env.example .env && $EDITOR .env && chmod 600 .env
docker compose up -d
```

- [ ] Each validator EOA holds a little WETH (one-time `ValidatorAnnounce.announce()` tx)
- [ ] Checkpoint files appear under `/checkpoints/validator-{1,2,3}` on the shared volume
- [ ] Record each validator's EVM address (needed in 4c)

### 4b — relayer

`deploy/relayer/docker-compose.yml`: hexKey, `--allowLocalCheckpointSyncers=true` (**required** with
localStorage validators; same host or shared NFS volume), and a `--whitelist` restricting relaying to
Hydration traffic — without it the relayer indexes the canonical ethereum/base mailboxes and pays gas
delivering third-party traffic until the EOA drains. Metrics on host port **9094** (9090 is reserved
for Prometheus).

```bash
cd deploy/relayer && cp .env.example .env && $EDITOR .env && chmod 600 .env
docker compose up -d    # relayer EOA must be funded on hydration, ethereum, AND base
```

### 4c — switch default ISM to 2-of-3 multisig

Only after 4a validators produce checkpoints. Fill `<VALIDATOR_N_EVM_ADDR>` in
`configs/hydration/core-config.multisig-ism.yaml`, then follow its header workflow
(`core read` → paste `defaultIsm` block → `core apply` → `core read` to confirm).
Keep `threshold ≤ M-1` (fewer than total validators) so one can be down.

### 4d — transfer ownership to a Safe

Deploy/pick a Safe on Hydration EVM (no Safe Transaction Service there — coordinate signers
off-chain). Use `configs/hydration/core-config.safe-owner.yaml` — it lists **every** owner field
including `interchainAccountRouter` and its ISM owners; missing one leaves the deployer EOA
privileged. `core apply` is the deployer's last signed action.

### 4e — strip trustedRelayerIsm from the warp route

> **Remote chains first.** Stripping makes each router fall back to its chain's Mailbox default ISM.
> On hydration that's your 4c multisig — fine. On ethereum/base it's the **canonical** default ISM,
> which has no route for domain 222222: set an explicit `messageIdMultisigIsm` (your validators,
> origin hydration) on those routers **before** removing the trusted relayer, or all
> hydration→remote transfers brick.

`hyperlane warp read` → edit → `hyperlane warp apply`.

### 4f — transfer warp route ownership to the Safe

Mirror 4d for the warp routers, **including the hydration router's `proxyAdmin`** (pinned in the
warp config precisely so it's part of this handover).

## Phase 5 — canonical registry PR (the "supported chains" listing)

Gate: working deployment + smoke test, and the explorer `apiUrl` TODO resolved.

```bash
gh repo fork hyperlane-xyz/hyperlane-registry --clone
cd hyperlane-registry
mkdir -p chains/hydration
cp ../hyperlane-deploy/chains/hydration/{metadata.yaml,addresses.yaml,logo.svg} chains/hydration/
yarn changeset add
git checkout -b add-hydration
git add . && git commit -m "Add hydration chain" && git push -u origin add-hydration
gh pr create
```

Merge = listed. The docs' supported-chains list is generated from this registry.

## Phase 6 — ongoing operations & monitoring

Stand up monitoring **as soon as agents run** (4a/4b): `deploy/monitoring/` scrapes host ports
9091-9093 (validators) and 9094 (relayer); run Prometheus on the same host (binary or
`network_mode: host`). Alerts cover agent-down, low gas, checkpoint stalls (traffic-guarded),
submit-queue backlog, and sync stalls — **wire an Alertmanager or nothing pages**.

| Concern | Action |
|---|---|
| Validator key rotation | Rotate key, update the multisig ISM via `core apply` (Safe ceremony after 4d) |
| Validator host failure | Threshold ≤ M-1 tolerates one down; fix or replace, re-announce |
| Relayer host failure | Start the warm standby (same DB volume) — never two live relayers on one key |
| RPC failure | Multiple `customRpcUrls` per chain in `.env`; reorder if one is flaky |
| Gas top-ups | Alerts fire below 0.05 (relayer) / 0.02 (validator); keep N days of burn |
| Message stuck | Destination `Mailbox.processed(id)` false → relayer logs (usually gas or RPC) |
| New warp route | Phase 3 again for the new token |
| ISM/hook change | `core apply` — owner is the Safe after 4d |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `No chain metadata set for hydration` | `~/.hyperlane/chains/hydration` missing or a symlink — re-run the Phase 1 `cp -R` |
| `insufficient funds for gas` | Deployer needs WETH on Hydration (and ETH on the destination for `--relay`) |
| Deploy hangs at a contract | RPC flake — CLI rotates through `rpcUrls` in order; reorder in metadata.yaml |
| Smoke message stays pending | Ephemeral relayer needs gas on **both** chains |
| `max fee per gas less than block base fee` | Base fee spiked past the 3 gwei cap in metadata `transactionOverrides` — raise it |
| Relayer ignores checkpoints | Not sharing the validators' volume, or `--allowLocalCheckpointSyncers` not `true` |
| YAML schema validation error | `hyperlane registry list -r .` from repo root surfaces the failing field |
