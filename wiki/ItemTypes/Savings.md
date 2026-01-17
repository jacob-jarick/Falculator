# Savings ItemType

Represents savings accounts, deposits, and emergency funds with compound interest.

Example configs: "Data/examples/001.Test.json", "Data/examples/HomeLoan.json"

---

## Overview

ItemType.Savings represents accounts that accumulate value over time with interest. Typical examples:
- **Bank savings accounts**: High-yield savings, checking accounts
- **Emergency funds**: Dedicated reserves for unexpected expenses
- **Deposit accounts**: Term deposits, CDs
- **Main savings account**: Central cash flow hub (IsMainSavingsAccount=true)

**Key characteristics:**
- Typically has positive Value (balance)
- Interest enabled for compound growth
- CashIn/CashOut optional (full flexibility)
- IsLiquidAsset typically true
- No ItemType-specific sanitization enforcement (except main savings account)

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Savings`
3. Set `Value` to starting balance
4. Set `IsLiquidAsset` to `true`
5. Set `StartEnabled` to `true`
6. Configure `Interest` for compound interest

---

## Typical Configuration

### Value (Balance)

**Purpose:** Current account balance

**Typical patterns:**
- Starting balance: $10,000, $50,000, etc.
- Updated by Interest each step
- Updated by CashIn/CashOut from other items

### Interest (Primary)

**Purpose:** Compound interest growth

**Typical patterns:**
- Annual rate with monthly compounding
- Fixed interest rate (e.g., 4.5% APY)
- Variable rates (via EventItem state changes)

**Configuration:**
```json
"Interest": {
  "Enabled": true,
  "Amount": 4.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

### CashIn (Optional)

**Purpose:** Direct deposits to this account

**Typical patterns:**
- Interest payments from other accounts
- Transfer from income (via EventItem more common)
- Percentage-based savings

**Note:** Most deposits flow through main savings account, not CashIn directly.

### CashOut (Optional)

**Purpose:** Withdrawals or fees

**Typical patterns:**
- Account maintenance fees
- Withdrawal penalties
- Transfers to other accounts (via EventItem more common)

---

## Special Behavior

### Main Savings Account

**If IsMainSavingsAccount=true**, strict sanitization is enforced:
- Type must be Savings
- EvalOrder must be 0 (evaluated first)
- StartEnabled must be true
- DisabledByUser must be false
- IsLiquidAsset must be true
- EndDate must be >= 95 years from now

**See:** FinancialItem.md for main savings account invariants

### No Other Enforcement

Regular Savings items (IsMainSavingsAccount=false) have no ItemType-specific rules. Full flexibility.

---

## Sanitization Rules

**Standard FinancialItem sanitization only** (unless IsMainSavingsAccount=true):
- CashIn, CashOut, Interest initialized if null
- PercentageBasis enforced to Self for all AmountFreqs
- Interest.IsPercentage enforced to true
- Name auto-generated if empty

---

## EventItem Support

**Supported.** Savings items can use EventItems for:
- **Automatic transfers**: "Transfer $500 from checking to savings monthly"
- **Emergency fund top-ups**: "Transfer to emergency fund when balance < $10k"
- **Investment rebalancing**: "Transfer excess savings to investments"

**Example:**
```json
"Events": [
  {
    "Name": "Investment Transfer",
    "TargetName": "Stock Portfolio",
    "CashOut": {
      "Enabled": true,
      "Amount": 1000,
      "Schedule": { "Frequency": "Monthly" }
    },
    "Triggers": {
      "MainSavingsBalance": {
        "Enabled": true,
        "Operator": "GreaterThanOrEqual",
        "ComparisonValue": 20000
      }
    }
  }
]
```

Transfers $1,000 to stocks monthly when savings >= $20k.

---

## Example Configuration

### Basic Savings Account with Interest

```json
{
  "Name": "High-Yield Savings",
  "Type": "Savings",
  "Value": 10000,
  "IsLiquidAsset": true,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 5.0,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

### Main Savings Account

```json
{
  "Name": "Main Savings Account",
  "Type": "Savings",
  "Value": 50000,
  "IsMainSavingsAccount": true,
  "IsLiquidAsset": true,
  "StartEnabled": true,
  "EvalOrder": 0,
  "Interest": {
    "Enabled": true,
    "Amount": 4.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

### Emergency Fund

```json
{
  "Name": "Emergency Fund",
  "Type": "Savings",
  "Value": 10000,
  "IsLiquidAsset": true,
  "StartEnabled": true,
  "Tags": ["emergency", "liquid"],
  "Interest": {
    "Enabled": true,
    "Amount": 3.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

---

## Common Patterns

### Pattern 1: Compound Interest Demonstration

**See:** Data/examples/001.Test.json

```json
{
  "Type": "Savings",
  "Value": 10000,
  "Interest": {
    "Amount": 5.0,
    "AnnualRateMonthlyCompounding": true
  }
}
```

After 1 year: $10,511.62 (5% APY compounded monthly)

### Pattern 2: Savings with Account Fees

```json
{
  "Type": "Savings",
  "Value": 5000,
  "Interest": { "Amount": 2.0, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 5, "Schedule": { "Frequency": "Monthly" } }
}
```

Earns 2% interest, pays $5 monthly fee.

### Pattern 3: Conditional Interest Rate

```json
{
  "Type": "Savings",
  "Value": 10000,
  "Interest": { "Amount": 3.0, "AnnualRateMonthlyCompounding": true },
  "Events": [
    {
      "Name": "Bonus Interest",
      "TargetName": "Savings Account",
      "SetStateOnTrigger": true,
      "TargetStateOnTrigger": "Enable",
      "Triggers": {
        "MainSavingsBalance": {
          "Operator": "GreaterThanOrEqual",
          "ComparisonValue": 50000
        }
      }
    }
  ]
}
```

Activates bonus interest tier when balance >= $50k (requires separate high-interest item).

### Pattern 4: Automatic Emergency Fund Top-Up

Configured on Income item:

```json
{
  "Name": "Salary",
  "Type": "Income",
  "Events": [
    {
      "Name": "Top Up Emergency Fund",
      "TargetName": "Emergency Fund",
      "CashOut": { "Amount": 500, "Schedule": { "Frequency": "Monthly" } },
      "Triggers": {
        "TargetBalanceTrigger": {
          "Operator": "LessThan",
          "ComparisonValue": 10000
        }
      }
    }
  ]
}
```

Transfers $500 monthly to emergency fund when balance < $10k.

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties, main savings account
- amountFreq.md - Interest calculation with compounding
- TriggerConditions.md - Conditional activation
- EventItem.md - Inter-account transfers

**ItemTypes:**
- Income.md - Flows CashIn to savings
- Expense.md - Flows CashOut from savings
- Asset.md - Alternative for long-term savings
- Shares.md - Investment alternative to savings

**Examples:**
- Data/examples/001.Test.json - Compound interest demonstration
- Data/examples/HomeLoan.json - Main savings account with salary deposits

---

## Implementation Details

**No ItemType-specific code** (except main savings account enforcement):

**Source file:** `Models/FinancialItem.cs`

**Main savings sanitization:** Lines 684-731 (enforces invariants when IsMainSavingsAccount=true)

**Standard sanitization:** Lines 446-683 (standard FinancialItem.Sanitize())

**Evaluation:** Standard trigger evaluation (EvaluateSelfTrigger)

**Processing:**
- SimFrame.CalculateItem() - Interest applied to Value
- Cash flow sweep - CashIn/CashOut from all items flows through main savings account

---
