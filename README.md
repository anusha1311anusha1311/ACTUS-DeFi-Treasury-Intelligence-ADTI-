# ACTUS-DeFi-Treasury-Intelligence-ADTI

> **ACTUS-based multi-asset-class treasury simulation — BTC/ETH digital assets, T-Bills, accounts receivable/payable with dynamic discounting, stablecoin holdings, and consolidated cash management — powered by the generalRisk MCP server.**

---

## What This Does

This project simulates a complete corporate treasury portfolio where digital assets (BTC, ETH), traditional investments (T-Bills), operational working capital (AR/AP with early settlement and late-payment penalties), and stablecoin holdings (USDC) are managed together under a single ACTUS simulation. The system monitors allocation drift, triggers rebalancing, handles early settlements and penalty accruals, and tracks consolidated cash flows with liquidity buffer protection.

When connected to Claude via the `generalRisk` MCP server, you upload a Postman collection JSON, ask a question, and receive real simulation results as interactive visualizations — all from actual ACTUS contract event computations, not mocked data.

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────────────┐
│  Claude AI   │────▶│  generalRisk     │────▶│    ACTUS Risk Engine     │
│  + JSON file │     │  MCP Server      │     │  :8082 Risk Factor Svc   │
│  + Query     │◀────│  (7 tools)       │◀────│  :8083 Simulation Svc    │
└──────────────┘     └──────────────────┘     └──────────────────────────┘
       │                                              │
       ▼                                     localhost (Docker) OR
  Allocation drift charts,                   AWS (34.203.247.32)
  cash flow dashboards,
  portfolio composition
```

---

## Client-Side Setup

### Step 1 — Clone the Repository

```bash
git clone <https://github.com/anusha1311anusha1311/ACTUS-DeFi-Treasury-Intelligence-ADTI-.git>
cd ADTI-actus-external-risk
```
### step 2 - Docker running

```powershell
.\start-risk-actusservice-local.bat
```

### Step 3 — Install Dependencies & Build the MCP Server

```bash
cd Backend
npm install
npm run build
```

This compiles `src/mcp-server.ts` → `dist/mcp-server.js` using TypeScript (`tsc`).

### Step 3 — Add the MCP Server to Claude Desktop

Open your Claude Desktop config file:

| OS | Path |
|---|---|
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |

Add the `generalRisk` server entry:

```json
{
  "mcpServers": {
    "generalRisk": {
      "command": "node",
      "args": [
        "<path-to-your-clone>\\ACTUS-DeFi-Treasury-Intelligence-ADTI-\\ADTI-interface\\Backend\\dist\\mcp-server.js"
      ]
    }
  }
}
```

Replace `<path-to-your-clone>` with your actual clone location. For example:


### Step 4 — Restart Claude Desktop

Close and reopen Claude Desktop. The `ACTUS-DeFi-Treasury-Intelligence-ADTI-` MCP server will start automatically.

### Step 5 — Verify the Tools Are Available

In Claude Desktop, look for the MCP tools icon (hammer/wrench). You should see the server named **`ACTUS-DeFi-Treasury-Intelligence-ADTI-`** with **7 tools**:

| Tool | Purpose |
|---|---|
| `run_simulation` | Execute ACTUS simulations (the main tool) |
| `list_simulations` | Browse available collections by domain |
| `load_simulation` | Load a specific collection file |
| `verify_portfolio` | Check stablecoin portfolio compliance |
| `get_threshold_presets` | Get EU MiCA / US GENIUS Act thresholds |
| `list_sample_portfolios` | List sample portfolio files |
| `load_sample_portfolio` | Load a sample portfolio |

When Claude calls any tool for the first time, you will see a permission prompt — click **Allow** (or **Allow for this chat**) to proceed.

### Step 6 — ACTUS Engine (Local Docker — Optional)

For local execution, you need the ACTUS Docker containers running. From the ACTUS engine repo:

```bash
cd ADTI-actus-external-risk
.\start-risk-actusservice-local.bat

