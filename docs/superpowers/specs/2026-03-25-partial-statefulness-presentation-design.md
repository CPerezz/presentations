# Partial Statefulness Presentation — Design Spec

> EthCC 2026 | 18 min + 2 min Q&A | CPerezz & @DanGoron2

## Overview

A 22-slide HTML presentation explaining Partial Statefulness for Ethereum — why it's needed, how it works, and what it enables. Narrative arc: "The Crisis" — open with urgency about state centralization, reveal the solution, focus heavily on what PS enables. Close with a practical CTA.

Target audience: staking operators, core devs, general Ethereum community.

---

## Visual Style: CRT Hybrid Terminal

**Concept:** Retro CRT terminal charm (VT323 headings, scanlines, phosphor glow, blinking cursor) combined with modern readability (IBM Plex Mono body text, clean layout, white + green mixed text).

### Typography

| Role | Font | Weight | Source |
|------|------|--------|--------|
| Headings, numbers, accents, terminal prompts | VT323 | 400 | Google Fonts |
| Body text, descriptions, labels, captions | IBM Plex Mono | 400, 500, 600, 700 | Google Fonts |

### Colors

```css
:root {
    --bg: #000000;
    --bg-surface: #0a0a0a;
    --border: #1a2a1a;
    --border-bright: #1a3a1a;

    --green: #33ff33;
    --green-dim: #22cc22;
    --green-muted: #1a5a1a;
    --green-glow: rgba(51, 255, 51, 0.3);
    --green-bg: rgba(51, 255, 51, 0.02);
    --green-bg-hover: rgba(51, 255, 51, 0.04);

    --text-primary: #e0e0e0;
    --text-secondary: #b0b0b0;
    --text-muted: #666666;
    --text-dim: #444444;
    --white: #e8e8e8;
}
```

### Signature Elements

- **Subtle CRT scanlines**: `repeating-linear-gradient` every 6px, opacity ~0.008
- **Vignette**: `radial-gradient` darkening edges, 60% transparent center
- **Phosphor glow**: `text-shadow: 0 0 6px rgba(51,255,51,0.3)` on green headings/numbers
- **Blinking cursor**: Block cursor on title slide, `animation: blink 1s step-end infinite`
- **Terminal prompts**: `>` before bullet points in VT323
- **Slide footer**: `$ command --flag` left, `slide N / 22` right (IBM Plex Mono, dim)
- **Section eyebrows**: `// SECTION NAME` in VT323, dim green, letter-spaced
- **Divider lines**: `linear-gradient(90deg, #1a3a1a, #33ff33 30%, #33ff33 70%, #1a3a1a)` at 30% opacity
- **Stat numbers**: VT323 at large size with glow
- **Cards**: 1px `#1a2a1a` border, `rgba(51,255,51,0.015)` background, green top-line accent
- **Code blocks**: `rgba(51,255,51,0.04)` background, syntax-colored (green for commands, white for flags, dim for comments)
- **Image placeholders**: Dashed border in `#1a3a1a`, dim green center text
- **Pull quotes**: 2px `#33ff33` left border, italic IBM Plex Mono, white text with green emphasis words
- **Progress bar**: Thin green line at top of viewport
- **Nav dots**: Right edge, green active dot with glow

### Animations

- Fade + slide up reveals (0.6s, ease-out-expo)
- Staggered child delays (0.1s increments)
- Blinking cursor on title slide
- Subtle glow pulse on key stat numbers
- No bouncy, no flashy — measured and serious

---

## Slide-by-Slide Structure

### Act 1: The Problem (~3.5 min)

#### Slide 1 — Title
- **Type:** Title slide
- **Eyebrow:** `// ETHEREUM RESEARCH`
- **Title (VT323, large):** `Who Holds` (white) `Ethereum's State?` (green) + blinking cursor
- **Divider**
- **Subtitle (Plex):** "Partial Statefulness — Rethinking state in a post-ZKEVM world"
- **Meta (Plex, dim):** CPerezz & @DanGoron2 · EthCC 2026
- **Footer:** `$ cat README.md` | `1 / 22`

