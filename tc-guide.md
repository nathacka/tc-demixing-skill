# 🌪️ Tornado Cash — Investigator Guide

A reference for analysts using the TC Demixing Tool (v3.3)

---

## How Tornado Cash Works

Tornado Cash is a mixer protocol on Ethereum that breaks the on-chain link between a source and destination address using zero-knowledge proofs.

```
Deposit Address ──→ [ TC Pool ] ──→ Fresh Withdrawal Address
    (known)            (break)           (unknown link)
```

1. **Deposit** — User sends a fixed amount of ETH to a TC pool contract and receives a cryptographic note off-chain.
2. **Wait** — The ETH sits in the pool alongside other deposits. Longer wait = larger anonymity set.
3. **Withdraw** — User presents the note via ZK proof. ETH goes to a fresh address with no on-chain link to the deposit.

The protocol uses zero-knowledge proofs so the contract can verify the withdrawal is valid without revealing which deposit it corresponds to.

---

## ETH Pool Denominations

TC only accepts fixed denominations. This constraint is what makes analysis possible.

| Pool Contract | Amount | Label |
|---|---|---|
| `0x12D66f87A04A9E220743712cE6d9bB1B5616B8Fc` | 0.1 ETH | 🟣 |
| `0x47CE0C6eD5B0Ce3d3A51fdb1C52DC66a7c3c2936` | 1 ETH | 🟠 |
| `0x910Cbd523D972eb0a6f4cAe4618aD62622b39DbF` | 10 ETH | 🟢 |
| `0xA160cdAB225685dA1d56aa342Ad8841c3b53f291` | 100 ETH | 🔴 |

To mix 32 ETH, an actor must split across multiple deposits — e.g. three 10 ETH + two 1 ETH. This creates a fingerprint.

Note: Withdrawals may be slightly less than the pool denomination due to relayer fees (e.g. 0.99 ETH instead of 1 ETH). The tool uses range matching to account for this.

---

## Deposit Vector

The fingerprint that makes demixing possible. A deposit vector describes which pools an actor used and how many times:

```
Vector: (0, 2, 3, 0)
         │  │  │  │
         │  │  │  └── 🔴 100 ETH pool: 0 deposits
         │  │  └───── 🟢 10 ETH pool:  3 deposits = 30 ETH
         │  └──────── 🟠 1 ETH pool:   2 deposits = 2 ETH
         └─────────── 🟣 0.1 ETH pool: 0 deposits
                                           Total = 32 ETH
```

**Key insight:** The anonymity set only protects you if many other people use the same pool, at the same time, in the same pattern. Unique combinations of pool usage, timing, and behaviour significantly narrow down withdrawal candidates.

---

## Inside vs. Outside Window Analysis

The core analytical approach of this tool.

**🟢 Inside Window (Withdrawal Window)** — TC withdrawal activity that occurred *during* your investigation time period. These are the withdrawals that could plausibly be your actor cashing out after their deposits.

**🔴 Outside Window (Total Activity)** — All *other* TC withdrawal activity for that same address, before and after your investigation period. This is their lifetime TC history.

**Why this matters:**

- An address with heavy outside activity but minimal inside activity matching your vector is likely a regular TC user who overlapped your timeframe
- Your actor's pattern should primarily appear in the inside columns
- Focus on candidates whose inside pattern aligns with your deposit vector

---

## Step-by-Step Workflow

### Step 1: Load Deposits

- Paste deposit addresses or transaction hashes (comma/newline separated)
- Or upload a CSV with an `addresses` or `transactions` column
- The tool queries Chainalysis Data Solutions for all TC deposits from those addresses
- The sidebar shows the deposit vector and total ETH
- **Opsec detection:** If any deposit address also received TC withdrawals, a ⚠️ TC WITHDRAWAL badge appears next to it

### Step 2: Select and Apply

- Use checkboxes to select the deposits relevant to your investigation
- Is there a concentrated time frame or are they spread over weeks/months?
- Selecting a tighter window increases the chance of finding high-probability withdrawal candidates
- Click Apply Selection — the sidebar updates with the selected vector and the withdrawal window auto-populates

### Step 3: Set Withdrawal Window

- **From Date & Time (UTC)** — auto-populated from your latest selected deposit
- **Add Days** — how far to extend the search window:
  - 1-3 days: catches impatient actors, smaller anonymity set
  - 7-14 days: good default, balances coverage with candidate count
  - 30-90 days: patient actors, use gas funder or MCA to narrow

### Step 4: Analyse Withdrawal Candidates

