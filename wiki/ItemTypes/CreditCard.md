# CreditCard ItemType

Represents credit card balances with always-enabled behavior and extensive sanitization.

Example config: "Data/examples/InteractiveMode.Test.json"

---

## Overview

ItemType.CreditCard represents credit card debt with strict enforcement and always-enabled behavior. Key differences from other debt types:
- **Always enabled** (cannot be disabled)
- **Interest always enabled** with strict configuration
- **Value cannot be negative** (no overpayment allowed)
- **Not a liquid asset** (debt cannot be easily transferred)
- **No self-triggers allowed** (always active)
- **Extensive sanitization** (most restrictive ItemType)

**Typical use case:**
- Credit card balances with APR interest
- Revolving debt tracking
- Scenarios requiring continuous interest accrual

---

## Quick Start

1. Create a new FinancialItem
2. Set `Type` to `CreditCard`
3. Set `Value` to starting balance (e.g., 5000 for $5k debt)
4. Interest is automatically configured (APR with monthly compounding)
5. Configure CashOut for monthly payments (optional)

**Note:** Most properties are auto-configured by sanitization. Minimal manual setup required.

---

## Typical Configuration

### Value (Credit Card Balance)

**Purpose:** Current card balance

**Restrictions:**
- Must be >= 0 (no negative balances)
- If Value < 0: sanitization sets to 0

**Typical patterns:**
- Positive values represent debt owed
- Zero means no balance
- Updated by Interest (increases balance)
- Updated by CashOut payments (decreases balance)

### Interest (Always Enabled)

**Purpose:** APR interest on balance

**Strict enforcement:**
- Must be enabled
- Must be percentage-based
- Must use annual rate with monthly compounding
- Must accrue on last day of month (DayOfMonth = 31)
- Must trigger monthly
- Cannot be negative
- Cannot have trigger limits

**Auto-configured:**
```json
"Interest": {
  "Enabled": true,
  "Amount": 18.99,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true,
  "PercentageBasis": "Self",
  "Schedule": {
    "Frequency": "Monthly",
    "DayOfMonth": 31,
    "DayOfWeek": null,
    "MonthOfYear": null,
    "TriggerLimit": 0
  }
}
```

### CashOut (Optional Payments)

**Purpose:** Monthly credit card payments

**Typical patterns:**
- Minimum payment (e.g., 2% of balance or $25)
- Fixed monthly payment
- Variable payments based on income
- Full balance payoff

