# Architecture

End-to-end Hyperlane deployment: Lark (Hydration testnet, chainId 222222) ↔ Base Sepolia (chainId 84532), with a working WETH warp route and a self-hosted relayer.

---

## The big picture

```
   ┌─────────────────────────────────┐                    ┌──────────────────────────────────────────┐
   │           Lark (222222)          │                    │            Base Sepolia (84532)           │
   │      Hydration parachain          │                    │            Coinbase L2 testnet            │
   │      Frontier EVM (Substrate)    │                    │            OP-stack EVM                   │
   ├─────────────────────────────────┤                    ├──────────────────────────────────────────┤
   │                                 │                    │                                          │
   │  ┌───────────────────────────┐  │                    │  ┌────────────────────────────────────┐  │
   │  │   Mailbox (deployed by    │  │                    │  │   Mailbox (canonical Abacus Works) │  │
   │  │   us, this repo)          │  │                    │  │   0x6966b0E5…2039                  │  │
   │  │   0xa4A46140…016c         │  │                    │  └─────────────┬──────────────────────┘  │
   │  └─────────────┬─────────────┘  │                    │                │                         │
   │                │                │                    │                │                         │
   │  ┌─────────────▼─────────────┐  │                    │  ┌─────────────▼─────────────────────┐  │
   │  │  HypNative warp router    │  │                    │  │  HypERC20 warp router              │  │
   │  │  0x8d974234…1cDD          │  │                    │  │  0x37884726…52bF                   │  │
   │  │  • locks WETH on send     │◀═════════════════════════│  • mints synthetic WETH on receive │  │
   │  │  • releases WETH on recv  │═════════════════════════▶│  • burns synthetic WETH on send    │  │
   │  └─────────────┬─────────────┘  │       enrolled       │  └─────────────┬──────────────────────┘  │
   │                │                │     remote routers    │                │                         │
   │  ┌─────────────▼─────────────┐  │                    │  ┌─────────────▼─────────────────────┐  │
   │  │  trustedRelayerIsm         │  │                    │  │  trustedRelayerIsm                  │  │
   │  │  0x73df7796…968b           │  │                    │  │  0xEd5AA25e…5Ed7                    │  │
   │  │  trusts 0x222…80           │  │                    │  │  trusts 0x238…3c                    │  │
   │  └────────────────────────────┘  │                    │  └────────────────────────────────────┘  │
   │                                 │                    │                                          │
   └────────┬────────────────────────┘                    └────────────────────────────────┬─────────┘
            │                                                                                │
            │ JSON-RPC (https://node3.lark.hydration.cloud)                                   │ JSON-RPC (basesepolia)
            │                                                                                │
            ▼                                                                                ▼
   ┌───────────────────────────────────────────────────────────────────────────────────────────────┐
   │                          Local Hyperlane Relayer (Docker)                                       │
   │                                                                                                 │
   │  • Watches both Mailboxes for `Dispatch` events                                                  │
   │  • Builds delivery metadata (single trustedRelayerIsm proof per route)                          │
   │  • Submits `Mailbox.process()` on the destination                                                │
   │                                                                                                 │
   │  Signers:                                                                                       │
   │    lark        → 0x222222ff…9D80     (also the trusted relayer for lark)                        │
   │    basesepolia → 0x23812ff0…7D3c     (also the trusted relayer for basesepolia)                 │
   └───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Deployed contracts (Lark)

Owner of all contracts: `0x222222ff7Be76052e023Ec1a306fCca8F9659D80`

| Contract | Address | Role |
|---|---|---|
| `mailbox` | `0xa4A46140fA8ea6979528AA0A11FDf8526F57016c` | Entry point. `dispatch()` on origin, `process()` on destination. |
| `proxyAdmin` | `0x66B7A499A27c110C0ebAC17E5998463Fc0633E2C` | Controls upgradeability for Mailbox + warp router. |
| `defaultIsm` (core, trustedRelayerIsm) | `0x744743c5268e1D035cE6189f08ab6a6FB5644637` | Mailbox-level default ISM. Trusts 0x222…80. |
| `defaultHook` (merkleTreeHook) | `0x5bE786f3cA66169f840F7CF6703f7D36a97D8E00` | Inserts dispatched messages into a Merkle tree. |
| `requiredHook` (protocolFee) | `0x176B044fBD570fBA4Ad19f2061A7249732aB749f` | Charges protocol fee (currently 0). |
| `validatorAnnounce` | `0x2323D357E708496fC43923a2D4288589f99291b8` | Validator checkpoint storage announcements. |
| `interchainAccountRouter` | `0x0FFfA2B2172B777788d4A5A42146E13Abc380366` | Cross-chain account calls. |
| `commitmentIsm` | `0xed316db9D759bF4baC024d29C1d61B99d179297e` | ICA offchain commitment lookup. |
| `testRecipient` | `0x9E3919831795342a49Cc9ddA2380921ff5c6A245` | Minimal `IMessageRecipient` for smoke tests. |
| ISM factories | `0x62ac…963C`, `0x294862…AFEc1`, `0x5B58…2A6`, `0x22ff…76CF`, `0x8Ebe…6854`, `0xEA8A…7Cd7`, `0x3645…fc67` | Lazy deployers for various ISM types. |
| Hook factories | `0x1e75…8CE6` | Lazy deployers for aggregation hooks. |
| `quotedCalls` | `0xA14e60…4165` | Batched on-chain quote helper. |
| **`warp.HypNative`** | **`0x8d9742340a4E722A504a63ED84ef567a7cb41cDD`** | **WETH bridge: locks WETH on send, releases on receive.** |
| **`warp.trustedRelayerIsm`** | **`0x73df77965401EfFf6E90d0277d09ea7Fbe6F968b`** | **ISM for warp route — trusts 0x222…80.** |

---

## Deployed contracts (Base Sepolia)

We did NOT deploy a Mailbox — we reuse the canonical Hyperlane Mailbox that Abacus Works deployed. We only deployed the warp router + its ISM.

Owner: `0x23812ff0cDdd7157C4760E3BB2d39f5f323a7D3c`

| Contract | Address | Role |
|---|---|---|
| `mailbox` (canonical, not ours) | `0x6966b0E55883d49BFB24539356a2f8A673E02039` | Canonical Hyperlane Mailbox on Base Sepolia. |
| `proxyAdmin` (ours, fresh) | `0xA0bb85EBFb4Bc952502C40C9A810155deE8940D1` | Owns the warp router proxy's upgradeability. |
| **`warp.HypERC20`** | **`0x37884726b773297d9a9F1d644580F959194452BF`** | **WETH synthetic: mints on receive, burns on send.** |
| **`warp.trustedRelayerIsm`** | **`0xEd5AA25e5BF4733Aafd898Ba70ba64ffCcE75Ed7`** | **ISM for warp route — trusts 0x238…3c.** |

---

## Dispatch flow (Lark → Base Sepolia)

User wraps WETH on Lark, receives synthetic WETH on Base Sepolia.

```
                       LARK (origin)
                            │
   1. user calls            │
      warpRouter.transferRemote(
        destDomain=84532,
        recipient=0x238…3c,
        amount=0.0001 WETH)
                            ▼
                ┌──────────────────────┐
                │   HypNative warp     │
                │   router (Lark)      │
                │   0x8d97…1cDD        │
                │                      │
                │  • takes msg.value   │
                │    = 0.0001 WETH     │
                │  • holds it in       │
                │    contract          │
                │  • calls Mailbox     │
                │    .dispatch()       │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │       Mailbox        │
                │     0xa4A4…016c       │
                │                      │
                │  • emits Dispatch    │ ◀── 2. log observed
                │    event             │      by relayer
                │  • increments nonce  │
                │  • runs hooks:       │
                └──────────┬───────────┘
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
        ┌─────────────┐       ┌──────────────┐
        │ merkleTree  │       │ protocolFee  │
        │   Hook      │       │     Hook     │
        │             │       │              │
        │ insert msg  │       │   fee = 0    │
        │ in tree     │       └──────────────┘
        └─────────────┘
