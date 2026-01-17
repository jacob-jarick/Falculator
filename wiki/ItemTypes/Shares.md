# Shares Item Type

Unitized financial items for modeling stock/ETF holdings. Value = UnitCount x UnitPrice.

Example config: "Data/examples/Shares.Test.json"

---

## Overview

ItemType.Shares represents stock or ETF holdings where:
- **Value** is calculated as `UnitCount * UnitPrice`
- **Interest** represents annual growth (applied to UnitPrice per step)
- **CashIn** represents dividend yield (percentage-based)
- **Liquidation** sells all shares, transferring proceeds to main savings

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Shares`
3. Configure `ShareDetails`:
   - `UnitCount` - Number of shares held
   - `UnitPrice` - Current price per share
4. Configure `Interest` for annual growth rate (e.g., 7% for market returns)
5. Optionally configure `CashIn` for dividend yield (e.g., 2%)

---

## ShareDetails Properties

| Property | Type | Description |
|----------|------|-------------|
| Id | string | Auto-generated 8-char identifier |
| Name | string | Share name (e.g., "VAS", "VDHG") |
| UnitCount | decimal | Number of shares (supports fractions) |
| UnitPrice | decimal | Current market price per share |
| TotalCostBase | decimal | Purchase cost for future CGT (tracked, not used) |

---

## Calculation Logic

### Value Calculation
```
Value = UnitCount * UnitPrice
```

### Interest (Growth)
- Treated as annual rate, compounded per simulation step
- Applied to UnitPrice, not Value directly
- Formula per step: `UnitPrice *= (1 + AnnualRate)^(1/StepsPerYear)`

### Dividends (CashIn)
- Must be percentage-based (enforced)
- Calculated as percentage of Value
- Flows to main savings via CashFlow sweep

---

## Sanitization Rules

When Type is Shares, the following are enforced:
- ShareDetails must exist (auto-created if null)
- ShareDetails.UnitCount >= 0
- CashOut.Enabled = false (shares don't consume funds)
- Interest.IsPercentage = true
- Interest.PercentageBasis = Self
- CashIn.IsPercentage = true (if enabled)
- If DisabledByUser: Value = 0

---

## EventItem Support

### Buying Shares (CashOut to Shares target)
- Calculates shares affordable: `Floor(Amount / UnitPrice)`
- Purchases whole shares, leftover cash stays in source
- Updates TotalCostBase for CGT tracking

### Selling Shares (CashOut from Shares source)
- Calculates shares to sell: `Ceiling(Amount / UnitPrice)`
- Capped at available shares
- Sale proceeds transferred to target

### Liquidation
- Sells all shares at current UnitPrice
- Proceeds transferred to main savings
- Item disabled for remainder of simulation

---

## Example Configuration

```json
{
  "Name": "Index Fund",
  "Type": "Shares",
  "ShareDetails": {
    "Name": "VAS",
    "UnitCount": 100,
    "UnitPrice": 95.50,
    "TotalCostBase": 9000
  },
  "Interest": {
    "Enabled": true,
    "Amount": 7.0,
    "IsPercentage": true
  },
  "CashIn": {
    "Enabled": true,
    "Amount": 3.5,
    "IsPercentage": true,
    "Schedule": { "Frequency": "Annual" }
  },
  "StartEnabled": true,
  "IsLiquidAsset": true
}
```

---

## Implementation Details

**Code Location:**
- `Models/ShareDetails.cs` - Share data class
- `Models/FinancialItem.cs` - ShareDetails property, Sanitize rules
- `Models/SimFrame.cs` - Shares-specific calculation logic

---
