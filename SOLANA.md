# Hyperlane messaging: Hydration ⇄ Solana (add-on to `README.md`)

Extends the Hydration mainnet runbook to add **interchain messaging** with **Solana mainnet**. This is a
*messaging* add-on only — a WETH/token **warp route to Solana is out of scope** (SPL tokens + separate
Rust `sealevel` warp tooling, much heavier; do messaging first). Read [README.md](README.md) first: this
reuses the *same* Hydration core deploy, the *same* validators, and the *same* relayer — you are only
**wiring Solana into what you already run**.

> Do the whole EVM path (README Phases 0–4) first, or at least Phase 1 (core deploy) + a working relayer.
> Everything here is additive.

## Mental model — why Solana is different but light

- **You deploy NOTHING on Solana.** Solana mainnet already runs canonical Hyperlane (Mailbox, IGP,
  validators operated by Abacus Works et al.). You use it exactly like Ethereum/Base — a pre-existing
  endpoint. The TS CLI's `core deploy` is **EVM-only**; there is no Solana core deploy step.
- **But it's a different VM (SVM / “sealevel”, not EVM).** So the add-on cost is off-chain wiring, not a
  deploy: a **Solana keypair + SOL**, a **private Solana RPC**, a **`sealevel` entry in the agent config**,
  the **relayer's Solana signer**, and **ISM config** for whichever direction(s) you use.
- **You do NOT run validators for Solana.** A validator attests to its *origin* chain; Solana-as-origin is
  covered by Solana's *canonical* validator set. Your validators only ever attest to Hydration.

## Verified facts (canonical registry + SDK + installed CLI, 2026-07-06)

| Item | Value |
|---|---|
| Chain name | `solanamainnet` |
| Domain ID | **`1399811149`** |
| Protocol | `sealevel` |
| Public RPC (rate-limited — get a private one) | `https://api.mainnet-beta.solana.com` |
| Explorer | `https://solscan.io` |
| Mailbox program | `E588QtVUvresuXq2KoNEwAmoifCzYGpRBdHByN9KQMbi` |
| InterchainGasPaymaster program | `JAvHW21tYXE9dtdG83DReqU2b4LUexFuCbtJT5tF8X6M` |
| MerkleTreeHook program (= mailbox program on sealevel) | `E588QtVUvresuXq2KoNEwAmoifCzYGpRBdHByN9KQMbi` |
| ValidatorAnnounce program | `pRgs5vN4Pj7WvFbxf6QDHizo2njq2uksqEUbaSghVA8` |
| Default ISM program | `LwNfVYMDzAe5dCJgA5CipTZcT34Eyf74zLr81K91jxk` |

**Solana's canonical validator set** (needed for the Hydration-side ISM that verifies Solana→Hydration) —
`defaultMultisigConfigs['solanamainnet']`, **threshold 3 of 5**:

| Validator | Address |
|---|---|
| Abacus Works | `0x28464752829b3ea59a497fca0bdff575c534c3ff` |
| Luganodes | `0x2b7514a2f77bd86bbf093fe6bb67d8611f51c659` |
| Eclipse | `0xcb6bcbd0de155072a7ff486d9d7286b0f71dcc2d` |
| Mitosis | `0x4f977a59fdc2d9e39f6d780a84d5b4add1495a36` |
| Zee Prime | `0x5450447aee7b544c462c9352bef7cad049b0c2dc` |

> ⚠️ **Validator sets rotate.** Re-pull the current set at config time from `@hyperlane-xyz/sdk`
> (`defaultMultisigConfigs.solanamainnet`) or the registry — don't rely on this snapshot long-term.

## What you need (delta beyond the EVM deploy)

- [ ] **A Solana keypair for the relayer** — `solana-keygen new --outfile relayer-solana.json` (JSON keypair
      file). Separate from the hex EVM relayer key.
- [ ] **SOL funding** for that key — Solana charges *rent* for accounts the relayer creates on delivery, plus
      tx fees. Hyperlane's own guidance observes **~2.35 SOL rent per account**; **fund with ≥ 5 SOL** and
      alert on it. (Messaging accounts are cheaper than warp accounts, but budget generously.)
- [ ] **A private/paid Solana RPC** (Helius / Triton / QuickNode-class). `api.mainnet-beta.solana.com`
      rate-limits hard and will stall an agent.
