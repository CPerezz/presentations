# Binary Tries: Where We Stand — Presentation Design Spec

> EthCC 2026 · Stateless Summit | 10 min | CPerezz

## Overview

17-slide presentation covering binary trie benchmarks in geth (group-depth optimization + MPT comparison) and the Partition Binary Tree spec. Honest framing: "here's where we are, here's the gap, here's what we're doing about it." PBT positioned as forward-looking design for post-ZKEVM Ethereum, no performance claims.

Audience: researchers + core devs + enthusiasts at Stateless Summit. Assume Ethereum state management knowledge.

---

## Visual Style

Reuse CRT Hybrid Terminal theme from `partial-statefulness-ethcc2026.html`:
- VT323 for headings/numbers/accents, IBM Plex Mono for body
- Black background, phosphor green (#33ff33), white (#e8e8e8)
- Subtle scanlines + vignette, blinking cursor on all titles
- Terminal footer commands + slide numbers
- **Font sizes 20% larger than the PS presentation** (reviewer feedback: audience needs bigger text)

### Font Size Overrides (vs PS presentation)

```css
--title-size: clamp(2.5rem, 8vw, 5.5rem);
--h2-size: clamp(1.7rem, 4.5vw, 3.2rem);
--h3-size: clamp(1.1rem, 2.5vw, 1.7rem);
--body-size: clamp(0.9rem, 1.7vw, 1.3rem);
--small-size: clamp(0.75rem, 1.2vw, 1rem);
--tag-size: clamp(0.7rem, 1vw, 0.9rem);
--stat-size: clamp(2.2rem, 6vw, 4.5rem);
```

---

## Slide-by-Slide Structure

### Slide 1 — Title
- **Eyebrow:** `// STATELESS SUMMIT`
- **Title:** `Binary Tries:` (white) `Where We Stand` (green) + cursor
- **Subtitle:** "Group-depth benchmarks, MPT comparison & the Partition Binary Tree"
- **Meta:** CPerezz · EthCC 2026
- **Footer:** `$ geth --bintrie` | `1 / 17`

### Slide 2 — Why Binary Tries?
- **Eyebrow:** `// CONTEXT`
- **Title:** `Why` (white) `Binary Tries?` (green) + cursor
- **4 bullet points:**
  - Unified trie: all accounts + storage in a single 256-bit keyspace
  - Smaller witnesses: shorter Merkle proofs for stateless clients
  - Simpler architecture: one tree instead of per-account storage tries
  - More optimal for ZKEVMs: binary structure aligns with ZK circuit design
- **Pull quote:** "But how does it actually perform?"
- **Footer:** `$ bintrie --motivation` | `2 / 17`

### Slide 3 — Breaking Myths
- **Eyebrow:** `// CLARIFICATION`
- **Title:** `What Binary Tries` (white) `Don't Give You` (green) + cursor
- **2 myth cards:**
  - **Myth: "Bintries enable hash function changes"** — Reality: it's the tree migration that allows re-hashing, not the tree structure itself. We could rehash in an MPT migration too.
  - **Myth: "Bintries enable code chunking"** — Reality: code chunking can be done in MPT as well. The tree transition is what creates the opportunity, not the binary structure.
- **Pull quote:** "These are benefits of the migration, not the tree."
- **Footer:** `$ bintrie --myths --debunk` | `3 / 17`

### Slide 4 — What is Group Depth?
- **Eyebrow:** `// BENCHMARKS`
- **Title:** `Group` (white) `Depth` (green) + cursor
- **Body text:** "In a binary trie, each node branches into 2 children. Group depth (GD) packs multiple binary levels into a single node — trading fewer disk seeks for more hashing per node."
- **Key details (3 bullets):**
  - GD-1: 2 children per node, ~256 path nodes, ~64 bytes per node
  - GD-5: 32 children per node, ~52 path nodes, ~1 KB per node
  - GD-8: 256 children per node, ~32 path nodes, ~8 KB per node
- **Small note:** "Wider nodes = fewer disk seeks but more hashing per node. Where's the sweet spot?"
- **Footer:** `$ benchmark --explain group-depth` | `4 / 17`

### Slide 5 — [DIAGRAM] Group Depth Visualization
- **Type:** Full-slide diagram
- **Image:** `group-depth-visual.png`
- **Content:** Shows 2-3 examples side-by-side: GD-1 (narrow, tall tree — many small nodes), GD-5 (balanced — medium nodes), GD-8 (wide, short tree — few huge nodes). Each example shows a path from root to leaf, with node sizes visually proportional. The sweet spot (GD-5) highlighted.
- **Footer:** `$ benchmark --visualize gd-1 gd-5 gd-8` | `8 / 17`

### Slide 6 — The Experiment
- **Eyebrow:** `// BENCHMARKS`
- **Title:** `The` (white) `Experiment` (green) + cursor
- **Key details (4 bullets):**
  - 360 GB database, ~400M state entries (accounts + storage)
  - 8 group-depth configs (GD-1 through GD-8), cold cache, 9 runs per config
  - Mann-Whitney U tests for statistical rigor
  - Primary metric: Mgas/s (normalizes for block composition differences)
- **Footer:** `$ benchmark --configs GD-1..GD-8 --runs 9` | `9 / 17`
- **Source:** cperezz.github.io/bintrie-benchmarks/group-depth-benchmarks/

### Slide 7 — [DIAGRAM] GD Sweet Spot Chart
- **Type:** Full-slide diagram
- **Image:** `gd-sweet-spot.png`
- **Content:** Line/bar chart showing Mgas/s for reads, writes, mixed across GD-1 to GD-8. GD-5 and GD-6 highlighted. Clear inflection at GD-7.
- **Footer:** `$ benchmark --plot mgas-vs-gd` | `4 / 17`

### Slide 8 — The Verdict
- **Eyebrow:** `// RESULTS`
- **Title:** `The Sweet Spot:` (white) `GD-5 / GD-6` (green) + cursor
- **3 stat cards:**
  - `6.94` Mgas/s — GD-5 writes (+160% vs GD-1)
  - `6.39` Mgas/s — GD-6 reads (+141% vs GD-1)
  - `GD-7+` — Past inflection (nodes exceed NVMe block size)
- **Body text:** "GD-5 for write-heavy workloads. GD-6 for read-heavy/mixed. Both in the optimal 5-6 bit range. GD-7+ collapse: ~4KB nodes saturate Pebble block size."
- **Footer:** `$ benchmark --verdict` | `8 / 17`

### Slide 9 — The Honest Gap
- **Eyebrow:** `// MPT vs BINTRIE`
- **Title:** `The Honest` (white) `Gap` (green) + cursor
- **Setup:** "BT-GD5 vs production MPT. Same hardware: 48-core EPYC, 126GB RAM, 3.5TB SSD."
- **3 stat cards (amber/warning accent for emphasis):**
  - `1.7x` slower — reads (11.2 vs 19.0 Mgas/s)
  - `2.5x` slower — writes per slot (0.569 vs 0.229 ms)
  - `91%` — of 12-second slot budget (mixed workload)
- **Pull quote (alert box style):** "The binary trie is not ready for production today."
- **Footer:** `$ benchmark --compare mpt bt-gd5` | `9 / 17`
- **Source:** cperezz.github.io/bintrie-benchmarks/mpt-vs-bintrie/

### Slide 10 — [DIAGRAM] MPT vs Bintrie Architecture
- **Type:** Full-slide diagram
- **Image:** `mpt-vs-bintrie-arch.png`
- **Content:** Side-by-side: MPT (shallow, ~5 branch nodes, per-account storage tries) vs Binary Trie (deep, ~52 nodes, unified 256-bit keyspace). Shows why depth difference is architectural.
- **Footer:** `$ trie --compare-depth mpt bintrie` | `10 / 17`

### Slide 11 — Why the Gap Exists
- **Eyebrow:** `// ROOT CAUSE`
- **Title:** `Why` (white) `the Gap Exists` (green) + cursor
- **4 bullet points:**
  - Path depth: 52 nodes (BT) vs 5-6 (MPT) — 10x more disk seeks per lookup
  - Hash cost: ~260 internal ops per write (BT) vs ~10-15 (MPT)
  - 95-98% of "hash time" is traversal + serialization, not cryptography
  - Cache hit rate plateaus at 35-38% — 256-bit keyspace too sparse for meaningful reuse
- **Footer:** `$ benchmark --breakdown state_read trie_hash commit` | `11 / 17`

### Slide 12 — Closing the Gap
- **Eyebrow:** `// FUTURE DIRECTIONS`
- **Title:** `Closing` (white) `the Gap` (green) + cursor
- **4 bullet points (optimization avenues):**
  - Snapshot layer / pathdb fast reads — biggest potential, unexplored
  - Pebble block size tuning — GD-5 nodes are ~1KB vs 4KB blocks
  - Parallel EVM — NVMe QD=8 gives 8.7x throughput over QD=1
  - DB schema redesign — co-locating path siblings to reduce seek overhead
- **Body note:** "4 optimization PRs merged in one week. Trajectory is encouraging."
- **Footer:** `$ geth --optimize --parallel-evm --pebble-blocksize 16k` | `12 / 17`

### Slide 13 — [SECTION TITLE] Designing Forward
- **Eyebrow:** `// ACT II`
- **Title (hero, centered):** `The Partition Binary Tree` (green) + cursor
- **Footer:** `$ cd /pbt-spec` | `13 / 17`

### Slide 14 — PBT: What & Why
- **Eyebrow:** `// POST-ZKEVM DESIGN`
- **Title:** `Designed for` (white) `the Future` (green) + cursor
- **4 bullet points:**
  - Three zones: accounts (000, 12.5%), code (001, 12.5%), storage (1, 50%) — 25% reserved
  - No sequential dependency: MPT requires storage_root in account leaves → PBT eliminates this
  - Single-pass parallel root computation: zones update independently, converge at root
  - Enables: state expiry (per-account subtree pruning), partial statefulness, VOPS, proof compression (29-44%)
- **Footer:** `$ pbt --spec --zones 3` | `14 / 17`
- **Source:** cperezz.github.io/pbt-spec/

### Slide 15 — [DIAGRAM] Single-Pass Root Computation
- **Type:** Full-slide diagram
- **Image:** `pbt-parallel-root.png`
- **Content:** Left: MPT sequential chain (storage trie → storage_root → account leaf → account trie → state root). Right: PBT parallel (zone 000, zone 001, zone 1 all update bottom-up independently, converge at root). The visual contrast: chain vs parallel lanes.
- **Footer:** `$ pbt --root-computation --parallel` | `15 / 17`

### Slide 16 — Open Questions
- **Eyebrow:** `// OPEN QUESTIONS`
- **Title:** `What We` (white) `Don't Know Yet` (green) + cursor
- **4 bullet points:**
  - PBT performance: design first, benchmark later — no claims yet
  - Hash function selection: BLAKE3 (fast native) vs Poseidon2 (SNARK-friendly) vs Keccak (ecosystem)
  - Optimal storage prefix (P=60 → ~43 collision pairs at 10B accounts)
  - Resurrection strategy for expired subtrees (full proof vs incremental vs epoch-based)
- **Footer:** `$ research --status open` | `16 / 17`

### Slide 17 — CTA + Credits
- **Title (hero, centered):** `Benchmarks.` (green) `Specs.` (white) `Feedback.` (green) + cursor
- **3 bullets:**
  - Group-depth benchmarks: cperezz.github.io/bintrie-benchmarks/
  - PBT spec: cperezz.github.io/pbt-spec/
  - Geth binary trie branch: github.com/ethereum/go-ethereum
- **Credits:** CPerezz · EthCC 2026 · Stateless Summit
- **Footer:** `$ exit 0` | `17 / 17`

---

## Diagram Images Required

4 images, all matching CRT terminal aesthetic (black bg, phosphor green #33ff33, white secondary). 16:9, 1920x1080 min.

| # | Filename | Slide | Content |
|---|----------|-------|---------|
| 1 | `group-depth-visual.png` | 5 | Side-by-side: GD-1 (narrow/tall), GD-5 (balanced), GD-8 (wide/short) showing node sizes and path lengths |
| 2 | `gd-sweet-spot.png` | 7 | Line/bar chart: Mgas/s across GD-1 to GD-8, three series (reads/writes/mixed), GD-5/GD-6 highlighted |
| 3 | `mpt-vs-bintrie-arch.png` | 10 | Side-by-side: MPT (shallow ~5 nodes, per-account tries) vs BT (deep ~52 nodes, unified trie) |
| 4 | `pbt-parallel-root.png` | 15 | Left: MPT sequential chain. Right: PBT parallel lanes converging at root |

---

## Data Sources

| Data Point | Value | Source |
|------------|-------|--------|
| GD benchmark DB size | 360 GB, ~400M entries | cperezz.github.io/bintrie-benchmarks/group-depth-benchmarks/ |
| GD-5 write throughput | 6.94 Mgas/s | Group-depth benchmarks |
| GD-6 read throughput | 6.39 Mgas/s | Group-depth benchmarks |
| GD-1 read throughput (baseline) | 2.65 Mgas/s | Group-depth benchmarks |
| MPT vs BT hardware | 48-core EPYC, 126GB RAM, 3.5TB SSD | cperezz.github.io/bintrie-benchmarks/mpt-vs-bintrie/ |
| Read gap | 1.7x (11.2 vs 19.0 Mgas/s) | MPT vs bintrie benchmarks |
| Write gap per slot | 2.5x (0.569 vs 0.229 ms/slot) | MPT vs bintrie benchmarks |
| Mixed slot budget | 91% of 12-second slot | MPT vs bintrie benchmarks |
| BT path depth | ~52 nodes | MPT vs bintrie benchmarks |
| MPT path depth | ~5-6 nodes | MPT vs bintrie benchmarks |
| BT internal hash ops/write | ~260 | MPT vs bintrie benchmarks |
| Cache hit rate | 35-38% | Group-depth benchmarks |
| NVMe QD=1→QD=8 improvement | 8.7x | Gary Rong disk benchmarks |
| PBT zone split | 50%/25%/12.5%/12.5% | cperezz.github.io/pbt-spec/ |
| PBT proof compression | 29-44% savings | PBT spec |
| PBT account key bits | 245 (birthday bound 2^122.5) | PBT spec |

---

## Technical Requirements

Same as `partial-statefulness-ethcc2026.html`:
- Single self-contained HTML file, all CSS/JS inline
- Viewport fitting: 100vh per slide, clamp() everywhere, responsive breakpoints
- SlidePresentation JS class: keyboard nav, touch/swipe, wheel, progress bar, nav dots
- IntersectionObserver for .visible → .reveal animations
- Alert box CSS (amber accent) reused for "not ready for production" callout
- Source footnotes on data slides

---

## Verification

1. Open in browser, navigate all 14 slides
2. Verify fonts are visibly larger than PS presentation
3. All images load from assets/
4. No content overflow on any slide
5. Footer numbers sequential 1-14
6. All data points match source documents
