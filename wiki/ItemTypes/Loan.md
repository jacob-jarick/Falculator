# Loan ItemType

Represents loans and mortgages that auto-disable when paid off.

Example configs: "Data/examples/HomeLoan.json", "Data/examples/005.Test.json"

---

## Overview

ItemType.Loan represents debt obligations that automatically deactivate when fully repaid. Typical examples:
- **Mortgages**: Home loans
- **Auto loans**: Car financing
- **Student loans**: Education debt
- **Personal loans**: Unsecured debt

**Key characteristics:**
- Negative Value (debt amount, e.g., -400000 for $400k mortgage)
- Interest represents loan interest (accrues against Value)
- CashOut represents loan repayments (reduces debt)
- **Auto-disables when Value = 0** (paid off)
- Otherwise behaves like standard Liability

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `Loan`
3. Set `Value` to **negative** debt amount (e.g., -400000)
4. Set `StartEnabled` to `true`
5. Configure `Interest` for loan interest rate
6. Configure `CashOut` for loan repayments

---

## Typical Configuration

### Value (Debt Amount)

**Purpose:** Outstanding loan balance

**Typical patterns:**
- Always negative (debt convention)
- Example: -400000 for $400k mortgage
- Increases (becomes more negative) from Interest
- Decreases (becomes less negative) from CashOut repayments

### Interest (Loan Interest)

**Purpose:** Interest accrued against debt

**Typical patterns:**
- Annual rate with monthly compounding
- Fixed interest rate (e.g., 6.5% APR)
- Applied to Value (makes debt larger)

**Configuration:**
```json
"Interest": {
  "Enabled": true,
  "Amount": 6.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

### CashOut (Loan Repayments)

**Purpose:** Monthly loan payments

**Typical patterns:**
- Fixed monthly payment (principal + interest)
- Principal-only extra payments
- Variable payments based on income

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 2500,
  "Schedule": { "Frequency": "Monthly", "DayOfMonth": 1 }
}
```

### CashIn (Rarely Used)

**Purpose:** Loan disbursements or refinancing

**Typical patterns:**
- Initial loan disbursement (one-time)
- Refinancing proceeds
- Rarely used (loans typically start with negative Value)

---

## Special Behavior

### Auto-Disable When Paid Off

**Trigger logic in EvaluateSelfTrigger():**

```csharp
if (Type == ItemType.Loan && Value == 0)
{
    EnabledBySim = false;
    return false;
}
```

**When Value reaches exactly 0:**
- Item is automatically disabled
- No further Interest accrual
- No further CashOut payments
- Logged at INFO level: "Loan paid off (Value = 0.00), disabling item"

**Implicit trigger:**
- Loans always have an implicit trigger condition (Value == 0 check)
- If no explicit SelfTrigger configured: stays active until paid off
- If explicit SelfTrigger configured: both conditions apply (AND logic)

### No Other Enforcement

No other ItemType-specific sanitization. Full flexibility with cash flows.

---

## Sanitization Rules

**No ItemType-specific sanitization rules.** Standard FinancialItem sanitization applies:
- CashIn, CashOut, Interest initialized if null
- PercentageBasis enforced to Self for all AmountFreqs
- Interest.IsPercentage enforced to true
- Name auto-generated if empty

**Note:** Auto-disable logic is in EvaluateSelfTrigger(), not Sanitize().

---

## EventItem Support

**Supported.** Loan items can use EventItems for:
- **Extra payments**: "Pay extra $1000 when savings > $20k"
- **Variable payments**: "Pay 10% of income toward loan"
- **Conditional payments**: "Pause payments when income inactive"

**Example:**
```json
"Events": [
  {
    "Name": "Extra Loan Payment",
    "TargetName": "Home Loan",
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

Makes extra $1,000 monthly payment when savings >= $20k.

---

## Example Configuration

### Basic Mortgage

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
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 1 }
  }
}
```

$400k mortgage at 6.5% APR, $2,500 monthly payment.

### Auto Loan with Extra Payments

```json
{
  "Name": "Car Loan",
  "Type": "Loan",
  "Value": -30000,
  "StartEnabled": true,
  "Interest": {
    "Enabled": true,
    "Amount": 5.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 600,
    "Schedule": { "Frequency": "Monthly" }
  },
  "Events": [
    {
      "Name": "Extra Payment",
      "TargetName": "Car Loan",
      "CashOut": { "Amount": 200, "Schedule": { "Frequency": "Monthly" } },
      "Triggers": {
        "MainSavingsBalance": { "Operator": ">=", "ComparisonValue": 15000 }
      }
    }
  ]
}
```