- [ ] **Nothing to deploy on Solana**, and **no Solana validators to run.**

## Step 1 — add `solanamainnet` to the agent config

Add this chain block to [`configs/agent-config.mainnet.json`](configs/agent-config.mainnet.json) alongside
`ethereum`/`base`. The program IDs are canonical (from the registry); the RPC is your private one:

```jsonc
"solanamainnet": {
  "protocol": "sealevel",
  "domainId": 1399811149,
  "name": "solanamainnet",
  "rpcUrls": [{ "http": "https://your-private-solana-rpc" }],
  "mailbox": "E588QtVUvresuXq2KoNEwAmoifCzYGpRBdHByN9KQMbi",
  "interchainGasPaymaster": "JAvHW21tYXE9dtdG83DReqU2b4LUexFuCbtJT5tF8X6M",
  "merkleTreeHook": "E588QtVUvresuXq2KoNEwAmoifCzYGpRBdHByN9KQMbi",
  "validatorAnnounce": "pRgs5vN4Pj7WvFbxf6QDHizo2njq2uksqEUbaSghVA8",
  "index": { "mode": "sequence" }
}
```

Notes:
- `index.mode` for sealevel is **`sequence`** (Solana has no EVM-style block/`from` cursor). Confirm against
  `hyperlane registry list` / your agent version.
- Because `solanamainnet` is canonical, you *can* instead give only `rpcUrls` (like `ethereum`/`base`) and let
  the agent pull the rest from the registry — but the explicit block above is self-contained and safer.

## Step 2 — wire the relayer for the Solana lane

In [`deploy/relayer/docker-compose.yml`](deploy/relayer/docker-compose.yml) / its `.env`:

1. **Add Solana to the relay set and RPCs** (`deploy/relayer/.env`):
   ```bash
   RELAY_CHAINS=hydration,ethereum,base,solanamainnet
   SOLANA_RPC_URLS=https://your-private-solana-rpc
   ```
   and pass `--chains.solanamainnet.customRpcUrls=${SOLANA_RPC_URLS:?set in .env}` in the compose `command`.

2. **Give the relayer its Solana signer.** The relayer signs on each chain it delivers to, so it needs a
   *Solana* signer in addition to the hex EVM key. Mount the keypair and configure a per-chain signer for
   `solanamainnet`:
   ```yaml
   # in the relayer service:
   #   - --chains.solanamainnet.signer.type=<confirm for sealevel>
   #   - --chains.solanamainnet.signer.key=${SOLANA_KEYPAIR:?set in .env}   # or a mounted keypair path
   ```
   > ⚠️ **Verify before prod:** the exact `signer.type` and key encoding (hex vs base58 vs mounted JSON
   > keypair) for a **sealevel** signer is agent-version-specific and I could not confirm it from an
   > authoritative source — check the current Hyperlane agent-configuration docs
   > (`docs.hyperlane.xyz` → operate → agent config) or `docs/…/svm` before relying on this. The EVM
   > `hexKey` type is **not** assumed to work for Solana.

3. **Extend the whitelist** to the Solana domain (keep it — without it the relayer would index Solana's
   canonical mailbox and pay to deliver all third-party Solana traffic):
   ```
   --whitelist=[{"originDomain":222222},{"destinationDomain":222222},{"originDomain":1399811149,"destinationDomain":222222},{"originDomain":222222,"destinationDomain":1399811149}]
   ```
   (Adjust to the direction(s) you actually run.)

4. **Fund** the Solana key with ≥ 5 SOL and add a low-SOL alert (mirror the EVM gas alerts in
   [`deploy/monitoring/alerts.yml`](deploy/monitoring/alerts.yml)).

## Step 3 — ISM config (per direction — the part to get right)

A MultisigISM on the **destination** verifies a message using the **origin's** validator set. So the two
directions need *different* validator sets:

### Solana → Hydration (verify inbound on Hydration)
The Hydration recipient's ISM must carry **Solana's** validators (the 3-of-5 set above), keyed to origin
domain `1399811149`. If Hydration **only** receives from Solana, set the Hydration Mailbox default ISM to:

