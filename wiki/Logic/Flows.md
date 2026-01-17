# Financial Item Cash Flow Model

High-level overview of cash flow semantics and ItemType patterns in Falculator.

---

## Overview

Falculator uses a flexible cash flow model where money flows TO and FROM the user through various financial instruments. Each FinancialItem has three cash flow properties:
- **CashIn**: Money TO user
- **CashOut**: Money FROM user
- **Interest**: Applied to item Value

**Important:** Cash flow patterns are NOT enforced. Users have full flexibility to configure any combination.

---

## Cash Flow Semantics

### CashIn (Money TO User)

**Purpose:** Revenue, income, or payments received

**Mechanics:**
- Flows TO main savings account
- Increases main savings balance
- Examples: Salary, rental income, dividends, cashback

**Typical ItemTypes:** Income, Asset (rental properties), Shares (dividends), Savings (interest payments)

**See:** [AmountFreq documentation](models/amountFreq.md) for payment calculation details

### CashOut (Money FROM User)

**Purpose:** Expenses, costs, or payments made

**Mechanics:**
- Flows FROM main savings account
- Decreases main savings balance
- Examples: Rent, loan repayments, utility bills, subscriptions

**Typical ItemTypes:** Expense, Loan, Liability, CreditCard, Asset (maintenance costs)

**See:** [AmountFreq documentation](models/amountFreq.md) for payment calculation details

### Interest (Applied to Value)

**Purpose:** Growth, decay, or interest on item balance

**Mechanics:**
- Applied directly to item's Value property
- Can be positive (appreciation, savings interest) or negative (depreciation, loan interest)
- Does NOT flow through main savings account
- Examples: Savings interest, loan interest, property appreciation, vehicle depreciation

**Typical ItemTypes:** Savings, Loan, CreditCard, Asset, Shares

**See:** [AmountFreq documentation](models/amountFreq.md) for compound interest calculations

---

## ItemType Overview

**Reminder:** These are typical patterns, NOT enforced restrictions. All ItemTypes have full flexibility except Shares and CreditCard.

| ItemType | Typical CashIn | Typical CashOut | Typical Interest | Auto Behavior |
|----------|----------------|-----------------|------------------|---------------|
| Income | YES (salary) | NO | NO | None |
| Expense | NO | YES (bills) | NO | None |
| Savings | Optional | Optional | YES (compound interest) | Main savings account enforcement |
| Asset | YES (rental income) | YES (maintenance) | YES (appreciation/depreciation) | Liquidation support |
| Liability | NO | YES (payments) | YES (interest) | None |
| Loan | NO | YES (repayments) | YES (interest) | Auto-disables when paid off |
| Shares | YES (dividends) | NO (enforced) | YES (growth) | Unit-based valuation, CashOut disabled |
| CreditCard | Optional (cashback) | YES (payments) | YES (always enforced) | Always-enabled, extensive sanitization |

**For detailed patterns and examples, see ItemType-specific documentation:**

- **[Income](ItemTypes/Income.md)** - Salary, freelance work, recurring income
- **[Expense](ItemTypes/Expense.md)** - Rent, subscriptions, recurring costs
- **[Savings](ItemTypes/Savings.md)** - Bank accounts, deposits, compound interest
- **[Asset](ItemTypes/Asset.md)** - Property, vehicles, collectibles (most flexible)
- **[Liability](ItemTypes/Liability.md)** - General debts without auto-disable
- **[Loan](ItemTypes/Loan.md)** - Mortgages, auto loans with auto-disable
- **[Shares](ItemTypes/Shares.md)** - Stock/ETF holdings, unit-based
- **[CreditCard](ItemTypes/creditCard.md)** - Credit card balances, always-enabled

---

## ItemType Enforcement Summary

### No Enforcement (Full Flexibility)

Most ItemTypes allow any combination of CashIn, CashOut, and Interest:
- Income
- Expense
- Savings (except main savings account)
- Asset
- Liability
- Loan (except auto-disable logic)

### ItemType-Specific Enforcement

**Shares:**
- CashOut.Enabled = false (no partial sales)
- CashIn must be percentage-based if enabled (dividends)
- ShareDetails required (UnitCount, UnitPrice)
- Value = UnitCount * UnitPrice (synced automatically)

