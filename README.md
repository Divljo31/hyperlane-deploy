# Hyperlane on Hydration — mainnet deployment

The single runbook for deploying, hardening, and registering Hyperlane on **Hydration mainnet**,
with a WETH warp route to **Ethereum** and **Base**. Everything else in this repo is config that
the phases below consume. Ordering is strict: **0 → 1 → 2 → (3) → 4a → 4b → 4c → 4d → (4e/4f) → 5 → 6**.

> **Just deploying the contracts?** For a clean, self-contained hand-off runbook that a colleague can follow
> to deploy the core (messaging) contracts and smoke-test — **no warp route, no hardening** — see
> **[DEPLOY.md](DEPLOY.md)**. (Add the WETH warp route later via Phase 3.)
>
> **Adding Solana?** To extend this deployment with interchain *messaging* to/from Solana mainnet (no
> token/warp route), see **[SOLANA.md](SOLANA.md)** — it's additive on this same core deploy + agents.

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
| Explorer | `https://hydration.subscan.io` (Subscan, `family: other`) | human link works; API needs a key & isn't Etherscan-compatible → no auto contract-verification (see Phase 0 note) |
| Canonical registry | no `hydration` entry; chainId/domainId 222222 unused | checked 2026-07-02 |

Design decisions baked into the configs: `technicalStack: polkadotsubstrate` (as Moonbeam/Astar),
`maxFeePerGas: 15 gwei` (SDK overrides are a hard cap with no escalation; the runtime's dynamic fee
is clamped to ~14.4 gwei max, and with priority fee 0 only base fee is ever paid, so the headroom
is free), initial ISM `trustedRelayerIsm` (hardened in 4c), hooks `merkleTreeHook` +
`protocolFee(0)` — **no IGP is deployed**. Hydration runs a modern Frontier EVM (Cancun/Shanghai-class,
PUSH0/MCOPY available) and EVM deploys are **permissioned** — we hold the authorized deployer key
(Phase 0), so no OpenGov referendum is needed.

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

- **New keys** for 3 validators + relayer — never reuse testnet keys; agent keys live in
  `deploy/*/.env` (`chmod 600`, or Docker secrets/Vault); one independent key per validator.
- **Deployer key**: we use the existing whitelisted contract-deployer key (see Phase 0). It's the
  *initial owner* of the core contracts — Mailbox, ProxyAdmin (upgrade authority), fee hook, and the
  interchainAccountRouter (plus its ISM owners; see Phase 4d for the full list) — and, once Phase 3
  runs, of the warp routers and their proxyAdmins. It stays privileged until the **4d** (core) and
  **4f** (warp) Safe handovers, so **do 4d promptly and do NOT retire the key until 4e/4f are done**.
  Pass it only via ephemeral `HYP_KEY`, never committed. (If that key is shared/used elsewhere,
  coordinate to avoid nonce clashes during deploy.)
- **≥2 private RPCs per chain** for the agents, set in `deploy/*/.env` (`customRpcUrls` overrides
  the JSON at runtime — never commit credentialed URLs; the public URLs in metadata/agent-config
  are correct for the registry and as unused fallbacks).
- **N-of-M multisig ISM** (4c) and a **Safe** (not EOA) as final owner (4d). 4c hardens only the
  **core mailbox** default ISM — the warp routers keep their own `trustedRelayerIsm` until 4e.
- **Strip `trustedRelayerIsm` from every warp router (4e), then hand them to the Safe (4f)** — until
  4e, all inbound WETH deliveries (native release on Hydration, **mint** on Ethereum/Base) are
  verified by trusting the single relayer hot key with no validator signatures, so a compromised
  relayer key can forge deliveries and mint/release value. Do 4e/4f before the route carries value.
- Relayer DB is **fresh**; **one relayer per key** (warm standby only — two concurrent instances on
  one EOA race nonces and pay gas for already-delivered reverts).

## Phase 0 — prep

- [x] `chains/hydration/metadata.yaml` — complete & alphabetized; on-chain facts (chainId/domainId,
      RPCs, gas market, deployer `Hydration / hydration.net`) live-verified 2026-07-02. Explorer
      switched to Subscan and gas cap raised to 15 gwei on 2026-07-03 (Subscan apiUrl unverified —
      see the explorer item below).
- [x] `chains/hydration/logo.svg` — official mark, passes registry SVG validation
- [ ] Explorer: metadata points at Subscan (`family: other`). Human link works as-is. Only if you
      want EVM contract verification / tx-link tooling: either add a Subscan API key, or ask infra to
      expose the Blockscout API (`explorer.evm.hydration.cloud`) and add a `family: blockscout` entry.
      Not a blocker for the registry PR.
- [ ] Generate/secure relayer EOA + 3 validator EOAs
- [x] **Deployer authorization — SORTED.** Hydration mainnet EVM is permissioned
      (`EVMAccounts::can_deploy_contracts` gates every `create`), and we already hold the authorized
      contract-deployer key — no OpenGov referendum needed. This same key does Phase 1 core deploy
      AND Phase 3 warp deploys. Two caveats: (1) it becomes initial owner of all core contracts (and,
      after Phase 3, the warp routers) → prioritize the Phase 4d/4f Safe handovers and keep the key
      until 4e/4f are done (see security note above); (2) if that key is
      governance/SigNet-MPC-controlled rather than a raw private key, the CLI's `HYP_KEY` flow won't
      work directly — deploy through your existing signing tooling instead.
      *Alternative (skip): whitelist a fresh dedicated deployer EOA via `add_contract_deployer`
      (GeneralAdmin OpenGov referendum) for clean key isolation — only if you don't want to reuse
      the shared key.*