```yaml
defaultIsm:
  type: messageIdMultisigIsm
  threshold: 3
  validators:
    - "0x28464752829b3ea59a497fca0bdff575c534c3ff"   # Abacus Works
    - "0x2b7514a2f77bd86bbf093fe6bb67d8611f51c659"   # Luganodes
    - "0xcb6bcbd0de155072a7ff486d9d7286b0f71dcc2d"   # Eclipse
    - "0x4f977a59fdc2d9e39f6d780a84d5b4add1495a36"   # Mitosis
    - "0x5450447aee7b544c462c9352bef7cad049b0c2dc"   # Zee Prime
```

If Hydration receives from **multiple** origins (e.g. Solana *and* EVM chains, each with its own validator
set), **do not** overwrite the default with one origin's set — use a **`domainRoutingIsm`** that maps each
origin domain to its own multisig:

```yaml
defaultIsm:
  type: domainRoutingIsm
  owner: "<DEPLOYER_EOA_OR_SAFE>"
  domains:
    1399811149:           # Solana
      type: messageIdMultisigIsm
      threshold: 3
      validators: [ ... the 5 Solana validators above ... ]
    # <ethereum-domain>: { type: messageIdMultisigIsm, threshold: N, validators: [ ...ethereum's set... ] }
```

Apply it the **same way as Phase 4c** — `core read`/`core apply` with **explicit `--config /tmp/…` on both**
(see README §4c; a bare `core apply` no-ops).

> **Important distinction:** README Phase 4c sets Hydration's multisig to **your** validators. Those secure
> messages **leaving** Hydration — they are enforced by the **remote** chain's ISM, not Hydration's. Verifying
> messages **arriving** at Hydration (from Solana, or from Ethereum/Base) needs **that origin's** validators
> here. Make sure your inbound ISM actually lists the origins' sets, not your own. *(This is worth
> double-checking for the EVM directions too — see the note at the bottom.)*

### Hydration → Solana (verify inbound on Solana)
The **Solana recipient program** picks its ISM. Two cases:
- **Smoke test / self-relay:** `--relay` (trusted-relayer / self relay) delivers regardless of ISM — nothing
  to configure. Good for proving the lane.
- **Production, trust-minimized:** the recipient on Solana must be configured to verify **Hydration's**
  validators (origin domain `222222`). If you don't already control a suitable recipient + ISM on Solana,
  that means deploying/configuring a small **sealevel** recipient/ISM program (Rust tooling) — the one place
  where “messaging to Solana” can require a Solana-side deployment.

## Step 4 — smoke test (both directions)

The CLI needs a signer for **each** protocol it touches, supplied as per-protocol keys:

```bash
export HYP_KEY_ETHEREUM=0x...            # hex EVM key (Hydration is protocol=ethereum)
export HYP_KEY_SEALEVEL=<solana-key>     # Solana key (base58/secret) — confirm exact form via `hyperlane send message --help`

# Hydration -> Solana
hyperlane send message --origin hydration --destination solanamainnet --relay

# Solana -> Hydration
hyperlane send message --origin solanamainnet --destination hydration --relay
```

`--relay` self-relays with those keys (works cross-VM). Verify the message shows delivered/processed on the
destination (Hydration `Mailbox.processed(id)`; Solana via solscan or the CLI status).

## What the TS CLI can't do for Solana (limits)

- **No `core deploy` to Solana** — EVM-only; Solana core is canonical (used, not deployed).
- **No TS-CLI sealevel warp deploy** — a warp route to Solana uses the separate Rust `sealevel` client and
  `solana-keygen` keys (out of scope here).
- Managing Solana ISMs/recipients beyond the canonical defaults is **Rust/sealevel tooling**, not the TS CLI.

## Verify-before-prod checklist

- [ ] Relayer **`signer.type`/key encoding for sealevel** — confirmed against current agent docs (the one
      thing I could not authoritatively pin here).
- [ ] Solana **validator set + threshold** re-pulled fresh (they rotate).
- [ ] `index.mode: sequence` accepted by your agent version.
- [ ] SOL balance ≥ 5 with a low-balance alert wired.
- [ ] Whitelist restricted to the Hydration↔Solana domains you actually run.
- [ ] Inbound ISM lists the **origins'** validator sets, not your own (see the distinction in Step 3).
