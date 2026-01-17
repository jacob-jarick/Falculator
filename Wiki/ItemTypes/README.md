# ItemType Documentation

Comprehensive documentation for all FinancialItem types and their behaviors.

---

## Overview

This directory contains detailed documentation for each ItemType in Falculator. ItemTypes define typical usage patterns and behaviors for financial entities, though most types have full flexibility with cash flows.

**Important:** ItemType is a classification, NOT a restriction. Most types allow full configuration of CashIn, CashOut, and Interest. Only Shares and CreditCard have ItemType-specific enforcement.

---

## ItemTypes

### Cash Flow Types

- **[Income](Income.md)** - Salary, freelance work, rental income, dividends
- **[Expense](Expense.md)** - Rent, subscriptions, utilities, recurring costs

### Savings & Investments

- **[Savings](Savings.md)** - Bank accounts, deposits, emergency funds, compound interest
- **[Shares](Shares.md)** - Stock/ETF holdings with unit-based valuation (enforced)

### Debt Types

- **[Loan](Loan.md)** - Mortgages, auto loans (auto-disables when paid off)
- **[Liability](Liability.md)** - General debts without auto-disable logic
- **[CreditCard](creditCard.md)** - Credit card balances (always-enabled, strict enforcement)

### Assets

- **[Asset](Asset.md)** - Property, vehicles, collectibles (most flexible type)

---

## ItemType Characteristics

| ItemType | Typical CashIn | Typical CashOut | Typical Interest | Special Behavior |
|----------|----------------|-----------------|------------------|------------------|
| Income | Yes (salary) | Optional (deductions) | Rarely used | None |
| Expense | Rarely used | Yes (bills) | Rarely used | None |
| Savings | Optional | Optional | Yes (compound interest) | Main savings account enforcement if IsMainSavingsAccount=true |
| Asset | Optional (rental income) | Optional (maintenance) | Yes (appreciation/depreciation) | Supports liquidation |
| Liability | Rarely used | Yes (payments) | Yes (interest) | No auto-disable |
| Loan | Rarely used | Yes (payments) | Yes (interest) | Auto-disables when Value = 0 |
| Shares | Yes (dividends) | Disabled (enforced) | Yes (growth) | Unit-based valuation, strict sanitization |
| CreditCard | Optional (cashback) | Yes (payments) | Always enabled (enforced) | Always-enabled, extensive sanitization |

---

## Enforcement Summary

### No Enforcement (Full Flexibility)

- Income
- Expense
- Savings (except main savings account)
- Asset
- Liability
- Loan (except auto-disable logic)

### ItemType-Specific Enforcement

**Shares:**
- ShareDetails required
- CashOut.Enabled = false
- CashIn must be percentage if enabled
- Value synced from UnitCount * UnitPrice

**CreditCard:**
- Always enabled (cannot disable)
- Interest always enabled with strict configuration
- Value cannot be negative
- Not a liquid asset
- No self-triggers allowed

**Main Savings Account (Savings type):**
- Type must be Savings
- EvalOrder must be 0
- StartEnabled must be true
- DisabledByUser must be false
- IsLiquidAsset must be true
- EndDate must be >= 95 years

---

## When to Use Each Type

### Income
- Salary or wages
- Freelance/contract work
- Any recurring income source

### Expense
- Rent or mortgage payments (alternative to Loan.CashOut)
- Subscriptions and recurring bills
- One-time purchases

### Savings
- Bank accounts with interest
- Emergency funds
- Main savings account (central cash flow hub)
- Term deposits

### Asset
- Real estate (rental properties, primary residence)
- Vehicles (cars, boats)
- Equipment or collectibles
- Any asset with potential income, expenses, and value changes

### Liability
- General debts without specific payoff behavior
- Perpetual obligations
- Business liabilities

### Loan
- Mortgages
- Auto loans
- Student loans
- Any debt that should auto-disable when paid off

### Shares
- Stock portfolios
- ETF holdings
- Mutual funds
- Any unit-based investment

### CreditCard
- Credit card balances
- Revolving debt with continuous interest accrual
- Scenarios requiring always-enabled debt tracking

---

## Related Documentation

**Models:**
- [financialItem](../models/FinancialItem.md) - Core FinancialItem properties
- [amountFreq](../models/amountFreq.md) - CashIn/CashOut/Interest configuration
- [eventItem](../models/EventItem.md) - Inter-item transfers
- [triggerConditions](../models/TriggerConditions.md) - Conditional activation

**Cash Flow Semantics:**
- [Flows](../Logic/Flows.md) - High-level overview of CashIn/CashOut/Interest patterns

---