**Configuration:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 200,
  "Schedule": { "Frequency": "Monthly" }
}
```

### CashIn (Rarely Used)

**Purpose:** Cashback rewards or refunds

**Typical patterns:**
- Cashback percentage
- Merchant refunds
- Points redemption

---

## Special Behavior

### Always-Enabled Enforcement

**In EvaluateSelfTrigger():**
```csharp
if (Type == ItemType.CreditCard)
{
    EnabledBySim = true;
    return true;
}
```

CreditCard items are always active regardless of:
- StartEnabled value
- DisabledByUser value
- SelfTrigger conditions
- Date ranges

**Rationale:** Interest must accrue continuously to accurately model credit card debt.

### No Overpayment

**Value cannot be negative:**
- If Value < 0 after sanitization: set to 0
- If payment exceeds balance during simulation: capped at balance

**Rationale:** Credit cards don't allow positive balances (unlike bank accounts).

---

## Sanitization Rules

**Extensive ItemType-specific enforcement** (lines 547-654 in FinancialItem.cs):

### Value Sanitization

- If Value < 0: set to 0, log WARN

### Interest Enforcement

| Property | Enforced Value | Reason |
|----------|----------------|--------|
| Enabled | true | Credit cards always accrue interest |
| Amount | >= 0 | Interest rate cannot be negative |
| IsPercentage | true | APR is percentage-based |
| AnnualRateMonthlyCompounding | true | Standard credit card interest calculation |
| PercentageBasis | Self | Interest based on card balance |
| Schedule.Frequency | Monthly | Interest accrues monthly |
| Schedule.DayOfMonth | 31 | Last day of month (statement date) |
| Schedule.DayOfWeek | null | Not applicable for monthly |
| Schedule.MonthOfYear | null | Accrues every month |
| Schedule.TriggerLimit | 0 | Unlimited (accrues indefinitely) |

### Lifecycle Enforcement

| Property | Enforced Value | Reason |
|----------|----------------|--------|
| DisabledByUser | false | Cannot be disabled by user |
| StartEnabled | true | Must start active |
| SelfTrigger | cleared | No triggers allowed |

### Configuration Enforcement

| Property | Enforced Value | Reason |
|----------|----------------|--------|
| IsLiquidAsset | false | Debt is not a liquid asset |

**All corrections are logged at WARN level.**

---

## EventItem Support

**Supported but limited.** CreditCard items can use EventItems for:
- **Payments**: Transfers from income/savings to pay down balance
- **Conditional payments**: Pay extra when savings exceed threshold
- **State changes**: NOT supported (always enabled)

**Cannot do:**
- Enable/disable credit card via EventItem.SetStateOnTrigger (always enabled)
- Self-reference in EventItem.SelfTrigger (cleared by sanitization)

**Example:**
```json
{
  "Name": "Salary",
  "Type": "Income",
  "Events": [
    {
      "Name": "Credit Card Payment",
      "TargetName": "Credit Card",
      "CashOut": {
        "Enabled": true,
        "Amount": 500,
        "Schedule": { "Frequency": "Monthly", "DayOfMonth": 15 }
      }
    }
  ]
}
```

Salary pays $500 toward credit card monthly.

---

## Example Configuration

### Basic Credit Card

```json
{
  "Name": "Credit Card",
  "Type": "CreditCard",
  "Value": 5000,
  "Interest": {
    "Enabled": true,
    "Amount": 18.99,
    "IsPercentage": true,
    "AnnualRateMonthlyCompounding": true
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 200,
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

$5k balance at 18.99% APR, $200 monthly payment. Sanitization auto-corrects all Interest properties.

### Minimum Payment (Percentage)

```json
{
  "Name": "Credit Card",
  "Type": "CreditCard",
  "Value": 3000,
  "Interest": {
    "Amount": 21.99
  },
  "CashOut": {
    "Enabled": true,
    "Amount": 2,
    "IsPercentage": true,
    "PercentageBasis": "Self",
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

Pays 2% of balance monthly (minimum payment). Sanitization auto-configures Interest.

### Revolving Balance with Variable Payments

Configured on Income item:

```json
{
  "Name": "Salary",
  "Type": "Income",
  "Events": [
    {
      "Name": "Credit Card Payment",
      "TargetName": "Credit Card",
      "CashOut": {
        "Amount": 10,
        "IsPercentage": true,
        "PercentageBasis": "Source",
        "Schedule": { "Frequency": "Monthly" }
      }
    }
  ]
}
```

Pays 10% of salary toward credit card each month.

---

## Common Patterns

### Pattern 1: Simple Credit Card Tracking

```json
{
  "Type": "CreditCard",
  "Value": 8000,
  "Interest": { "Amount": 19.99 },
  "CashOut": { "Amount": 300, "Schedule": { "Frequency": "Monthly" } }
}
```

### Pattern 2: Minimum Payment Strategy

```json
{
  "Type": "CreditCard",
  "Value": 10000,
  "Interest": { "Amount": 22.99 },
  "CashOut": {
    "Amount": 2.5,
    "IsPercentage": true,
    "Schedule": { "Frequency": "Monthly" }
  }
}
```

Pays 2.5% of balance (minimum payment). Balance grows due to high interest vs low payment.

### Pattern 3: Aggressive Payoff

```json
{
  "Type": "CreditCard",
  "Value": 5000,
  "Interest": { "Amount": 18.99 },
  "CashOut": { "Amount": 1000, "Schedule": { "Frequency": "Monthly" } }
}
```

$1,000 monthly payment aggressively pays down balance.

### Pattern 4: Conditional Extra Payment

Configured on Income item:

```json
{
  "Events": [
    {
      "TargetName": "Credit Card",
      "CashOut": { "Amount": 500, "Schedule": { "Frequency": "Monthly" } },
      "Triggers": {
        "MainSavingsBalance": { "Operator": ">=", "ComparisonValue": 20000 }
      }
    }
  ]
}
```

Extra $500 payment when savings >= $20k.

---

## Debugging

### Sanitization Warnings

Credit card sanitization is very verbose. Expect multiple WARN messages:

```
[FinancialItem-fi123456] Sanitize: CreditCard Value < 0, setting to 0
[FinancialItem-fi123456] Sanitize: CreditCard Interest was disabled, enabling
[FinancialItem-fi123456] Sanitize: CreditCard Interest.IsPercentage must be true, setting to true
[FinancialItem-fi123456] Sanitize: CreditCard Interest.AnnualRateMonthlyCompounding must be true, setting to true
[FinancialItem-fi123456] Sanitize: CreditCard Interest.Schedule.Frequency must be Monthly, setting to Monthly
[FinancialItem-fi123456] Sanitize: CreditCard Interest.Schedule.DayOfMonth should be 31, setting to 31
[FinancialItem-fi123456] Sanitize: CreditCard cannot be disabled by user, setting DisabledByUser to false
[FinancialItem-fi123456] Sanitize: CreditCard must start enabled, setting StartEnabled to true
[FinancialItem-fi123456] Sanitize: CreditCard cannot be a liquid asset, setting IsLiquidAsset to false
[FinancialItem-fi123456] Sanitize: CreditCard should not have self-trigger conditions, clearing trigger
```

**This is normal.** Sanitization auto-corrects all properties.

### Common Issues

**Issue:** Credit card disables during simulation.

**Solution:** Impossible. CreditCard type is always enabled. Check ItemType is actually CreditCard, not Liability/Loan.

**Issue:** Interest not accruing.

**Solution:** Impossible if Type=CreditCard. Sanitization forces Interest.Enabled=true. Check logs for sanitization warnings.

**Issue:** Negative balance allowed.

**Solution:** Check for manual Value modifications after sanitization. Sanitization sets Value=0 if negative.

---

## Related Documentation

**Models:**
- FinancialItem.md - Core FinancialItem properties, EvaluateSelfTrigger always-enabled logic
- amountFreq.md - Interest calculation with monthly compounding
- EventItem.md - Payment transfers

**ItemTypes:**
- Loan.md - Debt with auto-disable (opposite of always-enabled)
- Liability.md - General debt without enforcement
- Expense.md - Alternative for modeling recurring payments

**Examples:**
- Data/examples/InteractiveMode.Test.json - Credit card configuration

---

## Implementation Details

**Source file:** `Models/FinancialItem.cs`

**Always-enabled logic:** Lines 398-403 (EvaluateSelfTrigger)

```csharp
if (Type == ItemType.CreditCard)
{
    EnabledBySim = true;
    return true;
}
```

**Sanitization:** Lines 547-654 (extensive enforcement)

**Key enforcement blocks:**
- Value >= 0: Line 551-555
- Interest.Enabled: Line 558-562
- Interest.Amount >= 0: Line 565-569
- Interest.IsPercentage: Line 600-604
- Interest.AnnualRateMonthlyCompounding: Line 607-611
- Interest.PercentageBasis: Line 614-618
- Interest.Schedule.Frequency: Line 621-625
- Interest.Schedule.DayOfMonth: Line 635-639
- DisabledByUser: Line 572-576
- StartEnabled: Line 579-583
- IsLiquidAsset: Line 586-590
- SelfTrigger: Line 593-597

**Processing:**
- SimFrame.CalculateItem() - Interest applied monthly to Value
- Cash flow sweep - CashOut deducted from main savings, reduces card balance

---