#### Slide 2 — The Status Quo
- **Eyebrow:** `// TODAY`
- **Title:** `The` (white) `Status Quo` (green)
- **Bullet list (4 items):**
  - ~13,900 nodes run 1M+ validator keys on the network
  - Re-execution requires every full node to hold ~280 GB of state
  - This creates natural replication — state distributed across thousands of machines
  - Geographic distribution makes censorship nearly impossible
- **Stat row (3 stats):** `13,900` nodes | `1M+` validator keys | `280 GB` state per node
- **Footer:** `$ eth.syncing` | `2 / 22`

#### Slide 3 — The Centralization Trap
- **Eyebrow:** `// THE SHIFT`
- **Title:** `ZKEVMs` (green) `Change Everything` (white)
- **Bullet list (4 items):**
  - A SNARK proof replaces re-execution — verification in milliseconds
  - The mandate disappears — full nodes no longer need state to validate
  - State concentrates in machines with economic reasons to hold it — builders and RPC providers
  - Fewer state-holding nodes → fewer snap sync peers → harder to bootstrap → vicious cycle
- **Pull quote:** "If verification is free, why pay for state?"
- **Footer:** `$ zkvm --verify block.proof` | `3 / 22`

#### Slide 4 — [DIAGRAM] State Collapse
- **Type:** Full-slide diagram
- **Image:** `state-collapse.png`
- **HTML overlay labels (VT323):**
  - Left side: `TODAY: ~13,900 full nodes`
  - Right side: `POST-ZKEVM: builders + RPC providers only`
- **Footer:** `$ network --state-holders` | `4 / 22`
- **Source footnote (Plex, tiny, dim):** none needed — derived from network stats

#### Slide 5 — State Expiry Is Not the Answer
- **Eyebrow:** `// DEAD END`
- **Title:** `State Expiry` (green) `Is Not the Answer` (white)
- **3 feature cards (1x3 row):**
  - Card 1: **UX Nightmare** — Accounts "expire" after 6 months of inactivity
  - Card 2: **Protocol Complexity** — Address space extension, resurrection mechanics, etc.
  - Card 3: **Doesn't Solve It** — Wallets still need expired state from the same centralized providers
- **Pull quote:** "State expiry adds protocol complexity. The state remains centralized regardless."
- **Footer:** `$ state-expiry --status REJECTED` | `5 / 22`

### Act 2: How It Works (~2 min)

#### Slide 6 — What is Partial Statefulness
- **Eyebrow:** `// PARTIAL STATEFULNESS`
- **Title:** `Hold Only` (white) `What You Care About` (green)
- **3 feature cards (1x3 row):**
  - Card 1: **Full Account Trie** — Every account's balance, nonce, codeHash (~50 GB)
  - Card 2: **Selective Storage** — Choose which contracts to track (USDC, DAI, Uniswap...)
  - Card 3: **No Re-execution** — Follow chain tip via Block Access Lists (BALs)
- **Stat row (2 stats):** `59 GB` on mainnet | `79%` reduction vs full state
- **Footer:** `$ geth --partial-state` | `6 / 22`

#### Slide 7 — [DIAGRAM] PS Node Architecture
- **Type:** Full-slide diagram
- **Image:** `ps-node-architecture.png`
- **HTML overlay labels (VT323):**
  - Top section: `ACCOUNT TRIE (~50 GB) — complete`
  - Middle section: `TRACKED: USDC, DAI (~9 GB)`
  - Bottom section: `UNTRACKED — roots only`
  - Ghosted outline: `FULL STATE: ~280 GB`
- **Footer:** `$ du -sh ~/.ethereum/partial/` | `7 / 22`

