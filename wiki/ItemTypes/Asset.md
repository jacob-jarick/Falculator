# Asset ItemType

Represents property, vehicles, and other assets with complex cash flows.

Example configs: "Data/examples/HomeLoan.json" (Primary House), "Data/examples/006.Test.json"

---

## Overview

ItemType.Asset represents physical or financial assets with value appreciation/depreciation. Typical examples:
- **Real estate**: Rental properties, primary residence
- **Vehicles**: Cars, boats, motorcycles
- **Collectibles**: Art, antiques, precious metals
- **Equipment**: Business assets, machinery

**Key characteristics:**
- Value represents current market value
- CashIn for income generated (rental income, usage fees)
- CashOut for expenses (maintenance, property taxes, insurance)
- Interest for appreciation/depreciation (positive or negative)
- **Most flexible ItemType** - can use all three cash flows simultaneously
- No ItemType-specific sanitization enforcement

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Asset`
3. Set `Value` to current market value
4. Configure `CashIn` for income (optional)
5. Configure `CashOut` for expenses (optional)
6. Configure `Interest` for appreciation/depreciation (optional)

---

## Typical Configuration

### Value (Market Value)

**Purpose:** Current asset valuation

**Typical patterns:**
- Positive for owned assets (e.g., 500000 for $500k property)
- Can be negative for leveraged assets with loans exceeding value
- Updated by Interest each step (appreciation/depreciation)

### CashIn (Income Generated)

**Purpose:** Revenue from asset

**Typical patterns:**
- Rental income from properties
- Lease payments from vehicles
- Usage fees
- Asset-backed income streams

**Configuration:**
```json
"CashIn": {
  "Enabled": true,
  "Amount": 2000,
  "Schedule": { "Frequency": "Monthly", "DayOfMonth": 1 }
}
```

### CashOut (Operating Expenses)

**Purpose:** Costs to maintain/operate asset

**Typical patterns:**
- Property taxes and insurance
- Maintenance and repairs
- HOA fees
- Vehicle registration and insurance

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 500,
  "Schedule": { "Frequency": "Monthly" }
}
```

### Interest (Appreciation/Depreciation)

**Purpose:** Change in asset value over time

**Typical patterns:**
- Positive for appreciating assets (real estate growth)
- Negative for depreciating assets (vehicle depreciation)
- Can model loan interest against asset value

