---

# 🐋 SoDEX Whale Monitor — Tutorial

Complete guide on how the dashboard works, what each panel is for, and how to interpret the numbers it displays.

> Single HTML/CSS/JS file (`sodex-whale-monitor.html`), no build step — just open it in the browser or serve it with a static server.

---

## Table of Contents

1. [What is this dashboard](#1-what-is-this-dashboard)
2. [How data reaches the screen (architecture)](#2-how-data-reaches-the-screen-architecture)
3. [How to run the dashboard](#3-how-to-run-the-dashboard)
4. [SoDEX API Limits](#4-sodex-api-limits)
5. [Monitored Wallets Panel](#5-monitored-wallets-panel)
6. [Status Bar and Automatic Discovery](#6-status-bar-and-automatic-discovery)
7. [Live Positions & Events](#7-live-positions--events)
8. [Top 10 Ranking](#8-top-10-ranking)
9. [PnL per Wallet (Chart)](#9-pnl-per-wallet-chart)
10. [Equity Evolution](#10-equity-evolution)
11. [Whale Analysis](#11-whale-analysis)
    - [Dominant Direction](#111-dominant-direction)
    - [Whale Score](#112-whale-score)
    - [Whale Consensus](#113-whale-consensus)
    - [Conviction Score](#114-conviction-score)
    - [Whale Momentum](#115-whale-momentum)
    - [Whale Heatmap](#116-whale-heatmap)
    - [Whale Leaderboard](#117-whale-leaderboard)
12. [Signals (Entry Decision)](#12-signals-entry-decision)
    - [Signal Master Table](#121-signal-master-table)
    - [Composite Score](#122-composite-score)
    - [Entry Cluster Detection](#123-entry-cluster-detection)
    - [Smart Exit Monitor](#124-smart-exit-monitor)
13. [Risk & History](#13-risk--history)
    - [Market Risk Score](#131-market-risk-score)
    - [Historical Probability](#132-historical-probability)
    - [Open / Closed / History Operations](#133-open--closed--history-operations)
14. [Language Switch (PT/EN)](#14-language-switch-pten)
15. [Troubleshooting (CORS)](#15-troubleshooting-cors)
16. [Quick Glossary](#16-quick-glossary)

---

## 1. What is this dashboard

The **SoDEX Whale Monitor** is an observability dashboard that tracks, in real time, the open positions of up to **10 EVM wallets** on the **SoDEX** perpetuals exchange. The core idea is: if several historically profitable wallets ("whales") are entering the same direction, at the same time, with significant capital — this is a signal worth watching.

The dashboard turns open/closed positions into various metrics (consensus, conviction, momentum, entry clusters, historical probability, etc.) and consolidates everything into a final **actionable signals** table (`GO` / `WAIT` / `BLOCK`) that can even be consumed by an external bot (there is a ready JSON output for this).

It is **not** an exchange and does not execute orders — it is **read-only**: it reads public SoDEX data via WebSocket/REST and presents everything visually.

---

## 2. How data reaches the screen (architecture)

The operation is divided into 3 phases, documented at the top of the `<script>`:

```
PHASE 1 — DISCOVERY (~2 min, only runs if no wallets were provided manually)
  Opens 1 WebSocket and subscribes to the public "trade" channel (Market Trades Stream).
  Accumulates volume per address until the stop criterion is met (min. 90 unique traders).

PHASE 2 — RANKING BY REALIZED PNL (REST)
  For the top ~30 addresses by volume:
    GET /accounts/{addr}/positions/history?limit=500
  Sums the realizedPnL of each and selects the Top N (10) with the highest PnL.

PHASE 3 — REST SNAPSHOT + PERMANENT WEBSOCKET
  Initial snapshot: GET /accounts/{addr}/state → open positions + TP/SL.
  WebSocket subscribes (and keeps open):
    all_mark_prices, account_trades, account_updates,
    account_order_updates, account_events
  PnL is recalculated in real time with every new mark price received.
  Automatic reconnection with backoff + heartbeat {"op":"ping"} every 25s.
```

Used endpoints:

| Service | URL |
|---|---|
| Perpetuals WebSocket | `wss://mainnet-gw.sodex.dev/ws/perps` |
| Perpetuals REST | `https://mainnet-gw.sodex.dev/api/v1/perps` |
| Leaderboard (automatic top wallets search) | `https://mainnet-data.sodex.dev/api/v1/leaderboard` |

You can follow this flow in real time through the indicators in the **status bar** (`Mark Prices WS`, `Account Streams WS`, `REST Snapshot`).

---

## 3. How to run the dashboard

Because the file makes REST calls, opening it directly as `file://` in the browser blocks those calls due to CORS policy (WebSocket continues to work normally). It is recommended to serve it locally:

```bash
npx serve .
# or
python3 -m http.server 8080
```

Then access, for example:

```
http://localhost:8080/sodex-whale-monitor.html
```

If direct REST fails (due to CORS), the code automatically tries a public proxy (`corsproxy.io`) as fallback — but serving locally is still the most reliable method.

---

## 4. SoDEX API Limits

The dashboard respects (and displays in the status bar) the per-IP limits imposed by SoDEX:

- **10 unique wallets** in simultaneous WebSocket streams
- **10 simultaneous connections** (the dashboard uses only 1, multi-subscribed)
- **2,000 messages/minute**
- **1,200 "weight" REST/minute** (weighted rate limit by call cost)

That’s why the wallets field accepts a **maximum of 10 addresses**.

---

## 5. Monitored Wallets Panel

<img alt="wallets panel" />

**Purpose:** This is the starting point — here you manually enter up to 10 EVM addresses, or use automatic discovery.

**How to use:**
- Paste addresses into the grid and click **▶ Apply** to start tracking.
- **Search top wallets by PnL** (🔍): triggers automatic search — first tries SoDEX public leaderboard; if unavailable, listens to the trade stream for a few seconds to discover active addresses and then ranks them by realized PnL.
  - `Listen WS`: how many seconds to collect addresses in discovery mode.
  - `Top N`: how many wallets to fill (max 10).
  - `Min trades`: requires a minimum trade history before considering the wallet "valid".
- **Clear all**: resets the address grid.

---

## 6. Status Bar and Automatic Discovery

Right below the wallets panel, a thin bar shows:

- Current **Mode** (discovery, live, etc.)
- Connection status for each (green dot = connected): `Mark Prices WS`, `Account Streams WS`, `REST Snapshot`
- IP Limits (10 wallets · 10 connections · 2000 msg/min · 1200 REST weight/min)
- Message rate (`msg/min`) and update cycle counter

When automatic discovery is active, a progress bar shows how many "unique traders" have been seen in the public trades stream.

---

## 7. Live Positions & Events

Two side-by-side panels, updated in real time:

| Panel | Purpose |
|---|---|
| **Open positions** | Lists all currently open positions across monitored wallets: pair, side (long/short), size, leverage, unrealized PnL. |
| **Live events** | Chronological feed of events: opening, closing, increase, reduction, and liquidation of positions, as they arrive via WebSocket. |

**Reading example:** if the events feed shows "0x1a2b...ef34 opened LONG in BTC-USD with 10x leverage", it means that monitored wallet has just opened a long position in BTC with 10x leverage.

---

## 8. Top 10 Ranking

Section with two panels:

- **Realized PnL Ranking**: sorts monitored wallets by closed profit/loss, also showing number of Long/Short positions and total trades per wallet.
- **Monitored Wallets Details**: cards with complementary information per address.

This ranking defines the `#1, #2, #3...` order used as a "seal" (🥇🥈🥉) in other panels, such as the Whale Score.

---

## 9. PnL per Wallet (Chart)

Line chart (canvas) comparing the PnL evolution of each monitored wallet over time.

**Available filters:**
- `Wallet`: show all, or only Top 1/2/3/5 from the ranking.
- Selection of a specific wallet.

Use it to visually compare who is performing best in the analyzed period.

---

## 10. Equity Evolution

The densest panel in the dashboard: a **total equity** view of the wallets over time, with event markers.

**Main elements:**

- **Equity chart** (left): evolution of total value (or per wallet), with selectable periods: `1D · 1W · 1M · 3M · 6M · 12M · All`.
- **Position markers** on the chart:
  - 🟢 solid green dot = **closed Long** position
  - 🔴 solid red dot = **closed Short** position
  - 🟢 dashed green ring = **open Long** position
  - 🔴 dashed red ring = **open Short** position
  - You can toggle each marker type by clicking the legend.
- **Fetch full history** (⟳): triggers a deep search of all trades and funding history since the beginning — useful for reconstructing equity of older wallets (shows a progress bar).
- **Open Operations** (right): detailed table of currently open positions for the selected wallet(s), with filters by direction (Long/Short) and sorting by PnL, size, or open date.

---

## 11. Whale Analysis

Set of panels that translate the behavior of monitored wallets into collective conviction metrics.

### 11.1 Dominant Direction

Gives a quick summary of whether most monitored whales are currently **Long** or **Short** overall — a fast compass of aggregate sentiment.

### 11.2 Whale Score

**Purpose:** score from 0 to 100 that summarizes the *historical quality* of a wallet as a trader — the higher, the more successful that wallet has been historically.

Formula used (`calcWhaleScore`):

```
Win Rate   (wr)      = wins / total trades × 100
Profit Factor (pf)   = sum of gains / sum of losses
pfNorm                = normalizes PF to 0–100
                         (PF < 1 → pf × 50 | PF ≥ 1 → 50 + (pf-1) × 25, capped at 100)
Consistency (consist)= logarithmic scale of number of trades
                         (10 trades ≈ 50 | 50 trades ≈ 80 | 100+ trades ≈ 100)

Final Score = wr × 0.40 + pfNorm × 0.40 + consist × 0.20
```

**Tier Classification:**

| Score | Tier |
|---|---|
| ≥ 80 | 🟡 Elite |
| ≥ 60 | 🟢 Strong |
| ≥ 40 | 🔵 Average |
| < 40 | 🟠 Weak |

**Example:** a wallet with 65% Win Rate, 2.2x Profit Factor and 80 trades:
- `pfNorm = 50 + (2.2-1)×25 = 80`
- `consist ≈ log10(80)/log10(200) × 100 ≈ 84`
- `score = 65×0.4 + 80×0.4 + 84×0.2 = 26 + 32 + 16.8 ≈ 75` → **Strong** tier

### 11.3 Whale Consensus

**Purpose:** for each pair (e.g. `BTC-USD`), shows **how many** of the monitored whales are Long vs Short right now, with a percentage bar.

- Each colored dot represents a specific wallet (click to filter the entire dashboard by it).
- Filters: minimum whales in the pair, direction, and asset.

**Example:** `BTC-USD` with 7 whales Long and 3 Short → "Consensus: Long 70%".

### 11.4 Conviction Score

**Purpose:** different from Consensus (which counts *wallets*), Conviction weighs by **capital** (size × leverage) — i.e., it shows how much real money is bet on each side, not just how many people.

```
position weight = size (USDC) × leverage
Conviction (%) = weight of dominant side / total pair weight × 100
```

**Classification:** `≥75% = high` · `≥55% = medium` · `<55% = low`.

**Why it matters:** a pair may have 6 whales Long and 4 Short (Consensus 60% Long), but if the 4 Short have much larger/leveraged positions, Conviction may indicate Short dominance — meaning "the big money" disagrees with the simple majority.

### 11.5 Whale Momentum

**Purpose:** measures whether whale exposure in a pair is **growing or shrinking** between updates (window of ~5s, the dashboard render interval).

```
pctChange = (current exposure − previous exposure) / previous exposure × 100
```

**Classification:**

| Variation | Grade |
|---|---|
| > +50% | 🚀 Strong Momentum |
| +5% to +50% | 📈 Growing |
| -5% to +5% | ➡️ Stable |
| -50% to -5% | 📉 Shrinking |
| < -50% | 💥 Strongly Shrinking |

### 11.6 Whale Heatmap

**Purpose:** ranking of assets (pairs) by how "hot" whale activity is, combining three normalized factors from 0–100:

```
Score = (number of whales / max whales)   × 40%
      + (capital in pair / max capital)   × 40%
      + (pair conviction)                 × 20%
```

Assets at the top of the heatmap are where there are **more whales and more concentrated capital** at the same time.

### 11.7 Whale Leaderboard

Simplified podium-style ranking (🥇🥈🥉) of monitored wallets by Win Rate and Profit Factor, with trade count — a leaner version of the Whale Score focused on quick comparison.

---

## 12. Signals (Entry Decision)

This is the section that **consolidates everything** into a recommendation per pair: enter, wait, or avoid.

### 12.1 Signal Master Table

Table with one row per pair, combining: Consensus, Conviction, Momentum, Heatmap, Entry Cluster range, Exit %, historical Win Rate, "gates" (✓/✗), and the final **Score** + **Action**.

At the bottom there is also a **JSON** output (`whale-signal-json`) ready to be consumed by an external bot, with all calculated signals.

### 12.2 Composite Score

Same logic as the Signal Master Table, but in **card** format — more visual, showing the detail of each score component and which "gates" passed or failed.

**How the final Score is calculated** (function `_buildSignalData`, shared by both panels):

```
Consensus  = % of whales on the dominant side of the pair
Conviction = % of capital (size × leverage) on the dominant side
Momentum   = 50 + (exposure variation % × 2), capped at 0–100
Heatmap    = pair capital / capital of largest pair × 100
WinRate    = % of historical winning trades in that pair (min. 3 trades; otherwise uses 50 as neutral)

Score = Consensus×0.25 + Conviction×0.25 + Momentum×0.20 + Heatmap×0.15 + WinRate×0.15
```

**The 6 "gates" — each worth 1 approval point:**

| Gate | Condition to Pass |
|---|---|
| Consensus | ≥ 70% |
| Conviction | ≥ 65% |
| Exit | ≤ 25% from peak |
| Cluster | exists entry zone with ≥ 2 whales |
| Momentum | ≥ 55 (i.e., growing) |
| Win Rate | ≥ 60% (historical) |

**Final Decision (Action):**

```
GO    → Score ≥ 75  AND  at least 5 of 6 gates approved
WAIT  → Score ≥ 55  AND  at least 3 of 6 gates approved
BLOCK → any other case
```

**Practical example:** a pair with Consensus 82%, Conviction 70%, Momentum 60, Heatmap 55, Win Rate 65% →
`Score = 82×0.25 + 70×0.25 + 60×0.20 + 55×0.15 + 65×0.15 = 68`.
If 4 of 6 gates passed, the action would be **WAIT**.

### 12.3 Entry Cluster Detection

**Purpose:** identifies **price zones** where multiple whales opened positions close to each other (within 1% difference in entry price) — indicating a collective "zone of interest".

The algorithm groups entry prices (by pair + direction) using a greedy clustering approach: sorts prices and starts a new cluster whenever the next price is more than 1% away from the start of the current cluster.

**Example:** 4 whales opened Long in `ETH-USD` between $3,200 and $3,212 → single cluster "Whale Zone: 3200 → 3212 (4 whales)".

Filters: minimum whales in cluster, direction, and asset.

### 12.4 Smart Exit Monitor

**Purpose:** monitors whether whales are **exiting** a collective position. Compares the number of whales that have been in that pair/direction (session historical peak) with how many are still in now.

```
Exit % = (peak whales − whales currently open) / peak whales × 100
```

A high Exit % suggests the group that originally entered is already leaving — useful as a signal weakening alert. Filters by wallet, asset, direction, and variation level (high/low).

---

## 13. Risk & History

### 13.1 Market Risk Score

**Purpose:** a "thermometer" per pair that combines Consensus, Conviction, and Momentum into a single 0–100 score (leaner version of Composite Score, focused on risk/strength of the move, without depending on Win Rate/Heatmap/Cluster).

```
Score = Consensus × 0.40 + Conviction × 0.35 + Momentum × 0.25
```

**Classification:** `≥70 = 🟢 Strong` · `≥45 = 🟡 Medium` · `<45 = 🔴 Weak`.

Filters: asset, direction, minimum Consensus, minimum Conviction.

### 13.2 Historical Probability

**Purpose:** looks only at the **past**: of all closed positions (by any monitored whale) in a given pair, what % ended in profit (TP) vs loss (SL)?

```
Success Rate = TP / (TP + SL) × 100     (requires at least 3 operations in the pair)
```

Displayed as a visual gauge per pair. `≥65% = high probability`, `≥45% = medium`, below that = low.

### 13.3 Open / Closed / History Operations

Three paginated tables at the bottom of the dashboard:

| Table | Content |
|---|---|
| **Open Operations** | All currently open positions across monitored wallets, with filters by asset, wallet, direction, and PnL (positive/negative). |
| **Closed Positions** | Latest closed positions, with final result. |
| **History** | Consolidated record (via REST) of older trades and liquidations, plus the current session. |

All support pagination and the same basic filters (asset, wallet, direction).

---

## 14. Language Switch (PT/EN)

In the top right there is a selector 🇧🇷/🇺🇸. It **does not reload the page or call another backend** — a JavaScript dictionary (`DICT`) scans all visible text in the DOM (including `title`, `placeholder`, `aria-label`) and performs real-time string replacement, keeping a `MutationObserver` so that any newly rendered content (positions, events, etc.) is also automatically translated while English is selected.

---

## 15. Troubleshooting (CORS)

If you see the yellow banner:

> ⚠ REST blocked — CORS issue

This happens because the file was opened directly (`file://`) and the browser blocks REST calls for security. WebSocket continues to work (dashboard enters **WS-only** mode), but the initial positions snapshot does not load until the next real-time update. Solution:

```bash
npx serve .
# or
python3 -m http.server 8080
```

And access via `http://localhost:8080/...`.

---

## 16. Quick Glossary

| Term | Meaning |
|---|---|
| **Whale** | Wallet monitored by the dashboard, usually with relevant PnL history. |
| **Consensus** | % of whales agreeing in the same direction (counts wallets). |
| **Conviction** | % of capital (size × leverage) concentrated on the dominant side (weighs money, not wallets). |
| **Momentum** | Whether exposure is growing or shrinking between updates. |
| **Entry Cluster** | Price zone where several whales entered close to each other. |
| **Exit %** | How much of the peak number of whales in a position has already exited. |
| **Gates** | Binary criteria (passed/failed) used to validate a signal before turning it into "GO". |
| **GO / WAIT / BLOCK** | Final verdict of the Composite Score / Signal Master Table for a pair. |

---