#### Slide 8 — The Proof Path
- **Eyebrow:** `// ARCHITECTURE`
- **Title:** `Prove` (green) `What You Hold` (white)
- **Body text:** "A PS node produces full Merkle proofs for tracked contracts. The proof path traverses two trie layers:"
- **Code-style path (VT323, green):**
  ```
  storage slot → storage trie → storage root
      → account leaf → account trie → state root
  ```
- **Body text:** "Cryptographic certainty, not trust. Proofs can be batched across PS nodes sharing the same state root."
- **Footer:** `$ eth_getProof --contract 0xA0b8...` | `8 / 22`

#### Slide 9 — [DIAGRAM] MPT Proof Path
- **Type:** Full-slide diagram
- **Image:** `mpt-proof-path.png`
- **HTML overlay labels (VT323):**
  - Bottom: `STORAGE SLOT (leaf)`
  - Middle connection: `STORAGE ROOT → ACCOUNT LEAF`
  - Top: `STATE ROOT`
  - Left side labels: `STORAGE TRIE` (lower), `ACCOUNT TRIE` (upper)
- **Footer:** `$ mpt --trace-path 0xA0b8::slot[3]` | `9 / 22`

#### Slide 10 — The Sync Process
- **Eyebrow:** `// SYNC`
- **Title:** `Sync,` (white) `Don't Execute` (green)
- **4-step vertical flow (numbered, card-like):**
  1. **Snap sync account trie** — Download all accounts + intermediate MPT nodes (~50 GB)
  2. **Snap sync tracked storage** — Download full storage tries for selected contracts
  3. **Gap fill via BALs** — Process Block Access Lists from pivot to HEAD — no re-execution
  4. **Follow the tip** — Apply BAL diffs per block. Serve proofs for tracked state.
- **Body note (small):** "BALs (EIP-7928) record all state changes per block — consensus-enforced completeness. Landing in Glamsterdam 2026."
- **Footer:** `$ geth --syncmode partial` | `10 / 22`
- **Source footnote:** EIP-7928: eips.ethereum.org/EIPS/eip-7928

### Act 3: What PS Enables (~8 min)

#### Slide 11 — FOCIL / Censorship Resistance
- **Eyebrow:** `// CENSORSHIP RESISTANCE`
- **Title:** `Inclusion Lists` (green) `Need State` (white)
- **Bullet list (4 items):**
  - FOCIL (EIP-7805): 16 includers per slot select txs from the mempool
  - Includers need account data to pick VALID transactions
  - Valid txs in ILs = harder for builders to justify exclusion
  - PS drops state from ~280 GB to ~59 GB → dramatically expands the includer pool
- **Pull quote:** "Account data makes includers effective, not just eligible. Without PS, only builders and RPC providers can be includers post-ZKEVM."
- **Footer:** `$ focil --include-from-mempool` | `11 / 22`
- **Source footnote:** EIP-7805: eips.ethereum.org/EIPS/eip-7805

#### Slide 12 — [DIAGRAM] FOCIL Inclusion Flow
- **Type:** Full-slide diagram
- **Image:** `focil-flow.png`
- **HTML overlay labels (VT323):**
  - Stage 1: `MEMPOOL`
  - Stage 2: `PS NODE (validates)`
  - Stage 3: `INCLUSION LIST`
  - Stage 4: `BLOCK BUILDER`
  - Stage 5: `ATTESTERS (enforce)`
- **Footer:** `$ focil --status` | `12 / 22`

#### Slide 13 — State Markets
- **Eyebrow:** `// ECONOMICS`
- **Title:** `State` (green) `Markets` (white)
- **Body intro:** "PS creates a natural market for state serving — emergent, out-of-protocol, many flavours."
- **4 feature cards (2x2 grid):**
  - Card 1: **Load Balancers** — Marketplaces route queries to PS nodes tracking relevant contracts
  - Card 2: **Hot / Cold Pricing** — Hot state = cheap (many holders). Cold state = premium (few holders)
  - Card 3: **Domain Specialization** — DeFi nodes, NFT nodes, protocol-specific nodes. Agents, wallets, indexers as consumers.
  - Card 4: **The Attestation Analogy** — One query ≈ nothing. Millions/year ≈ real revenue. Economics work at scale.
