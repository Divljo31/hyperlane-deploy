# Hyperlane contract deployment — Hydration mainnet (hand-off runbook)

**What you're doing:** deploying the on-chain Hyperlane contracts for **interchain messaging** and verifying
they work, then handing the addresses back. Concretely:
1. **Core** on Hydration (Mailbox, ISM, hooks, ICA router, ValidatorAnnounce, factories) — this is what
   enables messaging.
2. **Solana** — *no contract to deploy* (canonical), but you'll smoke-test the messaging lane.

> **Messaging only — no token/warp route.** The warp-route config stays in the repo for later; add the token
> bridge via [README.md](README.md) Phase 3 when you want it. Nothing here depends on it.

**In scope:** deploy + smoke test. **Out of scope (someone's follow-on job):** running validators/relayer,
switching to the multisig ISM, the Safe ownership handover, the registry PR, monitoring, and the Solana
trust-minimised ISM. Those are in [README.md](README.md) §4–6 and [SOLANA.md](SOLANA.md).

> ⚠️ **Read first.** These contracts deploy **insecure by design**: owned by the deployer EOA and using
> `trustedRelayerIsm` (delivery is trusted, not validator-verified). **Do not carry real traffic you can't
> tolerate a trusted relayer seeing/forging** until the hardening follow-on is done. **Do not lose or retire
> the deployer key** — the hardening steps need it. Every command here is signed by the deployer key.
> Commands were verified against **Hyperlane CLI v33.1.1**; run `hyperlane <cmd> --help` if your version differs.

---

## Step 0 — Prerequisites (gather these before you touch anything)

**Keys**
- [ ] The **whitelisted deployer key** (Hydration EVM is permissioned — this key is already authorised to
      deploy). You'll use it as `HYP_KEY`. If it's MPC/SigNet-controlled rather than a raw hex key, the
      `HYP_KEY` flow won't work — deploy through your signing tooling instead and stop here to coordinate.
- [ ] *(Optional, for the Solana smoke)* a Solana keypair: `solana-keygen new --outfile solana.json`.

**Funds** (all on the **deployer** address)

| Chain | Fund with | Covers |
|---|---|---|
| Hydration | **≥ 0.01 WETH** | core deploy + gas |
| Ethereum | **small ETH** (~0.02) | smoke-message delivery |
| Base | **small ETH** (~0.01) *(optional)* | only if you also smoke Hydration↔Base messaging |
| Solana | **~1 SOL** *(optional)* | cross-VM message smoke (fees + account rent) |

**RPCs** — a working RPC per chain. Hydration public RPCs are fine for a one-off deploy
(`https://rpc.hydradx.cloud`); for **Solana** use a private/paid RPC (Helius/Triton) — `api.mainnet-beta`
rate-limits and will stall.

**Tooling** — Node.js LTS, `git`. Install the CLI in Step 1.

**Fill the config placeholders** (for a deploy + smoke done by one person, set the owner/relayer fields to the
**deployer address** — the real relayer/Safe are set during hardening, which replaces these ISMs anyway):
- [ ] [`configs/hydration/core-config.initial.yaml`](configs/hydration/core-config.initial.yaml): set
      `<DEPLOYER_EOA>` and `<RELAYER_EOA>` **both to the deployer address**. Leave `protocolFee: 0`.

---

## Step 1 — Install the CLI and stage the chain metadata

```bash
npm i -g @hyperlane-xyz/cli          # verify flags with `hyperlane <cmd> --help` if not on v33.1.1

# Copy Hydration's registry entry into a REAL dir (not a symlink — the CLI won't follow symlinks)
mkdir -p ~/.hyperlane/chains
cp -R chains/hydration ~/.hyperlane/chains/hydration
```

**Verify:** `hyperlane registry list -r ~/.hyperlane` shows `hydration`. (Ethereum, Base, and `solanamainnet`
are already in the CLI's default registry — nothing to add.)

---

## Step 2 — Deploy the core contracts on Hydration

```bash
# Load the deployer key into this shell only (not saved to history)
HISTFILE=/dev/null; read -rs HYP_KEY; export HYP_KEY

# --config is REQUIRED (the CLI's default path isn't this repo's config)
hyperlane core deploy --chain hydration --config configs/hydration/core-config.initial.yaml

# Save the output addresses back into the repo
cp ~/.hyperlane/chains/hydration/addresses.yaml chains/hydration/addresses.yaml
```

This deploys: Mailbox, ISM/hook factories, InterchainAccountRouter, ValidatorAnnounce, a test recipient —
all EOA-owned, `trustedRelayerIsm` (intentionally insecure until hardening). This is the entire messaging
layer — no per-remote contracts are needed (Ethereum/Base/Solana already run canonical Hyperlane).

**📌 RECORD (hand-back item):**
- The whole `chains/hydration/addresses.yaml` (commit it).
- The **Mailbox address** and its **deploy block number** (the hardening/agents step needs `index.from`).

---

## Step 3 — Smoke-test a message (Hydration → Ethereum)

```bash
hyperlane send message --origin hydration --destination ethereum --relay
```

`--relay` runs a one-off relayer with your `HYP_KEY` and delivers the message (needs the ETH from Step 0).
*(Optionally repeat with `--destination base` if you funded Base.)*

**Verify:** the CLI prints an origin tx, a destination tx, and confirms delivery
(destination `Mailbox.processed(messageId) == true`). ✅ Messaging works.

---

## Step 4 — Solana messaging (no contract to deploy)

**There is nothing to deploy on Solana** — it runs canonical Hyperlane (domain `1399811149`, mailbox
`E588QtVUvresuXq2KoNEwAmoifCzYGpRBdHByN9KQMbi`). Wiring the standing lane (agent config + the ISM that lets
Hydration verify Solana) is the **follow-on** in [SOLANA.md](SOLANA.md). You can prove the lane now with a
self-relay smoke:

```bash
export HYP_KEY_ETHEREUM="$HYP_KEY"          # Hydration is an EVM (protocol=ethereum) chain
export HYP_KEY_SEALEVEL=<your-solana-key>   # from Step 0; see `hyperlane send message --help` for the exact form

hyperlane send message --origin hydration --destination solanamainnet --relay
hyperlane send message --origin solanamainnet --destination hydration --relay
```

**Verify:** delivery on each side (Solana via [solscan.io](https://solscan.io); Hydration via
`Mailbox.processed`). If the cross-VM self-relay errors on the recipient's ISM, that's expected — the
trust-minimised Solana ISM setup is part of the follow-on, not this deploy. ✅ if it delivers.

---

## Step 5 — Hand back

Deliver to whoever runs the hardening follow-on:
- [ ] `chains/hydration/addresses.yaml` (committed) — core contract addresses.
- [ ] Mailbox **deploy block number**.
- [ ] Confirmation which smokes passed (Steps 3, 4).
- [ ] **The deployer key stays with the team** (hardening needs it — do not retire it).

---

## Follow-on (NOT part of this hand-off)

Production hardening turns the insecure-by-design deploy into a safe one — do it before real traffic matters:
run **validators + relayer**, switch the default ISM to the **2-of-3 multisig** (4c), transfer ownership to a
**Safe** (4d), wire **Solana's ISM** (SOLANA.md §3), then the **registry PR** and **monitoring**. See
[README.md](README.md) §4–6. **Adding the token bridge later:** deploy the WETH warp route via README Phase 3
(then 4e/4f to harden it) — no need to redeploy core.

---

## Troubleshooting (deploy-time)

| Symptom | Fix |
|---|---|
| `No chain metadata set for hydration` | `~/.hyperlane/chains/hydration` missing or a symlink — re-run the Step 1 `cp -R` |
| `insufficient funds for gas` | Deployer needs WETH on Hydration (and ETH on the destination for `--relay`) |
| Deploy rejected at gas estimation (`AddressNotWhitelisted` or similar, before any tx) | `HYP_KEY` isn't the whitelisted deployer key |
| `core deploy` can't find config | You omitted `--config configs/hydration/core-config.initial.yaml` |
| Smoke message stays pending | The ephemeral `--relay` needs gas on **both** chains |
| Solana RPC timeouts / stalls | Use a private/paid Solana RPC, not `api.mainnet-beta.solana.com` |