```

The message body looks like:
```
recipient: 0x238…3c   (20 bytes, padded to 32)
amount:    100000000000000   (0.0001 WETH in wei)
```

---

## Off-chain relayer

```
        Lark RPC                                      Base Sepolia RPC
       (node3.lark.hydration.cloud)                  (sepolia.base.org)
            ▲                                                ▲
            │ eth_getLogs (Mailbox.Dispatch)                 │ eth_getLogs
            │                                                │
            │                                                │
   ┌────────┴────────────────────────────────────────────────┴────────────┐
   │                                                                       │
   │                    Hyperlane Relayer (Docker)                          │
   │                    ghcr.io/hyperlane-xyz/hyperlane-agent:agents-v2.2.0 │
   │                                                                       │
   │   ┌──────────────────────────────────────────────────────────────┐   │
   │   │  MessageSync:   index Mailbox.Dispatch events on both chains  │   │
   │   │  Submitter:     build delivery, submit Mailbox.process()      │   │
   │   │  Signer:        per-chain HexKey                              │   │
   │   │                   lark → 0x42d8…be14 (= addr 0x222…80)        │   │
   │   │                   base → 0x5321…53e8 (= addr 0x238…3c)        │   │
   │   └──────────────────────────────────────────────────────────────┘   │
   │                                                                       │
   └───────────────────────────────────────────────────────────────────────┘
            │                                                ▲
            │ Mailbox.process() with                          │ confirmation
            │ (metadata, message)                              │
            ▼                                                ▲
       (delivery to destination chain)                       │
                                                              │
                                                              └── new
                                                                  HypERC20 balance