**Configuration (Appreciation):**
```json
"Interest": {
  "Enabled": true,
  "Amount": 3.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

**Configuration (Depreciation):**
```json
"Interest": {
  "Enabled": true,
  "Amount": -15.0,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

---

## Special Behavior

**None.** Asset type has no ItemType-specific enforcement. Full flexibility with all cash flow properties.

**Liquidation support:** Assets can be liquidated via:
- EventItem.Liquidate (target asset for conditional liquidation)
- FinancialItem.LiquidateSelfOnTrigger (self-liquidation when trigger met)

---

## Sanitization Rules

**No ItemType-specific sanitization rules.** Standard FinancialItem sanitization applies:
- CashIn, CashOut, Interest initialized if null
- PercentageBasis enforced to Self for all AmountFreqs
- Interest.IsPercentage enforced to true
- Name auto-generated if empty

---

## EventItem Support

**Fully supported.** Asset items can use EventItems for:
- **Liquidation**: "Sell property when value exceeds $600k"
- **Conditional income**: "Enable rental income when property active"
- **Variable expenses**: "Pay maintenance based on property value"
- **Acquisition**: Purchase assets when conditions met

**Example (Liquidation):**
```json
"Events": [
  {
    "Name": "Sell Property",
    "TargetName": "Rental Property",
    "Liquidate": true,
    "Triggers": {
      "Age": {
        "Enabled": true,
        "Operator": "GreaterThanOrEqual",
        "ComparisonValue": 70,
        "TriggerLimit": 1
      }
    }
  }
]
```

Liquidates rental property at age 70.

---

## Example Configuration

### Rental Property (Full Cash Flows)

**See:** Data/examples/HomeLoan.json (Primary House)

```json
{
  "Name": "Rental Property",
  "Type": "Asset",
  "Value": 500000,
  "IsLiquidAsset": false,
  "StartEnabled": true,
  "Tags": ["property", "rental", "income"],
  "CashIn": {
    "Enabled": true,
    "Amount": 2000,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 1 }
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 500,
    "Schedule": { "Frequency": "Monthly" }
  },
  "Interest": {
    "Enabled": true,
    "Amount": 3.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

- Value: $500,000 (appreciates 3.5% annually)
- CashIn: $2,000/month rental income
- CashOut: $500/month expenses (taxes, insurance, maintenance)
- Net cash flow: $1,500/month + appreciation

### Depreciating Vehicle

```json
{
  "Name": "Car",
  "Type": "Asset",
  "Value": 30000,
  "IsLiquidAsset": false,
  "StartEnabled": true,
  "CashOut": {
    "Enabled": true,
    "Amount": 300,
    "Schedule": { "Frequency": "Monthly" }
  },
  "Interest": {
    "Enabled": true,
    "Amount": -15.0,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

- Value: $30,000 (depreciates 15% annually)
- CashOut: $300/month (insurance, maintenance)
- After 1 year: ~$25,500 value, $3,600 expenses

### Primary Residence (No Cash Flows)

```json
{
  "Name": "Primary House",
  "Type": "Asset",
  "Value": 600000,
  "IsLiquidAsset": false,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 4.0,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

Simple appreciation model, no rental income or expenses.

### Leveraged Asset with Loan

```json
{
  "Name": "Investment Property",
  "Type": "Asset",
  "Value": 400000,
  "StartEnabled": true,
  "CashIn": { "Amount": 2500, "Schedule": { "Frequency": "Monthly" } },
  "CashOut": { "Amount": 800, "Schedule": { "Frequency": "Monthly" } },
  "Interest": {
    "Enabled": true,
    "Amount": -5.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

**Note:** Interest represents loan interest against property value (negative). Separate Loan item handles principal repayment.

---

## Common Patterns

### Pattern 1: Appreciating Real Estate

```json
{
  "Type": "Asset",
  "Value": 500000,
  "Interest": { "Amount": 5.0, "AnnualRateMonthlyCompounding": true }
}
```

### Pattern 2: Rental Property Cash Flow

```json
{
  "Type": "Asset",
  "Value": 300000,
  "CashIn": { "Amount": 1800, "Schedule": { "Frequency": "Monthly" } },
  "CashOut": { "Amount": 600, "Schedule": { "Frequency": "Monthly" } },
  "Interest": { "Amount": 4.0, "AnnualRateMonthlyCompounding": true }
}
```

Net: $1,200/month + 4% appreciation.

### Pattern 3: Liquidation on Trigger

```json
{
  "Type": "Asset",
  "Value": 500000,
  "LiquidateSelfOnTrigger": true,
  "SelfTrigger": {
    "Age": { "Operator": ">=", "ComparisonValue": 70, "TriggerLimit": 1 }
  }
}
```

Sells asset at age 70, transfers proceeds to main savings.

### Pattern 4: Conditional Purchase

Configured on Income item:

```json
{
  "Name": "Salary",
  "Type": "Income",
  "Events": [
    {
      "Name": "Buy Car",
      "TargetName": "Car",
      "SetStateOnTrigger": true,
      "TargetStateOnTrigger": "Enable",
      "CashOut": { "Amount": 30000, "Schedule": { "TriggerLimit": 1 } },
      "Triggers": {
        "Age": { "Operator": ">=", "ComparisonValue": 25 },
        "MainSavingsBalance": { "Operator": ">=", "ComparisonValue": 50000 }
      }
    }
  ]
}
```

Purchases $30k car at age 25 when savings >= $50k.

### Pattern 5: Complex Property with Loan

**Asset:**
```json
{
  "Name": "Primary House",
  "Type": "Asset",
  "Value": 500000,
  "Interest": { "Amount": 5.0, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 800, "Schedule": { "Frequency": "Monthly" } }
}
```

**Loan (separate item):**
```json
{
  "Name": "Mortgage",
  "Type": "Loan",
  "Value": -400000,
  "Interest": { "Amount": 6.5, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 2500, "Schedule": { "Frequency": "Monthly" } }
}
```

Asset appreciates 5%, loan accrues 6.5% interest. Net equity grows over time.

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties, liquidation
- amountFreq.md - CashIn, CashOut, Interest calculation
- TriggerConditions.md - Conditional activation
- EventItem.md - Liquidation, conditional purchases

**ItemTypes:**
- Income.md - Can generate CashIn (alternative to Asset rental income)
- Expense.md - Can represent CashOut (alternative to Asset expenses)
- Loan.md - Often paired with Asset for leveraged purchases
- Shares.md - Alternative investment asset type

**Examples:**
- Data/examples/HomeLoan.json - Primary House asset with appreciation
- Data/examples/006.Test.json - Complex trigger scenarios with assets

---

## Implementation Details

**No ItemType-specific code.** Asset uses standard FinancialItem behavior:

**Source file:** `Models/FinancialItem.cs`

**Sanitization:** Lines 446-731 (standard FinancialItem.Sanitize(), no Asset-specific rules)

**Liquidation:** Supported via EventItem.Liquidate and FinancialItem.LiquidateSelfOnTrigger

**Processing:**
- SimFrame.CalculateItem() - Interest applied to Value, CashIn/CashOut processed
- Cash flow sweep - CashIn/CashOut flows through main savings account

---
