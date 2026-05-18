# Relayers, validators, and ISMs

A field guide to the off-chain components of Hyperlane: what they do, what trust assumptions they introduce, and how the bridge evolves from a trusted-relayer testnet setup to a permissionless multisig-secured production deployment.

If you're here because you finished a deploy and want to understand "how does this thing actually work in production" — this doc.

For procedural steps, see [PRODUCTION.md](./PRODUCTION.md).

---

## The three components, summarized

| Component | On-chain or off? | What it does | Who runs it |
|---|---|---|---|
| **Mailbox** | On-chain | Origin: emits Dispatch events. Destination: verifies + delivers messages. | Deployed once per chain. |
| **Validator** | Off-chain | Signs Mailbox Merkle root after each block. Publishes signed checkpoints to durable storage (S3 / IPFS / local FS). | N independent operators. |
| **Relayer** | Off-chain | Watches Dispatch events. Fetches matching validator signatures. Submits `Mailbox.process()` on destination. | Anyone. Multiple relayers can race; deliveries are idempotent. |
| **ISM** | On-chain | The verification policy on the destination side. Defines what proof a `process()` call must include. | Configured per warp route or as Mailbox default. |

The ISM is the **trust anchor**. Validators are the trust assumption. The relayer is just transport.

---

## Local-agent setup (what we have now)

For the Lark ↔ Base Sepolia test bridge, we run a simplified version:

```
   ┌─────────────────────┐                                  ┌─────────────────────┐
   │  Lark Mailbox       │                                  │  Base Sepolia       │
   │  (your deploy)      │                                  │  Mailbox (canonical)│
   └──────────┬──────────┘                                  └──────────▲──────────┘
              │                                                         │
              │ Dispatch event                                          │ Mailbox.process(
              │                                                         │   minimal metadata,
              │                                                         │   message)
              ▼                                                         │
       ┌─────────────────────────────────────────────────────────────────────┐
       │                                                                       │
       │                       Local Relayer (Docker)                          │
       │                                                                       │
       │  • Indexes Dispatch events on both chains                              │
       │  • Builds minimal metadata (just the message — no validator sigs)     │
       │  • Submits process() with our trusted relayer key as msg.sender       │
       │                                                                       │
       └─────────────────────────────────────────────────────────────────────┘
                              │
                              │ Signs as 0x222…80 (on Lark)
                              │ Signs as 0x238…3c (on Base Sepolia)
                              ▼
                       ┌─────────────────────┐
                       │ trustedRelayerIsm    │
                       │                      │
                       │ verify() returns true│
                       │ iff                  │
                       │ msg.sender == trusted│
                       │ relayer address      │
                       └──────────────────────┘
```

Trust model: one key = one bridge security guarantee. If our relayer key leaks, an attacker can forge any message into Lark or Base Sepolia.

This is fine for a testnet. Not for production.

---

## Production setup