- **Footer:** `$ state-market --list-providers` | `13 / 22`

#### Slide 14 — [DIAGRAM] State Market Flywheel
- **Type:** Full-slide diagram
- **Image:** `state-market-flywheel.png`
- **HTML overlay labels (VT323, positioned around the circle):**
  - Top: `LOWER COST`
  - Upper-right: `MORE OPERATORS`
  - Lower-right: `GEO DISTRIBUTION`
  - Bottom: `BETTER LATENCY`
  - Lower-left: `MORE QUERIES`
  - Upper-left: `MORE REVENUE`
- **Footer:** `$ flywheel --momentum` | `14 / 22`

#### Slide 15 — Decentralized State Replication
- **Eyebrow:** `// THE VALUE`
- **Title:** `Natural` (green) `Redundancy` (white)
- **3-row comparison (before/after style):**
  - Row 1: `Today` → ~13,900 full nodes hold state (mandated by re-execution)
  - Row 2: `Post-ZKEVM without PS` → State held only by builders + RPC providers. Snap sync degrades.
  - Row 3: `Post-ZKEVM with PS` → Thousands of specialized nodes, each holding their slice
- **Body text:** "PS achieves what state expiry promises — through market forces, not protocol mandates. No protocol changes beyond BALs. Users never experience 'account expired.'"
- **Footer:** `$ state --replication-factor` | `15 / 22`

#### Slide 16 — [DIAGRAM] Redundancy Gradient
- **Type:** Full-slide diagram
- **Image:** `redundancy-gradient.png`
- **HTML overlay labels (VT323, below each column):**
  - Column 1: `USDC` (brightest)
  - Column 2: `WETH`
  - Column 3: `UNI/AAVE`
  - Column 4: `ENS`
  - Column 5: `LONG TAIL`
  - Column 6: `COLD`
- **Axis label:** `← MORE REPLICAS` (left) ... `FEWER REPLICAS →` (right)
- **Footer:** `$ state --redundancy-map` | `16 / 22`

#### Slide 17 — VOPS: The Validator Floor
- **Eyebrow:** `// VOPS`
- **Title:** `The Validator` (white) `Floor` (green)
- **Big stat (hero, centered):** `8.4 GB`
- **Body text:** "Validity-Only Partial Statelessness — the minimum to stay useful post-ZKEVM."
- **Bullet list (4 items):**
  - Just 4 fields per account: address (20B), nonce (8B), balance (12B), codeFlag (1 bit)
  - Enables: mempool management + FOCIL inclusion + SNARK verification
  - Any PS node is also a VOPS node (VOPS is a subset)
  - Post-ZKEVM: this is the cost of keeping Ethereum censorship-resistant
- **Footer:** `$ vops --min-requirements` | `17 / 22`

### Act 4: Open Questions (~2 min)

#### Slide 18 — AA vs Statelessness
- **Eyebrow:** `// OPEN QUESTION`
- **Title:** `Account Abstraction` (green) `vs Statelessness` (white)
- **Body text:** "Statelessness wants minimal state for validation. AA wants programmable validation with arbitrary state access. These pull in opposite directions."
- **Key insight (pull quote):** "The more state validation requires — and the more arbitrary it can be — the harder it becomes for nodes to keep the mempool healthy."
- **Body text:** "The AA scheme that makes validation NOT depend on state is the best for statelessness. Schemed Transactions (EIP-8130) are fully PS-compatible. Frame Transactions (EIP-8141) need witness-carrying (~12-20 KB per tx) to work."
- **Footer:** `$ mempool --validate-tx --state-required ?` | `18 / 22`