```

This starts:
- **MongoDB** on port `27018`
- **actus-riskserver-ce** (risk factor service) on port `8082`
- **actus-server-rf20** (simulation service) on port `8083`

Verify:
```bash
curl http://localhost:8082/findAllScenarios
curl http://localhost:8083/
```

If Docker is not running, the MCP server automatically falls back to the AWS-hosted engine at `34.203.247.32:8082` / `34.203.247.32:8083`.

---

## Demo Prompts

Upload a Hybrid Treasury collection JSON to Claude, then run these prompts:

### Prompt 1 — Run Consolidated Portfolio
```
Run the hybrid treasury consolidation simulation. Show me the full portfolio 
breakdown: BTC position value, ETH position value, T-Bill maturities, AR/AP 
settlements, USDC yield, and consolidated cash balance over the 6-month horizon.
```

### Prompt 2 — Allocation Drift Analysis
```
Show me when BTC and ETH allocation drift triggered rebalancing events. 
Plot the allocation percentage vs the min/target/max bands (15%/20%/25% for BTC, 
7%/10%/15% for ETH) and mark each PP event where drift correction occurred.
```

### Prompt 3 — Cash Flow Waterfall
```
Create a cash flow waterfall showing all inflows (AR settlements, T-bill 
maturities, digital asset sell proceeds, USDC yield) and outflows 
(AP payments, deployments). Track the running cash balance and show where 
the liquidity buffer model triggers.
```

### Prompt 4 — Early Settlement vs. Penalty
```
Compare AR early settlements (discounted) vs AP late payment (penalized). 
Show me which AR invoices settled early, what discount was applied, and 
what penalty accrued on the overdue AP-003 invoice.
```

---

## How It Works — Two-Pass Architecture

Consolidated collections (HT-CONS series) use a two-pass simulation:

**Pass 1 — Asset-Only Run (Phase 0–2):**
1. Load 6 base reference indexes (BTC price, ETH price, portfolio NAV, hurdle rate, counterparty liquidity, USDC peg deviation)
2. Load risk models (AllocationDriftModel, EarlySettlementModel, PenaltyAccrualModel)
3. Create Phase 0 scenario and run asset-only simulation
4. JavaScript post-request scripts capture all events and compute market-value proceeds (BTC/ETH sold at spot price, not book value)

**Pass 2 — Consolidated Run (Phase 3–6):**
5. Register 5 derived reference indexes + 7 ScheduledCashFlowModels + 1 LiquidityBufferModel
6. Create consolidated scenario (up to 28 risk factor descriptors)
7. Run full simulation with all contracts including central CASH-PAM treasury

The ACTUS endpoint for all hybrid treasury simulations is `POST :8083/rf2/scenarioSimulation`.

---

## ACTUS Contracts

A consolidated treasury simulation uses up to 15 contracts across 4 asset classes:

| Contract ID Pattern | Type | Role | Purpose |
|---|---|---|---|
| `BTC-CLM-SpotTrade-{qty}` | CLM | RPA | BTC position — `treasuryModels` attaches AllocationDriftModel |
| `ETH-CLM-SpotTrade-{qty}` | CLM | RPA | ETH position — `treasuryModels` attaches AllocationDriftModel |
| `TBILL-{tenor}-{amount}` | PAM | RPA | US Treasury bills (zero coupon, discount via `premiumDiscountAtIED`) |
| `AR-INV-{nnn}` | PAM | RPA | Accounts receivable — `discountingModels` for early settlement |
| `AP-INV-{nnn}` | PAM | RPL | Accounts payable — `discountingModels` for early settlement or penalty |
| `USDC-HOLD-{amount}` | PAM | RPA | Stablecoin holding at yield (e.g., 4.2%) |
| `CASH-PAM-{amount}` | PAM | RPA | Central cash pool — `treasuryModels` attaches 7 SCFs + LiquidityBuffer |

Key contract wiring:
- CLM contracts use `treasuryModels` (not `prepaymentModels`) for allocation drift
- AR/AP PAM contracts use `discountingModels` for early settlement / penalty models
- CASH-PAM uses `treasuryModels` for all 7 ScheduledCashFlowModels + LiquidityBufferModel
- AP contracts use `contractRole: "RPL"` (liability), all others use `"RPA"` (asset)

---

## Risk Models

### AllocationDriftModel
Monitors digital asset allocation vs. portfolio total and triggers rebalancing:
- `targetAllocation` / `minAllocation` / `maxAllocation` — e.g., BTC: 15%/20%/25%
- `positionQuantity` — fixed quantity (e.g., 20 BTC, 300 ETH)
- `useFixedQuantity: true` — quantity-based tracking
- Loaded via `POST :8082/addAllocationDriftModel`

### EarlySettlementModel
Models accelerated AR collection or AP payment with discount:
- `discountFunctionType` — `LINEAR`, `EXPONENTIAL`, or `STEPWISE`
- `hurdleRateAnnualized` — minimum return threshold for settlement decision
- `buyerCashMOC` — links to counterparty liquidity reference index
- Loaded via `POST :8082/addEarlySettlementModel`

### PenaltyAccrualModel
Escalating penalties on overdue AP invoices:
- `penaltyStepSchedule` — e.g., `{"0": 0.0005, "16": 0.00075, "31": 0.001}`
- Loaded via `POST :8082/addPenaltyAccrualModel`

### ScheduledCashFlowModel
Drives the CASH-PAM by injecting derived cash flows from Pass 1:
- 7 models: `scf_deployments`, `scf_btc_proceeds`, `scf_eth_proceeds`, `scf_tbill_flows`, `scf_ar_inflows`, `scf_ap_outflows`, `scf_usdc_flows`
- Loaded via `POST :8082/addScheduledCashFlowModel`

### LiquidityBufferModel
Ensures minimum cash reserves:
- `minBufferUSD: 150000` / `targetBufferUSD: 500000`
- Loaded via `POST :8082/addLiquidityBufferModel`

---

## Consolidated Scenario (28 Descriptors)

The full scenario binds: 13 ReferenceIndex (6 base + 7 derived) + 2 AllocationDriftModel + 4 EarlySettlement + 1 PenaltyAccrualModel + 7 ScheduledCashFlowModel + 1 LiquidityBufferModel.

---

## ACTUS Event Types

| Event | Meaning |
|---|---|
| `IED` | Initial Exchange Date — contract inception / capital deployment |
| `PP` | Prepayment — allocation drift rebalancing (CLM) or early settlement (AR/AP) |
| `IP` | Interest Payment — T-bill accretion, USDC yield |
| `RR` | Rate Reset — rate changes on floating instruments |
| `MD` | Maturity Date — contract close / principal return |

---

## Collection Library

**Domain:** `hybrid-treasury/` — 34 collections across 4 subcategories

- **Subcategory 1:** Core hybrid treasury — individual allocation drift, liquidity buffer, multi-contract, mirror/passthrough, fair value compliance, peg stress, yield arbitrage
- **Subcategory 2:** Central cash pool with CLM mirror and dynamic discounting — time-epoch bi-weekly evaluation
- **Subcategory 3:** Adaptive cash + digital asset CLM configurations
- **Subcategory 4:** Fully consolidated multi-asset-class — the HT-CONS series ($10M portfolios, 15 contracts, 28 scenario descriptors, two-pass architecture)

---