```
                       ORIGIN CHAIN                                    DESTINATION CHAIN
                            │                                                  ▲
   1. user dispatches      │                                                  │
      Mailbox.dispatch(...)│                                                  │
                            ▼                                                  │
                ┌──────────────────────┐                                        │
                │       Mailbox        │                                        │
                │                      │                                        │
                │  • Dispatch event    │                                        │
                │  • merkleTreeHook    │                                        │
                │    inserts msg id    │                                        │
                │  • new tree root     │                                        │
                └─────────┬────────────┘                                        │
                          │                                                     │
        ┌─────────────────┼────────────────┐                                    │
        │                 │                │                                    │
        ▼                 ▼                ▼                                    │
   ┌────────┐       ┌────────┐       ┌────────┐                                 │
   │valid-  │       │valid-  │       │valid-  │   2. each validator              │
   │ator 1  │       │ator 2  │       │ator 3  │     signs (root, idx)            │
   │        │       │        │       │        │     after every                  │
   │ signs  │       │ signs  │       │ signs  │     update                       │
   │ root   │       │ root   │       │ root   │                                  │
   └────┬───┘       └────┬───┘       └────┬───┘                                 │
        │                │                │                                    │
        │ store          │ store          │ store                              │
        ▼                ▼                ▼                                    │
   ┌──────────────────────────────────────────┐                                 │
   │   Validator storage (S3 / IPFS / local)   │  3. signatures published         │
   │                                          │                                 │
   │   /v1/<chain>/<root>/<idx>.sig            │                                 │
   └──────────────┬───────────────────────────┘                                 │
                  │                                                              │
                  │ 4. relayer fetches:                                          │
                  │    - Dispatch events                                          │
                  │    - validator sigs                                          │
                  │    - merkle proof                                            │
                  ▼                                                              │
   ┌──────────────────────────────────────────┐                                 │
   │              Relayer                      │  5. builds metadata =            │
   │                                          │     [N sigs, merkle proof,       │
   │   • collects ≥ threshold sigs            │      origin merkle index]        │
   │   • assembles ABI-encoded metadata        │                                 │
   │   • Mailbox.process(metadata, message)    │  6. submits on destination     │
   └──────────────────────────────────────────┘──────────────────────────────────┘
                                                                                  │
                                                                                  ▼
                                                      ┌─────────────────────────────┐
                                                      │   messageIdMultisigIsm       │
                                                      │                              │
                                                      │   • recover signer addrs     │
                                                      │     from each signature      │
                                                      │   • count enrolled validators│
                                                      │   • require count ≥ threshold│
                                                      │   • verify merkle proof      │
                                                      │     of message id at root    │
                                                      │                              │
                                                      │   → returns true / false     │
                                                      └─────────────────────────────┘
```

Critical: **the relayer is permissionless.** Anyone can call `process()` as long as they include valid signatures and the proof. The destination ISM does all the verification work.

---

## Trust model comparison

|  | trustedRelayerIsm | messageIdMultisigIsm |
|---|---|---|
| Validators required | 0 | N (typically 3–7) |
| Relayer keys trusted | 1 (specific address) | 0 — any address can deliver |
| Compromise of single key | full bridge takeover | requires ≥ threshold validator compromises |
| Operational cost | one relayer process | N validator processes + ≥1 relayer |
| Latency | ~1 block (immediate sign+deliver) | ~1 block + validator polling interval |
| Storage cost | none | ~10kB/checkpoint × N validators × time |
| Suitable for | testnets, dev, demos | mainnet |

---

## How validators work

A validator is a single Rust binary (`./validator` inside the Hyperlane agent Docker image). Its loop:

```
loop {
    1. fetch latest merkleTreeHook root from Mailbox
    2. if new root since last signed:
         compute (root, leaf_count)
         sign with validator_key
         write signed JSON blob to checkpointSyncer
            (e.g. s3://<bucket>/<chain>/<index>.json)
    3. sleep interval
}
```

The validator is **stateless per-message** — it doesn't track individual messages, just the Merkle root. Any message inserted into the tree before the root was signed becomes verifiable.

### Inputs the validator needs

| Input | Why |
|---|---|
| **`--originChainName <chain>`** | Which chain's Mailbox to watch |
| **`--validator.key`** (or AWS KMS alias) | Signing key. Goes into recovered `signer` in onchain verify. |
| **`--checkpointSyncer.type s3`** + bucket | Where signed checkpoints land |
| **`--db /hyperlane_db`** | Local persistence for the validator's view |
| Private RPC to origin chain | Don't share with relayer; validators need fast finality reads |
| Small amount of native token at the validator EOA address | One-time tx to `ValidatorAnnounce.announce(storageLocation)` so relayers know where to fetch sigs |

### Validator registration (`ValidatorAnnounce`)

Validators don't have to be added to an on-chain list to start signing — they just sign and publish. But for a **relayer to find them**, they call `ValidatorAnnounce.announce()` exactly once, telling the world where their checkpoint storage is.

```solidity
// ValidatorAnnounce
function announce(
    address validator,
    string calldata storageLocation,
    bytes calldata signature
) external;
```

After this one-time call, anyone running a relayer can discover the storage URL and start verifying signatures from that validator.