- [ ] Confirm the deployer key is usable as `HYP_KEY` (raw key) OR note the MPC-tooling path
- [ ] Fund the deployer EOA with ≥ 0.01 WETH on Hydration
- [ ] Fill `<DEPLOYER_EOA>` / `<RELAYER_EOA>` in `configs/hydration/core-config.initial.yaml`
      (and decide `protocolFee` — currently 0)

## Phase 1 — core deploy

```bash
npm i -g @hyperlane-xyz/cli   # CLI flags below verified vs v33.1.1; run `hyperlane <cmd> --help` to confirm on your version

# real directory, NOT a symlink — the CLI's registry walker doesn't follow symlinks
mkdir -p ~/.hyperlane/chains
cp -R chains/hydration ~/.hyperlane/chains/hydration

HISTFILE=/dev/null; read -rs HYP_KEY; export HYP_KEY     # deployer key, ephemeral shell
# --config is REQUIRED: core deploy reads its input from it (default ./configs/core-config.yaml, which we don't have)
hyperlane core deploy --chain hydration --config configs/hydration/core-config.initial.yaml
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
- [ ] **Do NOT run `hyperlane warp init` for this route** — it generates a fresh interactive config and
      discards the hand-pinned mailboxes/ISM/proxyAdmin. The current CLI resolves warp configs by
      `--warp-route-id` from the registry (no `--config` file flag), so stage the prepared file into
      the local registry and deploy by id:
      ```bash
      mkdir -p ~/.hyperlane/deployments/warp_routes/WETH
      cp configs/hydration/warp-WETH-hydration-ethereum-base.yaml \
         ~/.hyperlane/deployments/warp_routes/WETH/hydration-ethereum-base-deploy.yaml
      hyperlane warp deploy --warp-route-id WETH/hydration-ethereum-base
      ```
- [ ] Smoke it: `hyperlane warp send --warp-route-id WETH/hydration-ethereum-base --origin hydration
      --destination ethereum` (small amount), repeat for base, and back
- [ ] If direct ethereum↔base synthetic transfers should be relayed: extend the relayer
      `--whitelist` in `deploy/relayer/docker-compose.yml` with the warp router addresses

For other tokens (HOLLAR, HDX-as-ERC20, USDC…): `hyperlane warp init` **generates a new config from
scratch** → stage it under a new `warp-route-id` → `warp deploy`, same pattern.

## Phase 4 — production hardening

### 4a — validators

`deploy/validator/docker-compose.yml`: 3 validators, hexKey signing, localStorage checkpoints on the
shared `hyperlane_checkpoints` volume. Running all three on one host co-locates all keys, so one host
compromise yields the full 2-of-3 quorum — the attacker can sign fraudulent checkpoints and mint/release
value (an integrity/fund-theft risk, not just downtime). Split across independent hosts before the route
carries real value (back the shared volume with NFS).

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
`configs/hydration/core-config.multisig-ism.yaml`, then follow its header workflow — critically, pass
`--config /tmp/hydration-core.yaml` on **both** `core read` and `core apply`. A bare `core apply`
reads the default `./configs/core-config.yaml`, not your edit, so it silently applies nothing (treat
`"No updates needed"` as a red flag). Keep `threshold ≤ M-1` (fewer than total validators) so one can be down.

### 4d — transfer ownership to a Safe

Deploy/pick a Safe on Hydration EVM (no Safe Transaction Service there — coordinate signers
off-chain). Use `configs/hydration/core-config.safe-owner.yaml` — it lists **every** owner field
including `interchainAccountRouter` and its ISM owners (and the `requiredHook.beneficiary`); missing
one leaves the deployer EOA privileged. Same `--config /tmp/hydration-core.yaml`-on-read-and-apply
rule as 4c (a bare `core apply` no-ops). This is the deployer's last **core** action — but the warp
route stays EOA-owned until 4f, so **do not retire the key until 4e/4f are done**.

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

Gate: working deployment + smoke test. (Explorer is set to Subscan; no apiUrl blocker.)

```bash
DEPLOY_REPO="$(pwd)"   # run from the hyperlane-deploy repo root
gh repo fork hyperlane-xyz/hyperlane-registry --clone -- ../hyperlane-registry
cd ../hyperlane-registry
mkdir -p chains/hydration
cp "$DEPLOY_REPO"/chains/hydration/{metadata.yaml,addresses.yaml,logo.svg} chains/hydration/
yarn install            # the changeset binary isn't present until deps are installed
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
| `max fee per gas less than block base fee` | Base fee exceeded the 15 gwei cap in metadata `transactionOverrides` (runtime max ≈14.4 gwei, so this should not happen — check the runtime's fee params changed) |
| Deploy rejected at gas estimation (before any tx is sent; a `pallet-evm-accounts` error such as `AddressNotWhitelisted` — confirm the exact string against a rejected deploy) | `HYP_KEY` isn't the whitelisted deployer key (wrong address, or not on the `ContractDeployer` allowlist — add via `add_contract_deployer`) |
| Relayer ignores checkpoints | Not sharing the validators' volume, or `--allowLocalCheckpointSyncers` not `true` |
| YAML schema validation error | `hyperlane registry list -r .` from repo root surfaces the failing field |