- Click **Explore Withdrawal Candidates** to search
- **Pool filter table** — click pools to include/exclude:
  - Include pools your actor used — candidates must show activity in at least one
  - Exclude pools your actor never used — removes candidates with activity in those pools
  - Example: deposit vector (0,0,3,1) → exclude 🟣 0.1 and 🟠 1
- **Withdrawal candidates table** shows each address with:
  - Inside/outside pool activity (color-coded cells)
  - Status badges
  - Sending exposure (top outbound destination)
  - Days active
  - Balance

### Step 5: Drill Down

- **⛽ Gas Funder Analysis** — finds addresses that sent gas to 2+ withdrawal candidates. Fresh TC addresses need gas, and if multiple got it from the same source, they belong to the same actor.
- **🔀 Forward MCA** — traces where candidates sent funds after withdrawing. If multiple candidates forward to the same exchange or cluster, the consolidation re-links them.
- **📊 Reactor Graph** — select candidates and build a persistent Reactor graph with deposits on the left, TC cluster in the center, withdrawal candidates on the right with vector annotations, and sends-to destinations on the far right showing consolidation.

---

## Status Badges

| Badge | Meaning |
|---|---|
| ⚠️ **DEPOSIT ADDR** | This address also appears in your deposit list — major opsec failure |
| 🆕 **TC-ONLY** | 1 day active, only received from TC — classic single-use withdrawal address |
| **NEW** | 1 day active but has other inbound sources besides TC |
| **MIXED** | Active 30+ days — pre-existing address, less likely purpose-built |
| 💰 **HOLDING** | Balance over $100 — funds have not been moved yet |
| 📡 **obs** | IP-based geolocation observations with top 3 country codes |
| 🧅 **TOR** | Observations seen via Tor network |
| 🔀 **PROXY** | Observations seen via proxy |
| 🛡️ **VPN** | Observations seen via VPN |

---

## Tornado Cash General Activity

The collapsible panel below the withdrawal window shows:

**TC Withdrawals by Pool** — how many withdrawals happened per pool during your window, color-coded. Helps you understand the anonymity set size for each denomination.

**Where Funds Went** — top outbound destinations from all withdrawal candidates, by USD volume. Shows the overall pattern: are most going to DEXes, bridges, exchanges? Anomalies stand out.

---

## Reactor Graph Layout

When you select candidates and click "Add Selected to Reactor Graph":

```
[Deposit addr]        [Tornado.cash]      [Withdrawal 1] ──→ [LI.FI]
                                              (0,0,0,1)
                                          [Withdrawal 2] ──→ [LI.FI]
                                              (0,0,0,1)
                                          [Withdrawal 3] ──→ [Cow Swap]
                                              (0,0,0,1)
    LEFT                 CENTER               RIGHT           FAR RIGHT
```

- Deposit addresses on the left
- Real Tornado.cash cluster in the center
- Withdrawal candidates on the right with vector annotations connected by arrows
- Sends-to destinations as real Chainalysis cluster nodes on the far right
- Multiple candidates sending to the same destination share one node — consolidation is immediately visible

---

## Common Actor Behaviours

| Behaviour | What to Look For |
|---|---|
| Quick in-and-out | Deposits and withdrawals within hours. Small anonymity set. |
| Drip withdrawal | Withdraws to many addresses over days/weeks. Look for timing intervals and downstream consolidation. |
| Same gas funder | Withdrawal addresses all funded by the same gas source. |
| Bridge exit | Withdraws then immediately bridges to another chain. |
| Opsec failure | Withdraws back to the deposit address (⚠️ TC WITHDRAWAL badge). |
| DEX consolidation | Multiple withdrawals routed through the same DEX (visible in Sends To column and Reactor graph). |

---

## Limitations

- **Probabilistic, not deterministic.** TC analysis narrows candidates — it does not prove a link. Always corroborate with other evidence.
- **Anonymity set size matters.** If thousands of deposits happened in your window, vector matching alone may not be sufficient.
- **Sophisticated actors adapt.** They may add delays, use intermediary addresses, bridges, or multiple rounds of TC.
- **Ethereum only.** This tool covers the four ETH pool contracts. TC also has ERC-20 pools (DAI, cDAI, USDC, USDT) which are not covered.
- **Relayer fees.** Withdrawal amounts may be slightly below the pool denomination. The tool uses range matching (e.g. 0.9-1.0 for the 1 ETH pool).

---

## How to Launch

In a fresh Dialog chat:

> "Fetch tc-demixing.html from GitHub repo nathacka/tc-demixing-skill, save it to the workspace, and display it as an HTML artifact"

Or if registered in the investigator-skills catalog, just say:

> "Run the TC demixing tool"
