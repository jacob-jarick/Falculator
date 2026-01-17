# Income ItemType

Represents income sources like salary, freelance work, rental income, and dividends.

Example config: "Data/examples/HomeLoan.json" (Salary)

---

## Overview

ItemType.Income represents money flowing TO the user (CashIn). Typical examples:
- **Salary**: Regular employment income
- **Freelance**: Contract or consulting income
- **Rental Income**: Property rental payments (alternatively use Asset type)
- **Dividends**: Investment returns (alternatively use Shares type with CashIn)

**Key characteristics:**
- Primarily uses CashIn (money to user)
- Typically has no Value (pure cash flow)
- CashOut and Interest optional (full flexibility)
- No ItemType-specific sanitization enforcement

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Income`
3. Set `StartEnabled` to `true` (or configure SelfTrigger for conditional activation)
4. Configure `CashIn` for incoming payments

---

## Typical Configuration

### CashIn (Primary)

**Purpose:** Money flowing to user

**Typical patterns:**
- Fixed monthly salary
- Hourly/weekly wages
- Annual bonuses
- Irregular freelance payments

**Configuration:**
```json
"CashIn": {
  "Enabled": true,
  "Amount": 5000,
  "IsPercentage": false,
  "Schedule": { "Frequency": "Monthly", "DayOfMonth": 15 }
}
```

### CashOut (Optional)

**Purpose:** Deductions or expenses associated with income

**Typical patterns:**
- Tax withholding (alternative to Config.TaxMode)
- Union dues
- Retirement contributions
- Health insurance premiums

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 500,
  "Schedule": { "Frequency": "Monthly" }
}
```

**Note:** Most users model tax via Config.TaxMode instead of CashOut.

### Interest (Rarely Used)

**Purpose:** Growth or decay applied to Value

**Typical patterns:**
- Rarely used for Income type (Income is typically pure cash flow with no balance)

---

## Special Behavior

**None.** Income type has no ItemType-specific enforcement. Full flexibility with all cash flow properties.

---

## Sanitization Rules

**No ItemType-specific rules.** Standard FinancialItem sanitization applies:
- CashIn, CashOut, Interest initialized if null
- PercentageBasis enforced to Self for all AmountFreqs
- Interest.IsPercentage enforced to true
- Name auto-generated if empty

---

## EventItem Support

**Supported.** Income items can use EventItems for:
- **Automatic savings transfers**: "Transfer 20% of salary to savings account"
- **Bill payments**: "Pay $1000 rent when salary received"
- **Conditional activation**: "Enable investment contributions when salary > threshold"

**Example:**
```json
"Events": [
  {
    "Name": "Automatic Savings",
    "TargetName": "Savings Account",
    "CashOut": {
      "Enabled": true,
      "Amount": 20,
      "IsPercentage": true,
      "PercentageBasis": "Source",
      "Schedule": { "Frequency": "Monthly" }
    }
  }
]
```

Transfers 20% of income value to savings each month.

---

## Example Configuration

### Biweekly Salary

```json
{
  "Name": "Salary",
  "Type": "Income",
  "StartEnabled": true,
  "CashIn": {
    "Enabled": true,
    "Amount": 2500,
    "IsPercentage": false,
    "Schedule": {
      "Frequency": "Fortnightly",
      "DayOfWeek": "Friday"
    }
  }
}
```

### Age-Triggered Pension

```json
{
  "Name": "Government Pension",
  "Type": "Income",
  "CashIn": {
    "Enabled": true,
    "Amount": 1500,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 1 }
  },
  "SelfTrigger": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 67,
      "TriggerLimit": 1
    }
  }
}
```

Activates monthly pension payments at age 67.

### Freelance Income with Tax Deduction

```json
{
  "Name": "Freelance Work",
  "Type": "Income",
  "StartEnabled": true,
  "CashIn": {
    "Enabled": true,
    "Amount": 3000,
    "Schedule": { "Frequency": "Monthly" }
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 900,
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

$3,000 gross income, $900 tax withholding, $2,100 net to user.

---

## Common Patterns

### Pattern 1: Simple Monthly Salary

```json
{
  "Type": "Income",
  "CashIn": { "Amount": 5000, "Schedule": { "Frequency": "Monthly" } }
}
```

### Pattern 2: Hourly Wage (Weekly)

```json
{
  "Type": "Income",
  "CashIn": { "Amount": 1200, "Schedule": { "Frequency": "Weekly", "DayOfWeek": "Friday" } }
}
```

### Pattern 3: Annual Bonus

```json
{
  "Type": "Income",
  "CashIn": { "Amount": 10000, "Schedule": { "Frequency": "Annual", "DayOfMonth": 15, "MonthOfYear": "December" } }
}
```

### Pattern 4: Conditional Freelance Income

```json
{
  "Type": "Income",
  "CashIn": { "Amount": 2000, "Schedule": { "Frequency": "Monthly" } },
  "SelfTrigger": {
    "StartDate": "2027-01-01",
    "EndDate": "2028-12-31"
  }
}
```

Active only during 2027-2028.

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties and lifecycle
- amountFreq.md - CashIn payment configuration
- TriggerConditions.md - SelfTrigger conditional activation
- EventItem.md - Inter-item transfers

**ItemTypes:**
- Expense.md - Outgoing payments (opposite of Income)
- Asset.md - Can also generate CashIn (rental income)
- Shares.md - Can generate CashIn (dividends)

**Examples:**
- Data/examples/HomeLoan.json - Salary income example
- Data/examples/002.Test.json - Multiple income sources

---

## Implementation Details

**No ItemType-specific code.** Income uses standard FinancialItem behavior:

**Source file:** `Models/FinancialItem.cs`

**Sanitization:** Lines 446-731 (standard FinancialItem.Sanitize(), no Income-specific rules)

**Evaluation:** Standard trigger evaluation (EvaluateSelfTrigger)

**Processing:** Standard SimFrame.CalculateItem() - CashIn flows to main savings account

---