**CreditCard:**
- Interest always enabled with strict configuration (APR, monthly compounding)
- Always enabled (cannot disable)
- Value cannot be negative (no overpayment)
- Not a liquid asset
- No self-triggers allowed

**Main Savings Account (Savings type, IsMainSavingsAccount=true):**
- Type must be Savings
- EvalOrder must be 0 (evaluated first)
- Always starts enabled
- Cannot be disabled by user
- Must be liquid asset

**See:** [FinancialItem documentation](models/FinancialItem.md) for sanitization details

---

## Common Patterns

### Pattern 1: Salary Income

```yaml
Type: Income
CashIn: $5000/month
CashOut: NO
Interest: NO
```

**See:** [Income documentation](ItemTypes/Income.md) for examples

### Pattern 2: Rental Property (Negatively Geared)

```yaml
Type: Asset
Value: $500,000 (property value)
CashIn: $2000/month (rent)
CashOut: $500/month (rates, insurance, maintenance)
Interest: -5.5% annual (mortgage interest on property value)
```

**Note:** Mortgage principal repayment modeled as separate Loan item.

**See:** [Asset documentation](ItemTypes/Asset.md) for complex examples

### Pattern 3: Mortgage Loan

```yaml
Type: Loan
Value: -$400,000 (debt amount, negative)
CashOut: $2500/month (repayments)
Interest: 6.5% annual (loan interest)
```

**Auto-disables when Value reaches 0 (paid off).**

**See:** [Loan documentation](ItemTypes/Loan.md) for auto-disable behavior

### Pattern 4: Savings with Compound Interest

```yaml
Type: Savings
Value: $10,000
CashIn: NO
CashOut: NO
Interest: 4.5% annual (monthly compounding)
```

**See:** [Savings documentation](ItemTypes/Savings.md) and [001.Test.json](Data/examples/001.Test.json)

### Pattern 5: Stock Portfolio

```yaml
Type: Shares
ShareDetails:
  UnitCount: 100
  UnitPrice: $95.50
CashIn: 3% (dividend yield, percentage-based)
Interest: 7% (capital growth)
```

**See:** [Shares documentation](ItemTypes/Shares.md) and [Shares.Test.json](Data/examples/Shares.Test.json)

---

## Important Design Principles

### 1. No Enforcement by Default

The `Sanitize()` method does NOT restrict cash flows based on ItemType (except Shares and CreditCard).

**Rationale:** Users need flexibility to model real-world scenarios that don't fit neat categories.

### 2. Typical != Required

The patterns shown are common usage, not hard rules.

**Example:** An Asset CAN have all three cash flows simultaneously (rental property with income, expenses, and mortgage interest).

### 3. Main Savings Account Hub

All CashIn and CashOut flows are settled through the main savings account (IsMainSavingsAccount=true).

**Mechanics:**
- CashIn: Added to main savings balance
- CashOut: Deducted from main savings balance
- Interest: Applied to individual item Value (does NOT flow through main savings)

**See:** [Config documentation](models/config.md) for main savings account requirements

### 4. EventItems for Complex Transfers

For transfers between items (not just to/from main savings), use EventItems.

**Examples:**
- "Transfer $1000 from salary to savings monthly"
- "Pay credit card from checking account"
- "Rebalance: move excess savings to investments"

**See:** [EventItem documentation](models/EventItem.md) for transfer mechanics

---

## Related Documentation

**Core Models:**
- [FinancialItem](models/FinancialItem.md) - Core properties and cash flow semantics
- [AmountFreq](models/amountFreq.md) - Payment and interest calculations
- [EventItem](models/EventItem.md) - Inter-item transfers
- [Config](models/config.md) - Main savings account configuration

**ItemType Documentation:**
- [ItemTypes Overview](ItemTypes/readMe.md) - All ItemType documentation index
- Individual ItemType docs (see table above)

**Examples:**
- Data/examples/HomeLoan.json - Comprehensive scenario with multiple ItemTypes
- Data/examples/001.Test.json - Compound interest demonstration
- Data/examples/Shares.Test.json - Shares ItemType example
- Data/examples/InteractiveMode.Test.json - CreditCard example

**Implementation:**
- Models/FinancialItem.cs - FinancialItem class and Sanitize() method
- Models/AmountFreq.cs - Payment processing logic
- Models/ItemType enum - ItemType definitions

---
