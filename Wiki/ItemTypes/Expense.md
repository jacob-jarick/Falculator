# Expense ItemType

Represents recurring costs like rent, subscriptions, utilities, and maintenance fees.

No example config available (use synthetic examples below)

---

## Overview

ItemType.Expense represents money flowing FROM the user (CashOut). Typical examples:
- **Rent**: Monthly housing payment
- **Utilities**: Electric, water, gas bills
- **Subscriptions**: Streaming services, gym memberships, software
- **Recurring fees**: HOA dues, insurance premiums

**Key characteristics:**
- Primarily uses CashOut (money from user)
- Typically has no Value (pure cash flow)
- CashIn and Interest optional (full flexibility)
- No ItemType-specific sanitization enforcement

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Expense`
3. Set `StartEnabled` to `true` (or configure SelfTrigger for conditional activation)
4. Configure `CashOut` for outgoing payments

---

## Typical Configuration

### CashOut (Primary)

**Purpose:** Money flowing from user to expense

**Typical patterns:**
- Fixed monthly bills
- Variable utility costs
- Annual insurance premiums
- One-time purchases

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 2000,
  "IsPercentage": false,
  "Schedule": { "Frequency": "Monthly", "DayOfMonth": 1 }
}
```

### CashIn (Rarely Used)

**Purpose:** Refunds or rebates

**Typical patterns:**
- Utility rebates
- Insurance refunds
- Cashback rewards

### Interest (Rarely Used)

**Purpose:** Growth or decay applied to Value

**Typical patterns:**
- Rarely used for Expense type (Expense is typically pure cash flow with no balance)

---

## Special Behavior

**None.** Expense type has no ItemType-specific enforcement. Full flexibility with all cash flow properties.

---

## Sanitization Rules

**No ItemType-specific rules.** Standard FinancialItem sanitization applies:
- CashIn, CashOut, Interest initialized if null
- PercentageBasis enforced to Self for all AmountFreqs
- Interest.IsPercentage enforced to true
- Name auto-generated if empty

---

## EventItem Support

**Supported.** Expense items can use EventItems for:
- **Conditional payments**: "Pay gym membership only when income active"
- **State changes**: "Disable subscription when savings low"
- **Variable amounts**: "Pay percentage of income for variable expense"

**Example:**
```json
"Events": [
  {
    "Name": "Conditional Gym Payment",
    "TargetName": "Gym Membership",
    "SetStateOnTrigger": true,
    "TargetStateOnTrigger": "Disable",
    "Triggers": {
      "MainSavingsBalance": {
        "Enabled": true,
        "Operator": "LessThan",
        "ComparisonValue": 5000
      }
    }
  }
]
```

Disables gym membership when savings drop below $5,000.

---

## Example Configuration

### Monthly Rent

```json
{
  "Name": "Rent",
  "Type": "Expense",
  "StartEnabled": true,
  "CashOut": {
    "Enabled": true,
    "Amount": 2000,
    "Schedule": {
      "Frequency": "Monthly",
      "DayOfMonth": 1
    }
  }
}
```

### Annual Insurance Premium

```json
{
  "Name": "Car Insurance",
  "Type": "Expense",
  "StartEnabled": true,
  "CashOut": {
    "Enabled": true,
    "Amount": 1200,
    "Schedule": {
      "Frequency": "Annual",
      "DayOfMonth": 15,
      "MonthOfYear": "January"
    }
  }
}
```

### Temporary Subscription

```json
{
  "Name": "Streaming Service",
  "Type": "Expense",
  "CashOut": {
    "Enabled": true,
    "Amount": 15,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 10 }
  },
  "SelfTrigger": {
    "StartDate": "2026-01-01",
    "EndDate": "2026-12-31"
  }
}
```

Active only during 2026.

---

## Common Patterns

### Pattern 1: Fixed Monthly Bill

```json
{
  "Type": "Expense",
  "CashOut": { "Amount": 150, "Schedule": { "Frequency": "Monthly" } }
}
```

### Pattern 2: Weekly Groceries

```json
{
  "Type": "Expense",
  "CashOut": { "Amount": 200, "Schedule": { "Frequency": "Weekly", "DayOfWeek": "Saturday" } }
}
```

### Pattern 3: One-Time Purchase

```json
{
  "Type": "Expense",
  "CashOut": {
    "Amount": 1500,
    "Schedule": {
      "Frequency": "Monthly",
      "DayOfMonth": 15,
      "TriggerLimit": 1
    }
  },
  "SelfTrigger": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 25,
      "TriggerLimit": 1
    }
  }
}
```

Laptop purchase at age 25 (one-time).

### Pattern 4: Percentage-Based Expense

```json
{
  "Type": "Expense",
  "Value": 5000,
  "CashOut": {
    "Amount": 5,
    "IsPercentage": true,
    "PercentageBasis": "Self",
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

Spends 5% of its own balance monthly (e.g., lifestyle inflation tracking).

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties and lifecycle
- amountFreq.md - CashOut payment configuration
- TriggerConditions.md - SelfTrigger conditional activation
- EventItem.md - Inter-item transfers

**ItemTypes:**
- Income.md - Incoming payments (opposite of Expense)
- Asset.md - Can also have CashOut (maintenance costs)
- Loan.md - Debt with CashOut (loan repayments)

---

## Implementation Details

**No ItemType-specific code.** Expense uses standard FinancialItem behavior:

**Source file:** `Models/FinancialItem.cs`

**Sanitization:** Lines 446-731 (standard FinancialItem.Sanitize(), no Expense-specific rules)

**Evaluation:** Standard trigger evaluation (EvaluateSelfTrigger)

**Processing:** Standard SimFrame.CalculateItem() - CashOut deducted from main savings account

---
