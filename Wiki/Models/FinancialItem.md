# FinancialItem Model

Core financial entity representing income, expenses, savings, assets, liabilities, loans, shares, and credit cards.

---

## Overview

FinancialItem is the fundamental building block of the Falculator simulation. It represents any financial entity with:
- **Value**: Current balance, asset value, or debt amount
- **Cash Flows**: CashIn (income/revenue) and CashOut (expenses/costs)
- **Interest**: Growth or decay applied to Value
- **Lifecycle**: Activation triggers, start/end dates, enabled state
- **Relationships**: EventItems for transfers, SelfTrigger for conditional activation
- **Classification**: ItemType determines behavioral patterns (not enforced restrictions)

**Supported ItemTypes:**
- Income, Expense, Savings, Asset, Liability, Loan, Shares, CreditCard

**Key Features:**
- Full cash flow flexibility (CashIn, CashOut, Interest configurable on all types)
- Trigger-based activation (age, balance, tags, dates)
- Inter-item transfers via EventItems
- ItemType-specific sanitization for Shares and CreditCard
- Main savings account designation for cash flow sweep
- Tag-based grouping for conditional logic

---

## Quick Start

### Basic Savings Account

```json
{
  "Name": "Savings Account",
  "Type": "Savings",
  "Value": 10000,
  "IsLiquidAsset": true,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 4.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

### Salary Income

```json
{
  "Name": "Salary",
  "Type": "Income",
  "StartEnabled": true,
  "CashIn": {
    "Enabled": true,
    "Amount": 5000,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 15 }
  }
}
```

### Mortgage Loan

```json
{
  "Name": "Home Loan",
  "Type": "Loan",
  "Value": -400000,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 6.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 2500,
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

---

## Property Categories

### 1. Identity Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Id | string | auto-generated | 8-character unique identifier (read-only) |
| Name | string | "" | Human-readable name |
| Description | string | "" | Optional purpose or role description |
| Tags | BindingList<string> | [] | Tag list for grouping and conditional logic |

### 2. Configuration Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Type | ItemType | Savings | Classification (Income, Expense, Savings, Asset, Liability, Loan, Shares, CreditCard) |
| IsMainSavingsAccount | bool | false | Designates the main savings account for cash flow sweep |
| IsLiquidAsset | bool | false | True for easily liquidated assets (savings, shares) |
| Currency | string | "AUD" | Currency code (currently only AUD supported) |
| EvalOrder | int | 0 | Lower numbers evaluated first, higher numbers last |
| IncludeInResults | bool | true | If false, hidden from graphs/CSV but still calculated |

### 3. Financial Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Value | decimal | 0 | Current balance/value (negative for debts) |
| CashIn | AmountFreq | disabled | Money flowing TO user (see amountFreq.md) |
| CashOut | AmountFreq | disabled | Money flowing FROM user (see amountFreq.md) |
| Interest | AmountFreq | disabled | Applied to Value (growth/decay) |
| ShareDetails | ShareDetails? | null | Share-specific data (only for Shares type) |

### 4. Lifecycle Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| StartEnabled | bool | false | Initial state at simulation start |
| DisabledByUser | bool | false | Exclude from simulation entirely (overrides all triggers) |
| EnabledBySim | bool | false | Current state during simulation (read-only) |
| StartDate | DateOnly | now | Item activates on or after this date |
| EndDate | DateOnly | now + 100 years | Item deactivates after this date |

### 5. Relationship Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Events | List<EventItem> | [] | Payments/transfers to other items (see EventItem.md) |
| SelfTrigger | TriggerConditions | empty | Conditional activation logic (see TriggerConditions.md) |
| LiquidateSelfOnTrigger | bool | false | Liquidate when SelfTrigger conditions met (Asset type only) |

---

## ItemType Enum

| Value | Numeric | Typical Usage |
|-------|---------|---------------|
| Income | 0 | Salary, rental income, dividends |
| Savings | 1 | Bank accounts, deposits, emergency funds |
| Asset | 2 | Property, vehicles, collectibles |
| Liability | 3 | Debts without special logic |
| Expense | 4 | Rent, subscriptions, recurring costs |
| Loan | 5 | Mortgages, auto loans (auto-disables when paid off) |
| Shares | 6 | Stock/ETF holdings (unit-based valuation) |
| CreditCard | 7 | Credit card balances (always-enabled, strict sanitization) |

**Important:** ItemType is a classification, NOT a restriction. All types have full cash flow flexibility. See ItemTypes/ documentation for typical patterns.

---

## Cash Flow Semantics

### CashIn (Money TO User)

Examples: Salary, rental income, dividends, capital gains

**Typical ItemTypes:** Income, Asset (rental properties), Shares (dividends)

**Flow:** CashIn amount is deposited to main savings account

**Configuration:**
```json
"CashIn": {
  "Enabled": true,
  "Amount": 5000,
  "IsPercentage": false,
  "Schedule": { "Frequency": "Monthly" }
}
```

### CashOut (Money FROM User)

Examples: Rent, loan repayments, property expenses, subscription fees

**Typical ItemTypes:** Expense, Loan, Asset (maintenance costs)

**Flow:** CashOut amount is deducted from main savings account

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 2000,
  "IsPercentage": false,
  "Schedule": { "Frequency": "Monthly" }
}
```

### Interest (Applied to Value)

Examples: Investment returns, loan interest, credit card APR, asset depreciation

**Typical ItemTypes:** Savings, Loan, CreditCard, Asset, Shares

**Flow:** Interest is applied to the item's Value property (increases or decreases balance)

**Configuration:**
```json
"Interest": {
  "Enabled": true,
  "Amount": 6.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

**Note:** See amountFreq.md for detailed payment calculation logic.

---

## Main Savings Account

One FinancialItem must be designated as the main savings account (IsMainSavingsAccount=true).

**Purpose:** Central cash flow hub where all CashIn/CashOut flows are settled.

**Invariants (enforced by Sanitize):**
- Type must be Savings
- EvalOrder must be 0 (evaluated first)
- StartEnabled must be true (always starts active)
- DisabledByUser must be false (cannot be disabled)
- IsLiquidAsset must be true (is liquid)
- EndDate must be >= 95 years from now

**Configuration:**
```json
{
  "Name": "Main Savings Account",
  "Type": "Savings",
  "IsMainSavingsAccount": true,
  "IsLiquidAsset": true,
  "StartEnabled": true,
  "Value": 10000
}
```

**Creation:** Use FinancialItem.CreateDefaultSavingsAccount() for proper initialization.

---

## ItemType-Specific Behavior

### Shares ItemType

**Special characteristics:**
- Requires ShareDetails property (UnitCount, UnitPrice, TotalCostBase)
- Value calculated as UnitCount * UnitPrice (synced automatically)
- CashOut.Enabled = false (enforced)
- CashIn must be percentage-based if enabled (dividends)
- Interest represents annual growth applied to UnitPrice

**Sanitization enforcement:**
- ShareDetails created if null
- CashOut disabled
- CashIn.IsPercentage = true
- Value synced from ShareDetails.CalculateValue()
- If DisabledByUser=true, Value=0

**See:** Shares.md for full documentation

### CreditCard ItemType

**Special characteristics:**
- Always enabled (cannot be disabled)
- Interest always enabled with strict configuration
- Value cannot be negative (no overpayment)
- Not a liquid asset

**Sanitization enforcement:**
- Value >= 0 (no positive balance)
- Interest.Enabled = true
- Interest.Amount >= 0
- Interest.IsPercentage = true
- Interest.AnnualRateMonthlyCompounding = true
- Interest.Schedule.Frequency = Monthly
- Interest.Schedule.DayOfMonth = 31 (last day)
- DisabledByUser = false
- StartEnabled = true
- IsLiquidAsset = false
- SelfTrigger cleared (no triggers allowed)

**See:** creditCard.md for full documentation

### Loan ItemType

**Special characteristics:**
- Auto-disables when Value = 0 (paid off)
- Otherwise behaves like standard liability

**Evaluation logic:**
- If Value == 0: EnabledBySim = false (disabled)
- If Value != 0 and no SelfTrigger: EnabledBySim = true (active)
- If Value != 0 and SelfTrigger exists: uses trigger evaluation

**See:** Loan.md for full documentation

### Other ItemTypes

Income, Expense, Savings, Asset, Liability: No ItemType-specific enforcement. Full flexibility with cash flows.

**See ItemTypes/ documentation for typical patterns.**

---

## Sanitization Rules

Sanitize() is called automatically during config load and before simulation start.

### Identity Sanitization

- **Id registration:** Registers ID with IdGenerator, checks for duplicates
- **Name generation:** If Name is empty, sets to `"FinancialItem-{Id}"`
- **Currency enforcement:** Sets to "AUD" if empty or non-AUD

### AmountFreq Initialization

- CashIn, CashOut, Interest created if null (disabled defaults)
- Calls Sanitize() on all three AmountFreqs
- Enforces PercentageBasis = Self for all three
- Enforces Interest.IsPercentage = true

### ItemType-Specific Sanitization

- **Shares:** Extensive enforcement (see above and Shares.md)
- **CreditCard:** Extensive enforcement (see above and creditCard.md)
- **Other types:** No enforcement

### Main Savings Account Sanitization

If IsMainSavingsAccount=true:
- Type = Savings
- DisabledByUser = false
- EvalOrder = 0
- StartEnabled = true
- IsLiquidAsset = true
- EndDate >= now + 95 years

### Duplicate Tag Removal

- Removes duplicate strings from Tags list (case-sensitive)

---

## Trigger Evaluation

FinancialItem.EvaluateSelfTrigger() is called each simulation step to determine if the item should be active.

**Evaluation algorithm:**

1. **Check for trigger conditions:**
   - Type == Loan (always has implicit trigger)
   - OR SelfTrigger.HasAnyConditions() returns true

2. **If no triggers:** Use StartEnabled as default state

3. **Date range checks:**
   - If StartDate > simDate: EnabledBySim = false
   - If EndDate < simDate: EnabledBySim = false

4. **CreditCard special case:**
   - Always enabled (EnabledBySim = true)

5. **Loan special case:**
   - If Value == 0: EnabledBySim = false (paid off)
   - If Value != 0 and no explicit triggers: EnabledBySim = true

6. **Explicit trigger evaluation:**
   - Calls SelfTrigger.EvaluateTrigger()
   - Sets EnabledBySim based on result

**See:** TriggerConditions.md for full trigger evaluation logic

---

## Common Patterns

### Pattern 1: Simple Salary Income

```json
{
  "Name": "Salary",
  "Type": "Income",
  "StartEnabled": true,
  "CashIn": {
    "Enabled": true,
    "Amount": 5000,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 15 }
  }
}
```

### Pattern 2: Savings with Interest

```json
{
  "Name": "High-Yield Savings",
  "Type": "Savings",
  "Value": 50000,
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

### Pattern 3: Rental Property (Asset with Cash Flows)

```json
{
  "Name": "Rental Property",
  "Type": "Asset",
  "Value": 500000,
  "StartEnabled": true,
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
    "Amount": -3.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  }
}
```

Rental income (CashIn), property expenses (CashOut), and loan interest against value (negative Interest).

### Pattern 4: Age-Triggered Pension

```json
{
  "Name": "Pension",
  "Type": "Income",
  "CashIn": {
    "Enabled": true,
    "Amount": 3000,
    "Schedule": { "Frequency": "Monthly" }
  },
  "SelfTrigger": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 67
    }
  }
}
```

Activates when user reaches 67 years old.

### Pattern 5: Conditional Investment Purchase

```json
{
  "Name": "Stock Portfolio",
  "Type": "Shares",
  "ShareDetails": {
    "Name": "VAS",
    "UnitCount": 0,
    "UnitPrice": 95.50
  },
  "Interest": {
    "Enabled": true,
    "Amount": 7.0,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "Events": [
    {
      "Name": "Monthly Investment",
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
}
```

Purchases $1,000 of shares monthly when savings balance >= $20k.

---

## Debugging

### ToString() Format

```
Name - Currency Value (Liquid/Illiquid)
```

Example: `Savings Account - AUD 10,000.00 (Liquid)`

### Debug Logging

FinancialItem operations are logged at DEBUG level:

```
[FinancialItem-fi123456] FinancialItem created
[FinancialItem-fi123456] Sanitize: CashIn was null, initialized to default
[FinancialItem-fi123456] Item not yet started (StartDate 2030-01-01 > simDate 2026-01-17)
[FinancialItem-fi123456] Loan paid off (Value = 0.00), disabling item
```

**Enable debug logs:**
```bash
Falculator.exe --config "config.json" --run --loglevel DEBUG
```

---

## Related Models

- **AmountFreq** - See amountFreq.md for CashIn, CashOut, Interest
- **TriggerConditions** - See TriggerConditions.md for SelfTrigger logic
- **EventItem** - See EventItem.md for inter-item transfers
- **ShareDetails** - See Shares.md for Shares ItemType
- **Config** - Root container with MainSavingsAccountId property

**ItemType Documentation:**
- Income.md, Expense.md, Savings.md, Asset.md, Liability.md, Loan.md, Shares.md, creditCard.md

---

## JSON Schema Reference

Full schema: `Data/Schema/FinancialItem.schema.json`

**Required properties:**
- Id (auto-generated)
- Type (defaults to Savings)
- CashIn, CashOut, Interest (AmountFreq objects, initialized if missing)
- SelfTrigger (TriggerConditions object)
- Events (List<EventItem>)

**Optional properties:**
- Name (auto-generated if missing)
- Description, Tags
- Value (defaults to 0)
- IsMainSavingsAccount (defaults to false)
- IsLiquidAsset (defaults to false)
- ShareDetails (only for Shares type)

---

## Example Configurations

FinancialItem is used in all example configs:

- **Data/examples/HomeLoan.json** - Comprehensive example (salary, savings, loan, asset, events)
- **Data/examples/001.Test.json** - Compound interest demonstration
- **Data/examples/InteractiveMode.Test.json** - CreditCard example
- **Data/examples/Shares.Test.json** - Shares example
- **Data/examples/006.Test.json** - Complex trigger scenarios

---

## Implementation Details

**Source file:** `Models/FinancialItem.cs`

**Key methods:**
- `CreateDefaultSavingsAccount()` - Lines 126-167: Creates default main savings account
- `EvaluateSelfTrigger()` - Lines 365-434: Trigger evaluation and state management
- `Sanitize()` - Lines 446-731: Validation, enforcement, child sanitization
- `ToString()` - Lines 352-353: Display formatting

**Processing:**
- Evaluated in `SimFrame.CalculateItem()` based on EvalOrder
- Triggers evaluated before each calculation step
- Cash flows swept to/from main savings account

**Dependencies:**
- `AmountFreq` - CashIn, CashOut, Interest properties
- `TriggerConditions` - SelfTrigger property
- `EventItem` - Events list
- `ShareDetails` - Shares type property
- `ItemType` enum (FinancialItem.cs) - Classification
- `IdGenerator` - ID generation and registration
- `DebugLogger` - Logging infrastructure

**Used by:**
- `Config.FinancialItems` - List of all FinancialItem objects in simulation
- `SimFrame` - Simulation step processing

---