#### Slide 19 — [DIAGRAM] Validation Spectrum
- **Type:** Full-slide diagram
- **Image:** `validation-spectrum.png`
- **HTML overlay labels (VT323):**
  - Left end: `STATELESS`
  - Right end: `STATEFUL`
  - Marker 1: `SCHEMED TX`
  - Marker 2: `EIP-7702`
  - Marker 3: `ERC-4337`
  - Marker 4: `FRAME TX`
  - Below left zone: `PS COMPATIBLE`
  - Below right zone: `NEEDS WITNESSES`
- **Footer:** `$ aa --compatibility-check` | `19 / 22`

#### Slide 20 — The Inseparable Graph
- **Eyebrow:** `// HONEST CRITIQUE`
- **Title:** `The Inseparable` (green) `Graph` (white)
- **Body text:** "A fair critique: DeFi state is deeply interconnected. Serve USDC → need DEX pools → need routers → the closure is 'most of DeFi.'"
- **Two-column layout:**
  - LEFT: **When valid** — Large aggregators (CoW Swap, 1inch), MEV searchers, complex multi-hop routing
  - RIGHT: **When NOT valid** — VOPS nodes (account data only), ENS, privacy protocols, simple balance queries, specialized nodes serving one piece of a trade
- **Pull quote:** "PS nodes don't serve everything. They serve one piece. Proofs batch across nodes sharing the same state root."
- **Footer:** `$ state --graph-closure USDC` | `20 / 22`

### Act 5: Closing (~1.5 min)

#### Slide 21 — Implementation
- **Eyebrow:** `// ALREADY REAL`
- **Title:** `Already` (white) `Shipping` (green)
- **Code block:**
  ```
  # Run a partial stateful node
  geth --partial-state \
       --partial-state.contracts 0xA0b8...USDC,0x6B17...DAI \
       --partial-state.chain-retention 1024
  ```
- **Stat row (2 stats):** `59 GB` mainnet | `79%` reduction
- **Roadmap (3 items, horizontal):**
  - `Glamsterdam (2026)` → BALs land — PS becomes possible
  - `Hegota (2026)` → AA decision impacts PS compatibility
  - `Next forks (~2027+)` → ZKEVMs — PS becomes essential
- **Body note:** "go-ethereum PR #33764 — only geth for now. Opportunity for other clients."
- **Footer:** `$ git log --oneline ethereum/go-ethereum#33764` | `21 / 22`
- **Source footnote:** github.com/ethereum/go-ethereum/pull/33764

#### Slide 22 — CTA + Credits
- **Type:** Closing slide
- **Title (VT323, large):** `Try it.` (green) `Break it.` (white) `Tell us.` (green) + blinking cursor
- **Divider**
- **Body list (Plex):**
  - Run a PS node after Glamsterdam ships
  - Provide feedback on the implementation
  - Help design state markets
- **Credits (centered, dim):**
  - CPerezz & @DanGoron2
  - ethresear.ch/t/vops/22236
  - github.com/ethereum/go-ethereum/pull/33764
- **Footer:** `$ exit 0` | `22 / 22`

---

## Diagram Images Required

