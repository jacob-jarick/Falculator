# Liability ItemType

Represents debts and obligations without special auto-disable logic.

No example config available (use synthetic examples below or use Loan type for auto-disable)

---

## Overview

ItemType.Liability represents general debt obligations without automatic payoff behavior. Typical examples:
- **Unsecured debts**: Personal debts, IOUs
- **Business liabilities**: Accounts payable, accrued expenses
- **Other obligations**: Legal settlements, guarantees
- **Perpetual debts**: Debts that don't automatically disable when paid off

**Key characteristics:**
- Negative Value (debt amount, convention)
- Interest represents interest accrued on debt
- CashOut represents debt payments
- **No auto-disable when Value = 0** (unlike Loan type)
- Otherwise identical to Loan type in functionality
- No ItemType-specific sanitization enforcement

**When to use Liability vs Loan:**
- Use **Loan** if you want automatic deactivation when paid off
- Use **Liability** for debts that stay active regardless of balance

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Liability`
3. Set `Value` to **negative** debt amount (e.g., -10000)
4. Set `StartEnabled` to `true` (or configure SelfTrigger)
5. Configure `Interest` for accrued interest (optional)
6. Configure `CashOut` for debt payments (optional)

---

## Typical Configuration

### Value (Debt Amount)

**Purpose:** Outstanding liability balance

**Typical patterns:**
- Negative for debts (convention, e.g., -10000)
- Can be positive for prepaid liabilities or credit balances
- Increases (more negative) from Interest
- Decreases (less negative) from CashOut payments

### Interest (Accrued Interest)

**Purpose:** Interest on liability

**Typical patterns:**
- Annual rate with monthly compounding
- Fixed or variable interest rates
- Applied to Value (increases debt)

**Configuration:**
```json
"Interest": {
  "Enabled": true,
  "Amount": 8.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

### CashOut (Debt Payments)

**Purpose:** Payments toward liability

**Typical patterns:**
- Fixed monthly payments
- Variable payments based on conditions
- Minimum payments

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 200,
  "Schedule": { "Frequency": "Monthly" }
}
```

### CashIn (Rarely Used)

**Purpose:** Borrowing or disbursements

**Typical patterns:**
- Initial debt creation
- Additional borrowing
- Refinancing proceeds

---

## Special Behavior

**None.** Liability type has no ItemType-specific behavior:
- Does NOT auto-disable when Value = 0 (unlike Loan)
- No ItemType-specific sanitization enforcement
- Full flexibility with all cash flow properties

---

## Sanitization Rules

**No ItemType-specific sanitization rules.** Standard FinancialItem sanitization applies:
- CashIn, CashOut, Interest initialized if null
- PercentageBasis enforced to Self for all AmountFreqs
- Interest.IsPercentage enforced to true
- Name auto-generated if empty

---

## EventItem Support

**Supported.** Liability items can use EventItems for:
- **Variable payments**: "Pay 5% of income toward debt"
- **Conditional payments**: "Pay extra when savings exceed threshold"
- **State changes**: "Disable liability when balance reaches zero (manual payoff)"

**Example:**
```json
"Events": [
  {
    "Name": "Extra Debt Payment",
    "TargetName": "Personal Debt",
    "CashOut": {
      "Enabled": true,
      "Amount": 500,
      "Schedule": { "Frequency": "Monthly" }
    },
    "Triggers": {
      "MainSavingsBalance": {
        "Enabled": true,
        "Operator": "GreaterThanOrEqual",
        "ComparisonValue": 15000
      }
    }
  }
]
```

Makes extra $500 monthly payment when savings >= $15k.

---

## Example Configuration

### Basic Personal Debt

```json
{
  "Name": "Personal Loan",
  "Type": "Liability",
  "Value": -10000,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 8.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 300,
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

$10k debt at 8.5% APR, $300 monthly payment. Stays active even when paid off.

### Interest-Only Liability

```json
{
  "Name": "Business Liability",
  "Type": "Liability",
  "Value": -50000,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 5.0,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 208.33,
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

Interest-only payments: $50,000 * 5% / 12 = $208.33/month. Principal never decreases.

### Manual Payoff Trigger

```json
{
  "Name": "Temporary Debt",
  "Type": "Liability",
  "Value": -5000,
  "Interest": {
    "Enabled": true,
    "Amount": 6.0,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 200,
    "Schedule": { "Frequency": "Monthly" }
  },
  "SelfTrigger": {
    "BalanceTrigger": {
      "Enabled": true,
      "Operator": "LessThan",
      "ComparisonValue": 0
    }
  }
}
```

Disables when Value >= 0 (paid off). Effectively mimics Loan auto-disable but uses explicit trigger.

---

## Common Patterns

### Pattern 1: Simple Debt

```json
{
  "Type": "Liability",
  "Value": -8000,
  "CashOut": { "Amount": 250, "Schedule": { "Frequency": "Monthly" } }
}
```

No interest, fixed monthly payment.

### Pattern 2: High-Interest Debt

```json
{
  "Type": "Liability",
  "Value": -15000,
  "Interest": { "Amount": 12.0, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 400, "Schedule": { "Frequency": "Monthly" } }
}
```

12% APR, $400 monthly payment.

### Pattern 3: Conditional Payoff

```json
{
  "Type": "Liability",
  "Value": -20000,
  "Interest": { "Amount": 7.0, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 500, "Schedule": { "Frequency": "Monthly" } },
  "Events": [
    {
      "TargetName": "Personal Debt",
      "CashOut": { "Amount": 10000, "Schedule": { "TriggerLimit": 1 } },
      "Triggers": {
        "Age": { "Operator": ">=", "ComparisonValue": 30 },
        "MainSavingsBalance": { "Operator": ">=", "ComparisonValue": 30000 }
      }
    }
  ]
}
```

$500 monthly + $10k lump sum at age 30 when savings >= $30k.

### Pattern 4: Perpetual Obligation

```json
{
  "Type": "Liability",
  "Value": -100000,
  "StartEnabled": true,
  "Interest": { "Amount": 4.0, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 333.33, "Schedule": { "Frequency": "Monthly" } }
}
```

Interest-only perpetual debt. Balance never decreases (interest = payment).

---

## Liability vs Loan Comparison

| Feature | Liability | Loan |
|---------|-----------|------|
| Auto-disable when Value = 0 | No | Yes |
| ItemType-specific sanitization | No | No |
| Cash flow flexibility | Full | Full |
| Typical use case | General debts, perpetual obligations | Mortgages, auto loans, student loans |
| Stays active after payoff | Yes | No |

**Recommendation:**
- Use **Loan** for traditional loans (mortgages, auto loans) that should auto-disable
- Use **Liability** for debts that need manual control or perpetual tracking

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties
- amountFreq.md - Interest and CashOut calculation
- TriggerConditions.md - Manual payoff triggers
- EventItem.md - Variable payments

**ItemTypes:**
- Loan.md - Similar but with auto-disable when paid off
- creditCard.md - Always-enabled debt with strict enforcement
- Expense.md - Can represent debt payments (alternative)

---

## Implementation Details

**No ItemType-specific code.** Liability uses standard FinancialItem behavior:

**Source file:** `Models/FinancialItem.cs`

**Sanitization:** Lines 446-731 (standard FinancialItem.Sanitize(), no Liability-specific rules)

**Evaluation:** Standard trigger evaluation (EvaluateSelfTrigger) - NO auto-disable check

**Processing:**
- SimFrame.CalculateItem() - Interest applied to Value, CashOut processed
- Cash flow sweep - CashOut deducted from main savings account

**Comparison with Loan:**
- Loan has auto-disable check at lines 405-419 (EvaluateSelfTrigger)
- Liability does NOT have this check

---