```

---

## Process flow (Base Sepolia side)

The relayer submits `Mailbox.process(metadata, message)` on Base Sepolia.

```
                BASE SEPOLIA (destination)
                            │
   1. relayer submits       │
      Mailbox.process(...)  │
                            ▼
              ┌─────────────────────────┐
              │   Mailbox (canonical)   │
              │   0x6966…2039           │
              │                         │
              │  • decode message       │
              │  • call recipient.      │
              │      interchain         │
              │      SecurityModule()   │ ──── 2. router returns
              └────────────┬────────────┘        its ISM:
                           │                     0xEd5A…5Ed7
                           ▼
              ┌─────────────────────────┐
              │  trustedRelayerIsm      │
              │   0xEd5A…5Ed7           │
              │                         │
              │  verify() checks        │
              │  msg.sender == 0x238…3c │ ✓ (relayer signed)
              │  → returns true         │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │   Mailbox (canonical)   │
              │                         │
              │  • records processed[id]│
              │  • call                 │
              │      recipient.handle() │
              └────────────┬────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │  HypERC20 warp router   │
              │   0x37884726…52bF       │
              │                         │
              │  decode body:           │
              │    recipient,amount     │
              │  _mint(recipient,amount)│
              │                         │
              │  → synthetic WETH       │
              │    appears on user's    │
              │    balance              │
              └─────────────────────────┘
```

---

## Ownership map

```
                            Lark                                  Base Sepolia
                             │                                          │
              0x222222ff7…9D80                              0x23812ff0c…7D3c
                             │                                          │
              ┌──────────────┼──────────────┐         ┌─────────────────┼──────────────┐
              ▼              ▼              ▼         ▼                 ▼              ▼
         ┌─────────┐    ┌─────────┐  ┌──────────┐  ┌──────────┐   ┌──────────┐  ┌──────────┐
         │ Mailbox │    │ Proxy   │  │ HypNative│  │ HypERC20 │   │ Proxy    │  │ trusted  │
         │ owner   │    │ Admin   │  │ warp     │  │ warp     │   │ Admin    │  │ Relayer  │
         │         │    │ owner   │  │ owner    │  │ owner    │   │ owner    │  │ Ism      │
         │ can     │    │         │  │          │  │          │   │          │  │ (no      │
         │ change  │    │ can     │  │ can      │  │ can      │   │ can      │  │ owner)   │
         │ ISM /   │    │ upgrade │  │ change   │  │ change   │   │ upgrade  │  │          │
         │ hooks   │    │ Mailbox │  │ ISM,     │  │ ISM,     │   │ warp     │  │          │
         │         │    │ proxy   │  │ enroll   │  │ enroll   │   │ proxy    │  │          │
         │         │    │         │  │ routers  │  │ routers  │   │          │  │          │
         └─────────┘    └─────────┘  └──────────┘  └──────────┘   └──────────┘  └──────────┘