7 images to be generated via generative AI. All must match the terminal aesthetic: black background, phosphor green (#33ff33) primary, white secondary, no text (labels overlaid in HTML). 16:9 ratio, minimum 1920x1080.

| # | Filename | Slide | Content |
|---|----------|-------|---------|
| 1 | `state-collapse.png` | 4 | World map: dense green dots (left/today) → few concentrated white dots (right/post-ZKEVM) |
| 2 | `ps-node-architecture.png` | 7 | Vertical cross-section: dense account trie + tracked storage + empty untracked + ghosted full node |
| 3 | `mpt-proof-path.png` | 9 | Two-layer tree: storage trie → account trie, glowing proof path from leaf to state root |
| 4 | `focil-flow.png` | 12 | Left-to-right pipeline: mempool → PS node (filter) → inclusion list → builder → attesters |
| 5 | `state-market-flywheel.png` | 14 | Clockwise cycle: lower cost → more operators → geo distribution → better latency → more queries → more revenue |
| 6 | `redundancy-gradient.png` | 16 | Bar columns left-to-right: dense bright (USDC) → sparse dim (cold state) |
| 7 | `validation-spectrum.png` | 19 | Horizontal gradient bar green→red with 4 marker pins for each AA proposal |

Detailed generation prompts provided separately in conversation.

---

## Data Sources

All data points used in the presentation with their sources:

| Data Point | Value | Source |
|------------|-------|--------|
| Ethereum nodes | ~13,900 | ethernodes.org / nodewatch.io |
| Validator keys | 1M+ | beaconcha.in |
| Full node state size | ~280 GB | lab.ethpandaops.io/ethereum/execution/state-growth |
| PS node size (mainnet, USDC+DAI) | ~59 GB | go-ethereum PR #33764 (mainnet test) |
| Account trie size | ~50 GB | go-ethereum PR #33764 |
| USDC+DAI storage tries | ~9 GB | go-ethereum PR #33764 |
| State reduction | ~79% | Derived: (280-59)/280 |
| VOPS minimum storage | ~8.4 GB | ethresear.ch VOPS post (241M accounts × ~40 bytes) |
| FOCIL includers per slot | 16 | EIP-7805 |
| ERC-4337 smart accounts | 40M+ (end 2024) | Alchemy, Rhinestone |
| x402 machine-to-machine txs | 162M since Oct 2025 | Coinbase |
| Live AI agent projects | 15,800+ | Virtuals Protocol |
| State composition | 81.7% contract storage, 14.1% accounts | Core Paper (corepaper.org) |
| Falcon-512 gas: Schemed Tx | ~29,500 | ethereum-magicians.org Frame vs Schemed comparison |
| Falcon-512 gas: Frame Tx | ~63,000-82,880 | ethereum-magicians.org Frame vs Schemed comparison |

---

## Technical Requirements

### HTML Architecture
- Single self-contained HTML file, all CSS/JS inline
- Zero dependencies (fonts loaded from Google Fonts CDN)
- Semantic HTML (`<section>`, `<nav>`, `<main>`)

### Viewport Fitting (mandatory)
- Every `.slide`: `height: 100vh; height: 100dvh; overflow: hidden;`
- All font sizes: `clamp(min, preferred, max)`
- All spacing: `clamp()` based
- Breakpoints: 700px, 600px, 500px (height), 600px (width)
- `prefers-reduced-motion` support
- Content density limits enforced per slide type

### JavaScript
- `SlidePresentation` class with keyboard nav (arrows, space, pgup/pgdn), touch/swipe, mouse wheel
- `IntersectionObserver` for scroll-triggered `.visible` class → CSS `.reveal` animations
- Progress bar updates on scroll
- Navigation dots (right edge)
- Staggered reveal delays on child elements

### Content Density Limits
| Slide Type | Maximum |
|------------|---------|
| Title | 1 heading + 1 subtitle + tagline |
| Content | 1 heading + 4-6 bullets OR 1 heading + 2 paragraphs |
| Feature grid | 1 heading + 4 cards (2x2) |
| Stat slide | 1 heading + 3 stats + 1 paragraph |
| Diagram | 1 image + overlay labels + footer |
| Code | 1 heading + 8-10 lines |

### Image Handling
- Diagram images referenced via relative path: `assets/filename.png`
- `max-height: min(55vh, 480px)` on diagram slides
- Subtle border: `1px solid #1a2a1a`
- Border radius: `8px`
- Box shadow: `0 4px 20px rgba(0, 0, 0, 0.5)`

### Source Footnotes
- Displayed at bottom of relevant slides
- Font: IBM Plex Mono, `var(--small-size)`, color `#333`
- Format: `Source: domain.tld/path`