---

## How the multisig ISM works

`messageIdMultisigIsm` is configured with:

- A list of authorized **validator addresses** (the EOA addresses they sign with)
- A **threshold** (e.g., 3-of-5)

Verify call:

```solidity
function verify(bytes metadata, bytes message) external returns (bool) {
    // metadata format:
    //   origin merkle root
    //   merkle proof of message.id at root
    //   N signatures (variable)
    
    (bytes32 root, bytes32[] memory proof, bytes[] memory signatures) = decode(metadata);
    
    // 1. message.id must hash through the proof to the claimed root
    require(MerkleLib.verify(message.id, proof, root), "bad proof");
    
    // 2. each signature must recover to an enrolled validator
    address[] memory validators = abi.decode(constructorBytes, (address[]));
    uint8 threshold = abi.decode(constructorBytes, (uint8));
    
    uint256 valid = 0;
    for (uint i = 0; i < signatures.length; i++) {
        address signer = ECDSA.recover(checkpointHash(root, leaf_count), signatures[i]);
        if (isEnrolled(signer, validators)) valid++;
    }
    
    return valid >= threshold;
}
```

This is a static aggregation — the validator set and threshold are encoded into the ISM's constructor bytes, which means the ISM's address itself is a hash of those parameters. **Rotating validators = deploying a new ISM and pointing the recipient/Mailbox at it.**

For dynamic validator sets, use `domainRoutingIsm` + per-origin static multisigs (still static per origin) or build a custom ISM.

---

## Migration: trustedRelayerIsm → multisig

The procedure for switching your bridge from "us as the relayer" to "permissionless production":

### Step 1 — Stand up validators (off-chain)

Run N validator binaries on separate hosts/keys. Each one:
- Signs Mailbox checkpoints on whichever chain it validates
- Publishes signatures to its own S3 bucket
- Calls `ValidatorAnnounce.announce()` once

This is the longest step. AWS KMS for signing keys, S3 for storage, monitoring on top.

### Step 2 — Update Mailbox.defaultIsm to multisigIsm

```bash
# Edit configs/core-config.yaml:
defaultIsm:
  type: messageIdMultisigIsm
  threshold: 3
  validators:
    - "0xValidator1..."
    - "0xValidator2..."
    - "0xValidator3..."
    - "0xValidator4..."
    - "0xValidator5..."

hyperlane core apply --chain <chain>
```

This changes the **fallback** ISM for any recipient that doesn't override `interchainSecurityModule()`.

### Step 3 — Verify multisig works end-to-end

Send a test message via the production relayer (or your own). Confirm it delivers through the multisig ISM and your validators' signatures.

### Step 4 — Remove trusted relayer from warp route