```

Single EOA owns everything per chain. Production hardening = transfer all `owner` fields to a Gnosis Safe (PRODUCTION.md phase 4d).

---

## Domain ID summary

Hyperlane routes messages by **domainId**, distinct from EVM **chainId**.

| Chain | chainId | domainId | Source |
|---|---|---|---|
| Lark | 222222 | 222222 | Local registry override |
| Hydration mainnet (future) | 222222 | needs distinct value (e.g. `1222222`) | Will be added to repo when deployed |
| Base Sepolia | 84532 | 84532 | Canonical Hyperlane registry |

---

## Where files live

```
~/.hyperlane/                                   ← CLI's registry root (default)
├── chains/
│   ├── lark/
│   │   ├── metadata.yaml                       ← copy of repo entry
│   │   └── addresses.yaml                      ← core deploy result
│   └── basesepolia/
│       └── metadata.yaml                       ← override with API key + better RPCs
├── deployments/warp_routes/WETH/
│   ├── basesepolia-deploy.yaml                 ← warp deploy config
│   └── basesepolia-addresses.yaml              ← warp deploy result
└── strategies/
    └── default-strategy.yaml                   ← per-chain signer keys

/Users/nemanja/repos/hyperlane-deploy/          ← this git repo (source of truth)
├── chains/lark/
│   ├── metadata.yaml
│   └── addresses.yaml
├── configs/
│   ├── core-config.yaml                        ← input to `hyperlane core deploy`
│   ├── strategy.yaml                           ← (gitignored — contains keys)
│   ├── agent-config.json                       ← input to relayer
│   └── warp-routes/
├── hyperlane_db_relayer/                       ← relayer DB (gitignored)
├── README.md                                   ← deploy runbook
├── PRODUCTION.md                               ← production hardening roadmap
└── ARCHITECTURE.md                             ← this file
```

---

## Send a token transfer end-to-end

```bash
# In one terminal: start the relayer
docker run -it --rm \
  -e CONFIG_FILES=/config/agent-config.json \
  --mount type=bind,source=$(pwd)/configs/agent-config.json,target=/config/agent-config.json,readonly \
  --mount type=bind,source=$(pwd)/hyperlane_db_relayer,target=/hyperlane_db \
  ghcr.io/hyperlane-xyz/hyperlane-agent:agents-v2.2.0 \
  ./relayer \
  --db /hyperlane_db \
  --relayChains lark,basesepolia \
  --allowLocalCheckpointSyncers true \
  --chains.lark.signer.type hexKey --chains.lark.signer.key 0x42d8...be14 \
  --chains.basesepolia.signer.type hexKey --chains.basesepolia.signer.key 0x5321...53e8

# In another terminal: dispatch a message
hyperlane warp send \
  --warp-route-id WETH/basesepolia \
  --origin lark \
  --destination basesepolia \
  --recipient 0x23812ff0cDdd7157C4760E3BB2d39f5f323a7D3c \
  --amount 100000000000000

# Verify on Base Sepolia
cast call 0x37884726b773297d9a9F1d644580F959194452BF \
  "balanceOf(address)(uint256)" \
  0x23812ff0cDdd7157C4760E3BB2d39f5f323a7D3c \
  --rpc-url https://sepolia.base.org
```

Watch the relayer logs:
```
INFO  Indexed Dispatch event on lark, message_id=0x5fee...
INFO  Building metadata for ISM 0xEd5A…5Ed7 (trustedRelayerIsm)
INFO  Submitted Mailbox.process() on basesepolia, tx=0x...
INFO  Message 0x5fee... delivered
```

---

## What's next

| Goal | Path |
|---|---|
| Run validators (for messageIdMultisigIsm in production) | PRODUCTION.md phase 4a |
| Switch ISM trustedRelayerIsm → messageIdMultisigIsm | PRODUCTION.md phase 4c |
| Transfer ownership to a Gnosis Safe | PRODUCTION.md phase 4d |
| Deploy this on Hydration mainnet | Repeat the Lark flow with mainnet metadata + the friend's funded EOA |
| Submit to canonical Hyperlane registry | PRODUCTION.md phase 5 (mainnet only — testnets like Lark are typically self-hosted) |
