# Production deployment roadmap

End-to-end plan from current state (configs prepared) to production-hardened Hyperlane on Hydration. Steps are ordered. Skip phases only when explicitly noted as optional.

Reference: [docs.hyperlane.xyz/docs/guides/production/prod-overview](https://docs.hyperlane.xyz/docs/guides/production/prod-overview).

---

## Phase 0 — Prep (done)

- [x] `chains/hydration/metadata.yaml` — chain metadata, registered with CLI via symlink
- [x] `configs/core-config.yaml` — owner / relayer / ISM / hook config (trustedRelayerIsm, owner EOA)
- [ ] Deployer EOA funded with ≥ 0.01 WETH on Hydration EVM
- [ ] `logo.svg` (only needed at canonical-registry PR time, not for deploy)

---

## Phase 1 — Core deploy (initial, EOA-owned, trusted-relayer ISM)

> See [`README.md`](./README.md) for the detailed runbook. Summary here.

1. **Install CLI**: `npm i -g @hyperlane-xyz/cli`
2. **Symlink chain into Hyperlane registry**:
   ```bash
   mkdir -p ~/.hyperlane/chains
   ln -s "$(pwd)/chains/hydration" ~/.hyperlane/chains/hydration
   ```
3. **Export deployer key** (ephemeral shell):
   ```bash
   HISTFILE=/dev/null; read -rs HYP_KEY; export HYP_KEY
   ```
4. **Deploy**: `hyperlane core deploy --chain hydration`
5. **Commit** `chains/hydration/addresses.yaml`

**Output**: Mailbox + ISM factories + hook factories + ICA router + ValidatorAnnounce + testRecipient addresses.

---

## Phase 2 — Smoke test

```bash
hyperlane send message --origin hydration --destination ethereum --relay
```

`--relay` runs an ephemeral relayer using `HYP_KEY`. Requires:
- Hydration EOA balance for dispatch (already has it from phase 1)
- A tiny amount of ETH on Ethereum for delivery
- Ethereum is in the canonical registry already, so no extra config

Verify: tx hashes printed, message marked delivered. Check the Mailbox `processed(messageId)` view on the destination.

---

## Phase 3 — Warp route deployment (optional)

Skip if you only need raw message-passing. Required if you want to bridge a specific token (HOLLAR, HDX-as-ERC20, USDC, etc.).

1. `hyperlane warp init` — interactive, writes `configs/warp-route-deployment.yaml`. Pick:
   - Token type per chain: `native`, `collateral` (existing ERC20 on origin → wrapped on destination), `synthetic` (mint/burn on origin → unlock on destination)
   - Mailbox + owner addresses (use values from `chains/hydration/addresses.yaml`)
2. `hyperlane warp deploy` — deploys per-chain warp route contracts and links them.
3. Output: warp config under `chains/hydration/` registry path with route name (e.g., `hydration-ethereum-USDC.yaml`).
4. Test transfer: `hyperlane warp send` (small amount).

References:
- [Warp route types](https://docs.hyperlane.xyz/docs/protocol/warp-routes/warp-routes-overview)
- [Permissionless warp deploy](https://docs.hyperlane.xyz/docs/guides/quickstart/deploy-warp-route)

---

## Phase 4 — Production hardening

The phase-1 deploy is intentionally insecure (single EOA owner, single trusted relayer). Phase 4 fixes that.

### 4a — Run your own validator(s)

A validator signs Merkle root checkpoints of the Hydration Mailbox so destination chains can verify Hydration-origin messages.

1. **Provision a validator signing key** (AWS KMS recommended, hex for testing). [Agent Keys docs](https://docs.hyperlane.xyz/docs/operate/set-up-agent-keys).
2. **Provision checkpoint storage** — S3 bucket (production) or local FS (testing). [AWS bucket setup](https://docs.hyperlane.xyz/docs/operate/validators/validator-signatures-aws).
3. **Provision dedicated RPC** — *do not* use public RPCs for validator (rate limits, finality risk).
4. **Run validator** via Docker:
   ```bash
   docker run \
     -e AWS_ACCESS_KEY_ID=... -e AWS_SECRET_ACCESS_KEY=... \
     -v /opt/hyperlane_db:/hyperlane_db \
     ghcr.io/hyperlane-xyz/hyperlane-agent:agents-v2.2.0 ./validator \
     --db /hyperlane_db \
     --originChainName hydration \
     --chains.hydration.customRpcUrls "https://your-private-rpc" \
     --chains.hydration.blocks.reorgPeriod finalized \
     --validator.type aws \
     --validator.id alias/hyperlane-validator-hydration \
     --validator.region us-east-1 \
     --checkpointSyncer.type s3 \
     --checkpointSyncer.bucket your-bucket \
     --checkpointSyncer.region us-east-1
   ```
5. **Fund the validator EOA** with a small WETH amount — needed once for the `ValidatorAnnounce.announce()` tx.
6. Wait for first checkpoint files to appear in S3. Repeat for **N validators** if you want N-of-M security (run on separate keys/hosts/RPCs).

### 4b — Run your own relayer

A relayer subscribes to origin-chain Mailbox events and submits `Mailbox.process()` on destination chains.

1. **Provision relayer signing key** (separate from validator; AWS KMS or hex).
2. **Fund the relayer EOA** on **every destination chain** it will deliver to (Ethereum, Base, Hydration, …).
3. **Run relayer** via Docker:
   ```bash
   docker run \
     -e AWS_ACCESS_KEY_ID=... -e AWS_SECRET_ACCESS_KEY=... \
     -v /opt/hyperlane_db:/hyperlane_db \
     ghcr.io/hyperlane-xyz/hyperlane-agent:agents-v2.2.0 ./relayer \
     --db /hyperlane_db \
     --relayChains hydration,ethereum,base \
     --defaultSigner.type aws \
     --defaultSigner.id alias/hyperlane-relayer \
     --defaultSigner.region us-east-1 \
     --chains.hydration.customRpcUrls "https://your-private-hydration-rpc" \
     --chains.ethereum.customRpcUrls "https://your-private-eth-rpc" \
     --chains.base.customRpcUrls "https://your-private-base-rpc" \
     --allowLocalCheckpointSyncers false
   ```
4. Monitor relayer logs for `submitted` events; failed deliveries usually mean insufficient gas on a destination chain.

### 4c — Switch default ISM from trustedRelayer → multisig

Only do this **after** Phase 4a — your validators must already be producing checkpoints.

1. **Read current ISM** for sanity:
   ```bash
   hyperlane core read --chain hydration
   ```
2. **Edit `configs/core-config.yaml`** — replace the `defaultIsm` block:
   ```yaml
   defaultIsm:
     type: messageIdMultisigIsm
     threshold: 2                # require ≥2 of the listed validators
     validators:
       - "0x<validator-1-evm-addr>"
       - "0x<validator-2-evm-addr>"
       - "0x<validator-3-evm-addr>"
   ```
   Choose `threshold` based on your trust model — N-of-M where M = total validators.
3. **Apply**:
   ```bash
   hyperlane core apply --chain hydration
   ```
4. **Verify**:
   ```bash
   hyperlane core read --chain hydration   # should now show messageIdMultisigIsm
   ```
5. **Commit** the updated `core-config.yaml`.

### 4d — Transfer ownership to a multisig

You want the *owner* of Mailbox / ProxyAdmin / ISMs to be a Gnosis Safe, not an EOA.

1. **Deploy or pick a Safe** on Hydration EVM. Note its address. (Safe doesn't yet have a Transaction Service on Hydration — coordination off-chain.)
2. **Edit `configs/core-config.yaml`** — change every `owner:` field (top-level, `proxyAdmin.owner`, `requiredHook.owner`) to the Safe address.
3. **Apply**:
   ```bash
   hyperlane core apply --chain hydration
   ```
4. **Verify** with `hyperlane core read --chain hydration` — confirm new owner everywhere.
5. **From now on**, ISM/hook changes require Safe approval. The deployer EOA is no longer privileged.

### 4e — Remove trusted relayer from warp routes (if Phase 3 was done)

If you deployed warp routes in Phase 3, they have a separate trusted-relayer config. Strip it for production.

1. `hyperlane warp read --warp <warp-config>` — inspect current config
2. Edit warp config — remove the trusted relayer fields
3. `hyperlane warp apply` to commit on-chain
4. Reference: [docs.hyperlane.xyz/docs/guides/production/warp-route-deployment/remove-trusted-relayer](https://docs.hyperlane.xyz/docs/guides/production/warp-route-deployment/remove-trusted-relayer)

### 4f — Transfer warp route ownership (if Phase 3 was done)

Mirror 4d for each warp route deployed. [transfer-warp-route-ownership](https://docs.hyperlane.xyz/docs/guides/production/warp-route-deployment/transfer-warp-route-ownership).

---

## Phase 5 — Canonical registry PR

Only after a working deployment + smoke test.

1. Add to `chains/hydration/metadata.yaml`:
   ```yaml
   deployer:
     name: <your team / org>
     url: <homepage>
   gasCurrencyCoinGeckoId: weth
   ```
2. Add `chains/hydration/logo.svg` (square, vector, Hydration brandmark).
3. Fork + PR:
   ```bash
   gh repo fork hyperlane-xyz/hyperlane-registry --clone
   cd hyperlane-registry
   mkdir -p chains/hydration
   cp ../hyperlane-deploy/chains/hydration/{metadata.yaml,addresses.yaml,logo.svg} chains/hydration/
   yarn changeset add
   git checkout -b add-hydration
   git add . && git commit -m "Add hydration chain"
   git push -u origin add-hydration
   gh pr create
   ```

---

## Phase 6 — Ongoing operations

| Concern | Action |
|---|---|
| Validator key rotation | Rotate signing key; old key still verifies past checkpoints. Update Mailbox ISM with new validator address via `hyperlane core apply`. |
| Validator host failure | Run ≥ 2× validators on independent infra; threshold should be ≤ N-1 so one failure doesn't halt the bridge. |
| Relayer host failure | Stateless — run hot-spare with the same key, only one delivers (idempotent at the Mailbox layer). |
| RPC failure | Multiple `customRpcUrls`; EVM supports quorum mode for higher integrity. |
| Gas top-ups | Monitor relayer/validator EOA balances on every chain. Alert when balance < N days of typical burn. |
| Message stuck | Check destination `Mailbox.processed(messageId)`. If false, check relayer logs for delivery error (most often: insufficient gas estimate, RPC flake). |
| Adding new destination chain | Add chain to relayer `--relayChains`, ensure Hydration Mailbox knows the destination domain (no on-chain action needed — domain IDs are global). |
| Adding new warp route | Phase 3 again for the new token. |
| ISM policy change | `hyperlane core apply` (requires owner signature, which after 4d is the Safe). |

### Monitoring checklist

- Validator checkpoint freshness (S3 file last-modified < threshold)
- Relayer delivery latency (event-emitted → tx-confirmed)
- Mailbox `dispatched - processed` queue depth per chain
- Validator + relayer EOA gas balances per chain
- RPC error rate

Hyperlane provides [Prometheus metrics endpoints](https://docs.hyperlane.xyz/docs/operate/monitoring) on the validator/relayer; scrape into your existing monitoring stack.
