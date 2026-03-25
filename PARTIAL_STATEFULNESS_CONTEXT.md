# Partial Statefulness — Comprehensive Reference Document

> This document captures all context gathered for the EthCC 2026 presentation on Partial Statefulness.
> Authors: CPerezz, @DanGoron2
> Presentation: 18 min + 2 min Q&A

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [What Is Partial Statefulness](#2-what-is-partial-statefulness)
3. [How It Works — Technical Details](#3-how-it-works--technical-details)
4. [VOPS (Validity-Only Partial Statelessness)](#4-vops-validity-only-partial-statelessness)
5. [What PS Enables](#5-what-ps-enables)
6. [PS as Alternative to State Expiry](#6-ps-as-alternative-to-state-expiry)
7. [The "Inseparable Graph" Critique](#7-the-inseparable-graph-critique)
8. [AA vs Statelessness Tension](#8-aa-vs-statelessness-tension)
9. [AI Agents and State Growth](#9-ai-agents-and-state-growth)
10. [Security Model](#10-security-model)
11. [Implementation Status](#11-implementation-status)
12. [Ethereum Roadmap Context](#12-ethereum-roadmap-context)
13. [Presentation Structure](#13-presentation-structure)
14. [Key References](#14-key-references)

---

## 1. The Problem

### ZKEVMs Decouple Verification from State

Today, every Ethereum validator must hold the full state (~280 GB) to re-execute transactions and verify blocks. ZKEVMs change this fundamentally: a SNARK proof replaces re-execution. Validators can verify block correctness in milliseconds without holding ANY state.

### The Centralization Trap

If validators don't need state, rational actors will aggressively prune it. State concentrates in the hands of:
- **Block builders** (need state to build blocks)
- **RPC providers** (Infura, Alchemy — need state to serve queries)

This is dangerous because:
- Ethereum's state is currently highly replicated across ~10,000+ validators worldwide
- Geographic distribution makes censorship extremely difficult (you'd need to target the entire globe)
- With ZKEVMs, this replication disappears — state becomes centralized

### The Mempool Problem

Without state, nodes can't validate transactions (check nonces, balances). This breaks:
- **Mempool health** — nodes can't filter invalid txs
- **Censorship resistance** — FOCIL inclusion lists require knowing which txs are valid
- **State serving** — nobody is incentivized to serve state to others

### State Expiry Is Not the Answer

In-protocol state expiry (forcibly expiring cold accounts) has critical problems:
- **UX nightmare**: Users who haven't transacted in 6 months find their accounts "expired"
- **Protocol complexity skyrockets**: Address space extension, resurrection mechanics, etc.
- **Doesn't actually solve the problem**: Wallets still need access to expired state to bundle revival + transaction. The expired state is still concentrated in the same actors (builders, RPC providers)
- **The state remains centralized regardless**: Same machines hold it, just with extra protocol complexity

PS is a better alternative: let the market naturally stratify state. Hot state gets replicated by many PS nodes; cold state concentrates in specialized operators who charge more for it.

---

## 2. What Is Partial Statefulness

### Core Concept

A partial stateful node holds:
1. **Full account trie** — every account's balance, nonce, codeHash, storageRoot + all intermediate MPT nodes from leaves to root (~50-80 GB for 300M+ accounts)
2. **Selective contract storage** — full storage tries (including intermediates) for chosen contracts (e.g., USDC, DAI, WETH, Uniswap pools)
3. **No re-execution** — follows chain tip via Block Access Lists (BALs), applying state diffs directly

### What This Gives You

- **MPT Proof generation** for any account AND for tracked contracts' storage slots
- **Snap sync serving** — help other nodes bootstrap
- **RPC serving** — answer balance, nonce, storage queries with proofs for tracked state
- **Mempool management** — validate transactions that touch tracked state
- **Inclusion list participation** — become a FOCIL includer

### The Proof Path

For a tracked contract's storage slot, the full proof path is:

```
storage slot → storage trie intermediates → storage root → account leaf → account trie intermediates → state root
```

A PS node can produce this full proof because it holds BOTH the contract's full storage trie AND the full account trie.

For untracked contracts: only the storage root hash exists in the account leaf. No storage trie nodes. Cannot produce storage proofs.

### Proof Batching

Multiple PS nodes can serve different pieces of a complex query. Proofs can be batched as long as they share the same state root. A load balancer / marketplace could:
1. Route storage queries to the PS node that tracks that contract
2. Collect proofs from multiple PS nodes
3. Batch them together (all proving against the same root)
4. Return the combined proof to the requester

---

## 3. How It Works — Technical Details

### Sync Process

**Phase 1: Snap Sync (account trie)**
- Query peers for all account ranges
- Download all accounts + intermediate MPT nodes
- Skip storage tries and bytecode for untracked contracts (using skip markers)

**Phase 2: Snap Sync (tracked contract storage)**
- Download full storage tries for configured contracts
- Download bytecode for tracked contracts

**Phase 3: Gap Fill (pivot → HEAD)**
- Process blocks from snap sync pivot to chain tip using BALs
- Apply state diffs directly — no transaction re-execution needed

**Phase 4: Following the Tip**
- Process each new block's BAL
- Apply state diffs to account trie and tracked contract storage tries
- For untracked contracts: BAL gives you the leaf changes but NOT the contract storage root
  - Workaround: snap-query peers for updated storage roots
  - Wait 1-2 seconds and retry if no peer has updated yet
  - Can be improved if BALs are extended to include intermediate contract storage roots

### Block Access Lists (BALs) — EIP-7928

BALs record ALL accounts and storage locations accessed during block execution, plus post-execution values.

**Key guarantees:**
- **Completeness**: Missing or spurious entries invalidate the block. Enforced by consensus.
- **Deterministic encoding**: RLP with strict ordering → exact hash matching
- **Header commitment**: `block_access_list_hash` in block header must match `keccak256(rlp.encode(actual_bal))`
- **Fork-choice enforced**: Invalid BAL = invalid block

BAL structure per account:
- Address, storage changes (slot → new_value), storage reads, balance changes, nonce changes, code changes
- Sorted lexicographically by address
- Includes block_access_index for ordering

**BALs landing in Glamsterdam (2026).**

### MPT Topology

Each contract's storage is an independent subtrie hanging off the account leaf's storageRoot. You can skip entire subtries without breaking MPT integrity. The account trie stays complete; untracked contracts keep only their storageRoot hash in the account leaf.

---

## 4. VOPS (Validity-Only Partial Statelessness)

### What VOPS Is

The minimal form of partial statefulness. A VOPS node stores only:
- **4 fields per account** (~40 bytes): address (20B), nonce (8B), balance (12B), codeFlag (1 bit)
- **Total**: ~8.4 GB for ~241M accounts (uncompressed)
- **No intermediate trie nodes** — cannot generate MPT proofs
- **No contract storage at all**

### What VOPS Enables

1. **Mempool management** — validate EOA transactions (check nonce, balance, gas)
2. **FOCIL inclusion lists** — become an includer with minimal hardware
3. **SNARK verification** — verify ZK proofs in milliseconds (post-ZKEVM)
4. **codeFlag handling** (EIP-7702): accounts with delegation (codeFlag=1) limited to 1 pending tx because delegated code can unpredictably change nonce/balance

### VOPS vs Full PS

- Any PS node can also be a VOPS node (VOPS is a subset)
- VOPS = "validator floor" — minimal to participate in consensus + inclusion lists
- Full PS = "RPC-capable" — can serve proofs and state for tracked contracts
- Running PS gives you VOPS capabilities + the ability to serve RPC and earn from state markets

### VOPS Security (Post-ZKEVM)

- Verify SNARK binding transactions + state diff to pre→post state root transition
- Extract modified account data from verified stateDiff
- Update local table
- Enforce inclusion list rules locally
- Resource profile: ~8.4 GB disk, SNARK verification per block (milliseconds), minimal bandwidth

---

## 5. What PS Enables

### 5.1 Inclusion Lists (FOCIL — EIP-7805)

**How FOCIL works:**
- 16 validators per slot selected as IL committee members
- Each includer selects transactions from the mempool
- ILs can contain any txs (valid or invalid) — no EL-side validity checks required
- Attesters enforce: builders MUST include IL txs unless they're genuinely invalid or block is full
- No slashing for invalid IL txs — just wasted space. Only equivocation (conflicting ILs) gets you ignored.
- System assumes 1-of-N honesty. No incentive mechanism — altruistic.

**Why PS matters for FOCIL:**
- Account data makes includers EFFECTIVE, not just eligible
- PS node can verify tx validity (good nonce, sufficient balance) → picks valid txs → harder for builders to justify exclusion
- Without PS, in post-ZKEVM world, only full nodes (builders + RPC providers) can be effective includers → censorship resistance degrades

**The centralization argument:**
FOCIL doesn't matter if nobody serves you state to build transactions. If state is centralized in a few providers, censorship becomes: just don't serve state to the target address. PS distributes state → distributes censorship resistance.

### 5.2 State Markets

**Emergent, not protocol-level.** Out-of-protocol allows different flavours:
- Entities act as middlemen / load balancers
- Route queries to PS nodes that track the relevant contracts
- Different policies: pricing, decentralization level, latency, geographic distribution
- PS nodes register which contracts they track
- RPC users get routed to the right node

**Revenue model:**
- Primary: serving RPC queries for tracked contracts (paid per query)
- Hot/recent state → lower price (more nodes have it)
- Cold state → higher price (fewer nodes, more valuable)
- Node operators choose their strategy: cover hot state for volume, or cold state for premium pricing

**The attestation analogy:**
Like attestations: one query is worth almost nothing. But millions of queries over a year = meaningful revenue. The economics work at scale, not per-event.

**Flywheel:**
Lower cost → more operators → geographic distribution → better latency/redundancy → more queries → more revenue → attracts more operators

### 5.3 Decentralized State Replication

**The core value proposition:**
- Today: state replicated across ~10K+ validators worldwide
- Post-ZKEVM without PS: state concentrated in ~10-50 major operators
- Post-ZKEVM with PS: state distributed across thousands of PS nodes, each holding their chosen slice
- Natural redundancy gradient: popular contracts (USDC, WETH) replicated by thousands; obscure ones by a few

### 5.4 Alternative to Portal Network

PS addresses Portal's biggest issue: **revenue/motivation to hold state**. Portal lacks economic incentives for operators. PS creates natural incentives through state markets.

Open question: could an intermediate solution combine Portal's p2p infrastructure with PS's economic model? Micropayments or payment channels (2017/2018 ideas: state channels, Plasma-like) could bridge the gap, but no concrete proposal exists yet.

---

## 6. PS as Alternative to State Expiry

### Why State Expiry Is Hard

- Protocol complexity explodes (address space extension, resurrection proofs, etc.)
- UX nightmare: accounts "expire" after inactivity
- Wallets need expired state anyway (to bundle revival + transaction)
- State still concentrates in same actors — just with extra protocol complexity
- Doesn't actually reduce who needs to hold state

### How PS Addresses This

- No protocol changes needed (beyond BALs, which have independent value)
- Hot state naturally replicated by many PS nodes
- Cold state held by specialized operators (builders, archival services)
- Pricing incentives: cold state costs more to query → operators are compensated
- Users never experience "account expired" — their state is just held by fewer nodes

### Scaling Concern

PS's account trie size scales with number of accounts:
- If accounts grow linearly → PS scales well, buys significant time
- If accounts grow exponentially (AI agents scenario) → account trie could become 40%+ of total state
- Unknown: nobody knows the growth curve. AI agents skew toward more accounts.
- PS may fully replace state expiry, or it may buy substantial time (years/decades)

---

## 7. The "Inseparable Graph" Critique

### The Argument (from "Future of State Part 2")

If you try to serve "just USDC," users immediately need:
- DEX pool data (pricing)
- Router contracts (trade execution)
- Vault wrappers, lending derivatives, LP positions
- Each dependency pulls in more contracts

"The closure is 'most of DeFi + witness paths.'" Fragmentation either leaves nodes useless or forces them to hold most of the economic state.

### When This Critique IS Valid

- Large DeFi aggregators (CoW Swap, 1inch) that need to route across all possible paths
- MEV searchers / builders who need complete state to find opportunities
- Complex multi-hop trades requiring full DeFi graph

### When This Critique Is NOT Valid

- **VOPS nodes**: only need account data, no contract storage at all
- **Privacy protocols** (e.g., privacy pools): self-contained, don't pull in DeFi graph
- **ENS**: mostly self-contained (resolver + registry)
- **Specialized DeFi nodes**: track USDC + all its pools → serve one piece of a trade. Other PS nodes serve other pieces. Proofs are batched.
- **Simple balance queries**: "What's this address's USDC balance?" — answerable with proof, even without swap routing data
- **Inclusion list includers**: only need account data to validate tx validity

### The Key Rebuttal

PS nodes don't need to serve EVERYTHING to be useful. A node tracking USDC answers "what's this USDC balance?" with a cryptographic proof. It can't answer "what's the best swap route?" — but that's a different service. Different PS nodes serve different pieces. The system works through specialization, not universality.

---

## 8. AA vs Statelessness Tension

### The Fundamental Conflict

- **Statelessness/PS**: wants minimal state for validation. Ideal: signature check + nonce/balance → done.
- **AA (Frame Transactions)**: wants programmable validation with arbitrary state access. Validation logic lives in smart contracts → needs to execute EVM code → needs state.

### Impact on Mempool

If most nodes are partial stateful and can't validate AA txs that touch untracked state:
- Only full nodes can maintain mempool for arbitrary AA txs
- Mempool propagation degrades for AA txs
- PS nodes can still manage mempool for: EOA txs, EIP-7702 delegated txs (conservatively, 1 pending tx), and AA txs whose validation only touches tracked state

### AA Proposals and PS Compatibility

| Proposal | Validation | State Needed | PS Compatible? |
|----------|-----------|--------------|----------------|
| Regular EOA (ECDSA) | Stateless (signature) | Nonce + balance only | Fully compatible |
| EIP-7702 (delegation) | Stateless sig + conservative mempool | Nonce + balance + codeFlag | Compatible (1 pending tx limit) |
| Schemed Transactions | Stateless (scheme_id + signature) | Nonce + balance only | Fully compatible |
| Frame Transactions (EIP-8141) | EVM execution | Arbitrary contract state | Problematic for PS |
| EIP-7701 (native AA) | EVM execution | Validation contract state | Problematic for PS |

### Key Insight

The AA scheme that requires LESS "universal state" for mempool management is better for PS. Schemed Transactions are the most PS-friendly AA approach. Frame Transactions are the least.

### Open Research Question

Can both coexist? Possible paths:
- Restricted validation scope (limit what validation code can access)
- Witness-carrying transactions (tx carries its own state proof)
- Stratified mempool: PS nodes handle simple txs, full nodes handle complex AA txs
- "Conservative then expand": start with restricted mempool rules, expand over time

### The Validation Spectrum (for presentation)

Frame as a gradient from fully stateless to fully stateful validation:

| Proposal | Validation State | PS Compatible? | Trade-off |
|----------|-----------------|----------------|-----------|
| **Schemed Tx (EIP-8130)** | Zero — pure crypto | Fully | No programmable validation, no crypto-agility without hard forks |
| **EIP-7702** | Account data only | Fully (VOPS sufficient) | 1 pending tx per delegated account |
| **ERC-4337** | Alt-mempool | Compatible by design separation | CR depends on bundler decentralization |
| **Frame Tx (EIP-8141)** | Arbitrary EVM in VERIFY frames | Problematic | Requires witnesses or selective caching |

### The Frame Transaction Problem

Frame Txs enter the PUBLIC mempool (not alt-mempool like 4337). FOCIL includers must validate them. If includers are PS nodes that can't execute VERIFY frames → can't include smart wallet txs in ILs → censorship resistance degrades for AA transactions.

**Three mitigation paths:**
1. **Witness-carrying transactions** — txs carry Merkle/Verkle proofs (~4KB per external storage slot, ~12-20KB for typical smart wallet validation). PS nodes verify witness against state root, then execute VERIFY locally. Emerging consensus approach.
2. **AA-VOPS** — FOCIL includers opt into caching popular smart wallet factory contracts. Witnesses bootstrap from Portal. BALs keep cache current.
3. **Accept bifurcation** — PS nodes handle EOA/Schemed txs; full nodes handle Frame Txs. Degrades but doesn't break.

### Gas Cost Comparison

- Falcon-512 via Schemed Tx: ~29,500 gas
- Falcon-512 via Frame Tx + smart wallet: ~63,000-82,880 gas

### The Realistic Path Forward

Ethereum will likely ship BOTH:
- **Schemed Tx** for simple stateless PQ signature validation
- **Frame Tx** for programmable AA, with witness attachment required for non-VOPS state

Layered approach: VOPS data handles common case → witnesses handle AA case → selective caching (AA-VOPS) optimizes for popular wallets.

### Key Presentation Thesis

> "Partial statefulness is not just a storage optimization — it is the enabling layer that determines which forms of account abstraction are viable without centralizing the mempool."

### Research Gap to Flag

The "Protocol Design View on Statelessness" (ethresear.ch #22060) does NOT address AA at all. The AA-statelessness interaction is under-researched at protocol level. Worth flagging as open area.

### Sources
- EIP-8141, EIP-8130, EIP-7702, EIP-7701, ERC-4337, ERC-7562
- ethresear.ch VOPS thread (posts #13-15 by soispoke on AA-VOPS)
- Biconomy Q1/26 Native AA State-of-Art review
- Vitalik on FOCIL + EIP-8141 synergy (Feb 2026)
- ethresear.ch #22060 (Protocol Design View on Statelessness)
- ethresear.ch #16299 (Towards Stateless Account Abstraction)

---

## 9. AI Agents and State Growth

### The Argument

1. AI agents massively increase on-chain activity → more accounts, more state
2. Each agent needs state to build transactions
3. With billions of agents, centralized RPC can't serve them all
4. Decentralized PS node network becomes the natural answer

### Critical Assessment

**The trust argument is the strongest angle — NOT scale.**

AI agents are the first class of Ethereum users that are both autonomous and trust-sensitive. Unlike humans who trust MetaMask + Infura implicitly, agents managing real capital need *verifiable state* — Merkle proofs, not promises. PS nodes can serve proof-carrying responses. This is qualitatively different from centralized RPC.

### Arguments FOR (strong)

1. **Trust & verifiability**: Autonomous agents managing funds MUST verify state. Can't trust centralized RPC any more than DeFi contracts trust off-chain oracles. PS nodes serve Merkle proofs. This is the killer feature.
2. **Domain specialization alignment**: DeFi agent tracks DeFi contracts. NFT agent tracks NFT marketplaces. Maps directly to PS state market vision.
3. **FOCIL/CR angle remains compelling**: Agents need local state for inclusion lists regardless.
4. **Agent-to-agent commerce creates novel state**: x402/Machine Payments Protocol → new accounts, new contract interactions.

### Arguments AGAINST (honest)

1. **"Billions of agents" is 2030+, not near-term**: Current numbers are millions. 162M x402 txs total (not per day). Scale argument is premature.
2. **Most agent activity is on L2s**: Base handles ~60% of L2 txs. PS on L1 may not be where agents transact.
3. **Centralized providers CAN scale technically**: 50B RPC calls/day is doable for cloud infra. Bottleneck is cost and trust, not capacity.
4. **Shared infrastructure consolidation**: One operator runs 10,000 agents off one node. "Billions of agents" → "thousands of operators."
5. **Hot state concentration**: Agents mostly access the same popular DeFi contracts → favors centralized caching too.

### Recommended Framing (intellectually honest)

> "AI agents are the first class of Ethereum users that are both autonomous and trust-sensitive. The agent economy doesn't create the need for partial statefulness — the ZKEVM transition does — but agents become partial statefulness's most natural consumer."

Ground in trust model, not speculative scale. Mention numbers but don't oversell timeline.

### Key Numbers

| Metric | Value | Source |
|--------|-------|--------|
| ERC-4337 smart accounts deployed | 40M+ (end 2024), 200M+ projected | Alchemy, Rhinestone |
| x402 machine-to-machine txs | 162M since Oct 2025 | Coinbase |
| Live AI agent projects (Virtuals) | 15,800+ | Virtuals Protocol |
| Ethereum L1 state size | ~1TB+, growing ~100GB/year | Paradigm |
| State composition | 81.7% contract storage, 14.1% accounts | Core Paper |
| L2 tx share (Base) | ~60% of all L2 | The Block |

### Slide Suggestion

One slide: "Who consumes partial state?" → agents as canonical PS consumer, trust argument front and center. Mention numbers, don't oversell timeline.

---

## 10. Security Model

### Pre-ZKEVM (Current)

Light-client-like strategy with extra state root verification:

1. **Sync committee signature validation** — verify the signing committee (beacon chain sync committee) signature. Trust assumption: honest majority of sync committee.
2. **State root recomputation** — apply BAL diffs to previous state root, recompute new root, verify it matches. This is STRONGER than pure light client because you're actually recomputing the root from diffs, not just trusting the header.
3. **BAL completeness** — guaranteed by consensus (invalid BAL = invalid block, enforced by fork choice)

**Comparison to full node:**
- Full node: re-executes all transactions, computes state root from scratch
- PS node: applies verified diffs, recomputes state root from previous root + changes
- Attack surface is the same (multiple peers, easy to verify roots)
- PS gets extra security from state root recomputation vs pure light client

### Post-ZKEVM

- SNARK verification replaces sync committee trust
- Full cryptographic guarantees: proof binds transactions + state diff to pre→post root transition
- PS node verifies SNARK (milliseconds), extracts state diffs, updates local state
- Security equivalent to full node verification

### Contract Storage Root Workaround

BALs don't include storage roots for untracked contracts. Workaround:
- When BAL shows changes to an untracked contract's storage, snap-query peers for the updated storage root
- Wait 1-2 seconds and retry if no peer has updated yet
- Can be improved by extending BALs to include intermediate contract storage roots
- Once obtained, recompute state root to verify consistency

---

## 11. Implementation Status

### go-ethereum PR #33764

- **Status**: Working implementation, tested on mainnet
- **Size**: ~59 GB for full account trie + USDC + DAI storage (mainnet). Breakdown: ~50GB account trie (incl. intermediates) + ~9GB USDC/DAI storage tries
- **Reduction**: ~79% reduction vs full state sync (~280 GB)
- **Minimum viable PS node**: Expected ~70-80 GB for a useful configuration
- **Only geth** — no other clients (Nethermind, Besu, Erigon, Reth) have implementations yet

### Configuration

```
--partial-state                    # Enable partial statefulness
--partial-state.contracts          # Which contracts to track
--partial-state.chain-retention    # Block retention window (default 1024, min 256)
```

### Key Implementation Details

- Modified snap sync with contract filtering (skip markers for untracked contracts)
- BAL processing for state updates without re-execution
- New RPC error codes: `StorageNotTrackedError (-32001)`, `CodeNotTrackedError (-32002)`
- Account queries (balance, nonce, proofs) work for ALL accounts
- 30-second peer cooldown (not permanent ban) for empty responses in partial mode
- Snapshot generator disabled for untracked contracts (prevents goroutine leak)
- 2-minute cooldown for pivot advances during sync

---

## 12. Ethereum Roadmap Context

| Fork | Target | Relevant Features |
|------|--------|-------------------|
| **Glamsterdam** | 2026 | BALs (EIP-7928) — enables PS to follow chain tip |
| **Hegota** | 2026 | AA (Frame Transactions or Schemed Transactions) |
| **Next 1-2 forks** | ~2027+ | ZKEVMs — makes PS/VOPS essential for decentralization |

- BALs are the prerequisite for PS — landing in Glamsterdam makes PS possible
- AA choice (Frame vs Schemed) directly impacts PS compatibility
- ZKEVMs are the catalyst — once live, validators no longer need state, PS becomes the answer to state centralization

---

## 13. Presentation Structure (Proposed — 18 min)

### Act 1: The Problem (3 min)
- ZKEVMs decouple verification from state → validators drop state → centralization trap
- State replication disappears: from ~10K validators to ~10-50 operators
- Censorship risk: FOCIL doesn't matter if nobody serves you state to build txs
- State expiry is not the answer (UX nightmare, protocol complexity, doesn't actually decentralize state — same actors hold it anyway)
- Concrete censorship scenario: e.g., government pressures Infura to stop serving state for Tornado Cash addresses → users can't build txs → FOCIL includers never see those txs → invisible censorship without touching the protocol

### Act 2: Partial Statefulness — How It Works (1.5 min)
- Account trie + selective storage + BALs → no re-execution
- MPT proof generation: full path from storage slot → state root
- Visual: the MPT proof path diagram
- Key number: ~59GB on mainnet (account trie + USDC + DAI) vs ~640GB full node

### Act 3: What PS Enables (8 min) ← MAIN FOCUS
- **Inclusion Lists / FOCIL** (2 min)
  - PS nodes as effective includers (validate txs → pick valid ones → harder for builders to exclude)
  - Hardware drops from ~640GB to ~50-80GB → dramatically expands includer pool
  - Without PS in post-ZKEVM: only builders + RPC providers can be includers → CR degrades
  - 16 includers per slot, 1-of-N honesty assumption
- **State Markets** (2 min)
  - Emergent, out-of-protocol → many flavours of decentralization
  - Load balancer / marketplace model: PS nodes register tracked contracts, queries get routed
  - Hot/cold pricing: hot state = cheap (many holders), cold state = premium (few holders)
  - Attestation analogy: one query ≈ nothing, millions/year ≈ real revenue
  - Flywheel: lower cost → more operators → geo distribution → better latency → more queries
- **Decentralized State Replication** (1.5 min)
  - Natural redundancy gradient: USDC replicated by thousands, obscure contracts by a few
  - PS as alternative to state expiry: no protocol changes, market handles hot/cold stratification
  - Scaling concern: account trie grows with accounts (AI agents may skew this)
- **VOPS** (1 min)
  - Validator floor: ~8.4GB, enables staking in post-ZKEVM world
  - Any PS node is also a VOPS node (subset)
  - Enables: mempool management + FOCIL inclusion + SNARK verification
- **Who Consumes Partial State?** (1.5 min)
  - AI agents as canonical PS consumer → trust argument (verifiable state, not promises)
  - Domain specialization alignment: DeFi agent ↔ DeFi PS node
  - Honest: "billions of agents" is 2030+. The ZKEVM transition creates the need; agents are the natural consumer.

### Act 4: Interactions & Open Questions (4 min)
- **AA tension / "The Validation Spectrum"** (2 min)
  - Schemed Tx = free for PS. Frame Tx = needs witnesses or caching.
  - The core tension: statelessness wants minimal state for validation, AA wants programmable validation with state access
  - Witness-carrying txs as bridge: ~12-20KB per smart wallet tx
  - Key insight: PS determines which forms of AA are viable without centralizing the mempool
  - Under-researched at protocol level — flag as open area
- **"Inseparable graph" critique** (1 min)
  - When valid: large aggregators, MEV searchers (need full DeFi graph)
  - When NOT valid: VOPS, ENS, privacy protocols, specialized nodes serving one piece of a trade
  - Key rebuttal: PS nodes don't need to serve everything. Specialization + proof batching.
- **Portal Network comparison** (1 min)
  - PS addresses Portal's biggest issue: revenue/motivation
  - Open question: intermediate solution combining Portal p2p + PS economics?
  - Micropayments / payment channels needed but no concrete proposals yet

### Act 5: Implementation & CTA (1.5 min)
- Working in geth PR #33764, tested on mainnet
- ~59GB, ~79% reduction
- Glamsterdam (2026) enables BALs → PS becomes possible
- Only geth for now — opportunity for other clients
- Screenshot of running PS node
- CTA: Try it after Glamsterdam, provide feedback
- Credit: @DanGoron2

---

## 14. Key References

- **VOPS ethresear.ch post**: https://ethresear.ch/t/a-pragmatic-path-towards-validity-only-partial-statelessness-vops/22236
- **Future of State Part 2**: https://ethresear.ch/t/the-future-of-state-part-2-beyond-the-myth-of-partial-statefulness-the-reality-of-zkevms/23396
- **Geth PR #33764**: https://github.com/ethereum/go-ethereum/pull/33764
- **EIP-7928 (BALs)**: https://eips.ethereum.org/EIPS/eip-7928
- **EIP-7805 (FOCIL)**: https://eips.ethereum.org/EIPS/eip-7805
- **Frame Transactions**: https://ethereum-magicians.org/t/hegota-headliner-proposal-frame-transaction/27618
- **Frame vs Schemed**: https://ethereum-magicians.org/t/frame-transactions-vs-schemedtransactions-for-post-quantum-ethereum/28056
- **Existing presentation**: `partial-statefulness-presentation.html` in this repo
