# Tornado Cash Demixing

Ethereum Tornado Cash deposit analysis, withdrawal candidate identification with vector matching, inside/outside window analysis, geolocation observations, sending exposure, and Reactor graph generation.

## Investigator Guide

See `tc-guide.md` for the full reference — how TC works, deposit vectors, inside/outside windows, status badges, common actor behaviours, and limitations.

## How to Launch

Fetch the interactive page from this repo and display it:

```python
from chainalysis_skill_github import GitHubClient
import base64

client = GitHubClient()
content = client.request("GET",
    "repos/nathacka/tc-demixing-skill/contents/tc-demixing.html",
    params={"ref": "main"}
)
html = base64.b64decode(content["content"]).decode("utf-8")
with open("tc-demixing.html", "w") as f:
    f.write(html)
```

Then call `displayArtifact` with `artifactType: "html"`, `path: "tc-demixing.html"`.

## Workflow

- **Slug:** `tc-demixing-v2` (org-public)
- **Actions:** `deposits`, `withdrawals`, `withdrawal_exposure`, `gas_funder`, `forward_mca`, `reactor_graph`

## Requirements

- Chainalysis Data Solutions access (Ethereum)
- Reactor graph access (for graph generation)

## Features

### Step 1: Load Deposits
- Input addresses or transaction hashes (paste or CSV upload)
- Returns deposit list with pool denomination (0.1 / 1 / 10 / 100 ETH)
- Builds deposit vector fingerprint
- Detects opsec failures (TC withdrawals back to deposit address)

### Step 2: Withdrawal Candidates
- Set time window (start date/time + days)
- Inside window: TC withdrawal activity during your investigation period
- Outside window: all other lifetime TC activity (context)
- Status badges: TC-ONLY, NEW, MIXED, HOLDING, DEPOSIT ADDR
- Geolocation observations with country flags + TOR/PROXY/VPN detection
- Top sending exposure (where funds went: exchange, bridge, DEX)
- Pool inclusion/exclusion filtering based on deposit vector
- Sortable columns, resizable

### Step 3: Drill Down
- Gas Funder Analysis: find shared gas sources across candidates
- Forward MCA: find shared forward destinations (consolidation)
- Reactor Graph: build persistent graph with deposits, TC cluster, withdrawal candidates, vector annotations, and sends-to destinations

### TC General Activity
- Pool-level breakdown of all TC withdrawals in the window
- Top outbound destinations by USD volume

## Input

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Which analysis to run |
| `addresses` | array/string | For deposits | Deposit addresses |
| `transactions` | array/string | For deposits | Transaction hashes (alternative) |
| `start_date` | string | For withdrawals | Start of window (YYYY-MM-DDTHH:MM:SS) |
| `days` | integer | For withdrawals | Window duration in days (default 3) |

## Triggers

tornado cash, TC demixing, demix, tornado cash analysis, TC withdrawal, TC deposits, mixer demixing