$600 base payment + $200 extra when savings >= $15k. Auto-disables when paid off.

### Student Loan with Age-Based Activation

```json
{
  "Name": "Student Loan",
  "Type": "Loan",
  "Value": -50000,
  "Interest": {
    "Enabled": true,
    "Amount": 4.5,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 500,
    "Schedule": { "Frequency": "Monthly" }
  },
  "SelfTrigger": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 25
    }
  }
}
```

Repayments start at age 25. Auto-disables when paid off OR if explicit trigger fails (both must pass).

---

## Common Patterns

### Pattern 1: Standard Mortgage

**See:** Data/examples/HomeLoan.json

```json
{
  "Type": "Loan",
  "Value": -400000,
  "Interest": { "Amount": 6.5, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 2500, "Schedule": { "Frequency": "Monthly" } }
}
```

### Pattern 2: Accelerated Payoff

```json
{
  "Type": "Loan",
  "Value": -200000,
  "Interest": { "Amount": 5.0, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 2000, "Schedule": { "Frequency": "Monthly" } },
  "Events": [
    {
      "TargetName": "Home Loan",
      "CashOut": { "Amount": 10, "IsPercentage": true, "PercentageBasis": "Source" },
      "Triggers": { "MainSavingsBalance": { "Operator": ">=", "ComparisonValue": 50000 } }
    }
  ]
}
```

Base payment + 10% of source balance when savings >= $50k.

### Pattern 3: Interest-Only Period

```json
{
  "Type": "Loan",
  "Value": -300000,
  "Interest": { "Amount": 5.5, "AnnualRateMonthlyCompounding": true },
  "CashOut": { "Amount": 1375, "Schedule": { "Frequency": "Monthly" } },
  "SelfTrigger": {
    "StartDate": "2026-01-01",
    "EndDate": "2028-12-31"
  }
}
```

Interest-only payments 2026-2028 (amount calculated as interest only), then switches to principal+interest.

### Pattern 4: Conditional Loan Payoff

**See:** Data/examples/005.Test.json

Tests auto-disable behavior when loan reaches $0.

---

## Debugging

### Auto-Disable Logging

When loan is paid off:

```
[FinancialItem-fi123456] Loan paid off (Value = 0.00), disabling item
```

**Enable debug logs:**
```bash
Falculator.exe --config "config.json" --loglevel DEBUG
```

### Common Issues

**Issue:** Loan doesn't disable after paying off.

**Solution:** Check that Value reaches exactly 0. Floating-point precision may cause Value to be very small but not exactly 0. Use CashOut amounts that divide evenly into the loan balance.

**Issue:** Loan disables too early.

**Solution:** Check for explicit SelfTrigger conditions that might fail before loan is paid off. Both Value==0 AND SelfTrigger must pass.

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties, EvaluateSelfTrigger logic
- amountFreq.md - Interest and CashOut calculation
- TriggerConditions.md - SelfTrigger conditional activation
- EventItem.md - Extra payments via EventItems

**ItemTypes:**
- Liability.md - Similar but no auto-disable logic
- creditCard.md - Always-enabled debt (opposite of Loan)
- Asset.md - Can have negative Value for leveraged assets

**Examples:**
- Data/examples/HomeLoan.json - Mortgage with repayments
- Data/examples/005.Test.json - Loan payoff testing

---

## Implementation Details

**Source file:** `Models/FinancialItem.cs`

**Auto-disable logic:** Lines 405-419 (EvaluateSelfTrigger)

```csharp
// Loan payoff check: Only disable if value has reached zero (paid off)
if (Type == ItemType.Loan && Value == 0)
{
    DebugLogger.Log(Id, Name, $"Loan paid off (Value = {Value:F2}), disabling item", DebugLogger.LogLevel.Info);
    EnabledBySim = false;
    return false;
}

// For loans without explicit trigger conditions, default to enabled (stay active until paid off)
if (Type == ItemType.Loan && !SelfTrigger.HasAnyConditions())
{
    EnabledBySim = true;
    return true;
}
```

**Standard sanitization:** Lines 446-731 (standard FinancialItem.Sanitize(), no Loan-specific rules)

**Processing:**
- SimFrame.CalculateItem() - Interest applied to Value (makes debt larger)
- Cash flow sweep - CashOut deducted from main savings account (reduces debt)
- Each step: Value checked for == 0 condition

---