The [linked docs page](https://docs.hyperlane.xyz/docs/guides/production/warp-route-deployment/remove-trusted-relayer) covers this.

```bash
hyperlane warp read -w <warpRouteId>      # inspect current ISM

# Edit warp-route-deployment.yaml — remove the trustedRelayerIsm module:
basesepolia:
  interchainSecurityModule:
    type: defaultFallbackRoutingIsm   # falls back to Mailbox.defaultIsm
    owner: 0x...
    domains: {}                       # empty → use Mailbox default
```

```bash
hyperlane warp apply -w <warpRouteId> --strategy ...
```

After apply: warp router's `interchainSecurityModule()` returns the new fallback ISM. Inbound messages now have to pass the Mailbox's defaultIsm (the multisig). No more single relayer trust.

### Step 5 — Optionally retire the trusted relayer key

Your old `0x222…80` key can stop running the relayer. Other relayers (or the canonical Abacus Works relayer, if your chain is in the registry) take over delivery.

---

## How a production relayer is run

The same Docker binary as our local agent, with different config:

```bash
docker run --restart=always -d \
  -e CONFIG_FILES=/config/agent-config.json \
  --mount type=bind,source=/etc/hyperlane/agent-config.json,target=/config/agent-config.json,readonly \
  -v /var/lib/hyperlane/relayer-db:/hyperlane_db \
  ghcr.io/hyperlane-xyz/hyperlane-agent:agents-vX.X.X \
  ./relayer \
  --db /hyperlane_db \
  --relayChains lark,basesepolia,ethereum,base,arbitrum,optimism,... \
  --defaultSigner.type aws \
  --defaultSigner.id alias/hyperlane-relayer \
  --defaultSigner.region us-east-1 \
  --chains.lark.customRpcUrls "https://your-private-lark-rpc" \
  --chains.basesepolia.customRpcUrls "https://your-private-base-rpc" \
  --metrics-port 9090
```

Differences from our local setup:

| Setting | Local | Production |
|---|---|---|
| Signer | hex private keys in CLI args | AWS KMS (key never leaves the HSM) |
| RPC URLs | public endpoints | private RPCs (paid Alchemy/Infura/Tenderly) |
| Storage | local Docker volume | persistent EBS / network FS, backed up |
| Processes | one in `-it` foreground | systemd / k8s with `--restart=always` |
| Funding | the relayer's hot key has gas on every chain it relays | same, but monitored with alerts when balance < threshold |
| Monitoring | tail the logs | Prometheus scrape on `--metrics-port`, alerting |

---

## Failure modes and how the bridge handles them

| Failure | Effect | Recovery |
|---|---|---|
| Single validator goes down | Threshold still met if other validators online | Restart validator; signatures resume from where it left off |
| Validators lose quorum (>N-threshold offline) | Bridge halts — relayers can't build valid metadata | Operationally critical. Get validators back up or do an emergency ISM swap. |
| Relayer goes down | Messages queue at origin; no delivery until someone resumes | Anyone can resume — relayer is stateless |
| Validator key compromise | Attacker can sign fake checkpoints | Replace validator address on the multisig ISM (`core apply` with new validator set). Or rotate threshold so compromised key alone can't meet it. |
| Origin RPC goes flaky | Validators can't read root; relayer can't read events | Add more RPCs to fallback list; pay for a private one |
| Destination RPC throttles | Relayer queues retries (this happened to us with Base Sepolia public RPC) | Same — use a paid RPC for destination |
| Wrong threshold (too low) | Single compromise can forge | Raise threshold via `core apply` and redeploy multisigIsm with stricter params |
| Wrong threshold (too high) | One validator down halts the bridge | Lower threshold via `core apply` |

---

## Decision points before going to production

1. **How many validators?** 5 is a common balance for permissioned testnets; 7-10 for mainnet bridges that move serious volume. Threshold ~⅔.
2. **Who runs them?** Mixed parties (you, validating-as-a-service providers, foundation, partner chains) is harder to attack than one team running all validators on the same provider.
3. **Where do they store signatures?** S3 is standard. Make sure buckets are publicly readable (validators publish, anyone can fetch).
4. **What's your validator key strategy?** AWS KMS is the canonical answer. Never put hex keys on disk on a long-running validator host.
5. **Who runs the relayer?** You can run one or rely on the canonical Hyperlane relayer (if your chain is in the canonical registry). Multiple is fine and reduces latency.
6. **What's the ownership migration?** Owner of Mailbox + ProxyAdmin + warp router → Gnosis Safe (multi-signer), not a single EOA. See PRODUCTION.md phase 4d.

---

## References

- [Hyperlane docs: production guide overview](https://docs.hyperlane.xyz/docs/guides/production/prod-overview)
- [Hyperlane docs: remove trusted relayer](https://docs.hyperlane.xyz/docs/guides/production/warp-route-deployment/remove-trusted-relayer)
- [Hyperlane docs: run validators](https://docs.hyperlane.xyz/docs/operate/validators/run-validators)
- [Hyperlane docs: run relayers](https://docs.hyperlane.xyz/docs/operate/relayer/run-relayer)
- [Hyperlane docs: agent keys](https://docs.hyperlane.xyz/docs/operate/set-up-agent-keys)
- [PRODUCTION.md](./PRODUCTION.md) phase 4 — procedural steps for this same migration
