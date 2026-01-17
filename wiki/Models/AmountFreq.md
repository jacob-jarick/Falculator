# AmountFreq Model

Payment and frequency system for defining recurring financial transactions.

---

## Overview

AmountFreq (Amount + Frequency) represents a recurring payment or cash flow with a specified schedule. It's the fundamental building block for modeling:
- **CashIn**: Incoming payments (salary, dividends, rental income)
- **CashOut**: Outgoing payments (rent, subscriptions, loan repayments)
- **Interest**: Growth or decay rates (investment returns, loan interest, credit card APR)

**Key Features:**
- Fixed value OR percentage-based amounts
- Flexible scheduling (Daily, Weekly, Fortnightly, Monthly, Annual)
- Annual rate with monthly compounding support
- Configurable percentage basis (source vs destination value)
- Payment count accumulation for irregular step intervals

---

## Quick Start

### Fixed Monthly Payment

```json
{
  "Name": "Monthly Salary",
  "Enabled": true,
  "Amount": 5000,
  "IsPercentage": false,
  "Schedule": {
    "Frequency": "Monthly",
    "DayOfMonth": 15
  }
}
```

This configuration pays $5,000 on the 15th of each month.

### Percentage-Based Interest

```json
{
  "Name": "Savings Interest",
  "Enabled": true,
  "Amount": 4.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

This configuration applies 4.5% annual interest compounded monthly.

---

## Properties

### Identity Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Id | string | auto-generated | 8-character unique identifier (read-only) |
| Name | string | "AmountFreq-{Id}" | Human-readable name for debugging |

### Payment Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Enabled | bool | false | Enable or disable this payment |
| Amount | decimal | 0 | Payment value (fixed amount OR percentage) |
| IsPercentage | bool | false | If true, Amount is treated as a percentage |
| PercentageBasis | BasisType | Source | Which value to use for percentage calculations |
| AnnualRateMonthlyCompounding | bool | false | Treat Amount as annual rate compounded monthly |

### Schedule Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Schedule | TriggerFrequency | Monthly | Payment frequency and timing configuration |

---

## PercentageBasis Enum

Controls which value is used for percentage calculations:

| Value | Numeric | Behavior |
|-------|---------|----------|
| Source | 0 | Calculate percentage of source item value (default) |
| Destination | 1 | Calculate percentage of destination item value (EventItem only) |
| Self | 2 | Alias for Source (same behavior) |

**FinancialItem restriction:** CashIn, CashOut, and Interest always use Source/Self basis (enforced by Sanitize).

**EventItem flexibility:** EventItem transfers can use Destination basis to calculate amounts based on target item value.

**Example use case:**
- **Source basis:** "Transfer 10% of my savings to investments" (10% of savings balance)
- **Destination basis:** "Top up investments to 90% of their target allocation" (10% of investment balance)

---

## Calculation Logic

### ProcessPayment Method

Calculates the delta (change) in value for a given time period.

**Signature:**
```csharp
decimal ProcessPayment(DateOnly frameDate, DateOnly prevFrameDate, decimal value, decimal? destinationValue = null)
```

**Parameters:**
- `frameDate` - Current simulation step date
- `prevFrameDate` - Previous simulation step date
- `value` - Source item value (or current value for self-based calculations)
- `destinationValue` - Optional destination item value (for PercentageBasis.Destination)

**Return value:** Change in value based on payment frequency and amount

### Calculation Steps

1. **Early exit checks:**
   - If `Enabled == false`: return 0
   - If `Amount == 0`: return 0

2. **Payment count calculation:**
   - Uses `Schedule.ProcessTrigger(frameDate, prevFrameDate)` to determine how many payments occur in this period
   - If payment count == 0: return 0

3. **Amount calculation:**
   - **Fixed value:** `result = Amount * paymentCount`
   - **Percentage (simple):** `result = value * ((1 + Amount/100)^paymentCount - 1)`
   - **Percentage (annual compounded monthly):** `result = value * ((1 + Amount/100)^(1/12 * paymentCount) - 1)`

4. **Percentage basis handling:**
   - If `destinationValue` is provided and `PercentageBasis == Destination`: use `destinationValue` instead of `value`
   - Otherwise: use `value` (source/self)

### Fixed Value Example

```
Amount = 100
IsPercentage = false
Frequency = Weekly
PaymentCount = 2 (two weeks passed)

Result = 100 * 2 = $200
```

### Simple Percentage Example

```
Amount = 5.0 (5%)
IsPercentage = true
AnnualRateMonthlyCompounding = false
Value = 10,000
PaymentCount = 1

Multiplier = 1 + (5.0 / 100) = 1.05
Result = 10,000 * (1.05^1 - 1) = 10,000 * 0.05 = $500
```

### Annual Rate Monthly Compounding Example

```
Amount = 6.0 (6% annual)
IsPercentage = true
AnnualRateMonthlyCompounding = true
Value = 10,000
PaymentCount = 1 (one month)

Multiplier = (1 + 0.06)^(1/12) = 1.004867551...
Result = 10,000 * (1.004867551^1 - 1) = 10,000 * 0.004867551 = $48.68
```

---

## Sanitization Rules

Sanitize is called automatically during config load and before simulation start.

### Identity Sanitization

- **Id registration:** Registers ID with IdGenerator, checks for duplicates
- **Name generation:** If Name is empty, sets to `"AmountFreq-{Id}"`

### Annual Rate Monthly Compounding Enforcement

When `AnnualRateMonthlyCompounding == true`, the following properties are enforced:

| Property | Enforced Value | Reason |
|----------|----------------|--------|
| IsPercentage | true | Annual rate must be a percentage |
| Schedule.Frequency | Monthly | Compounding requires monthly payment triggers |
| Schedule.DayOfMonth | 31 | Last day of month for consistent compounding |
| Schedule.MonthOfYear | null | Must trigger every month, not just one month per year |

**Warning logs:** Any corrections are logged at WARN level.

### Schedule Sanitization

`Schedule.Sanitize()` is called to validate TriggerFrequency properties.

---

## Common Patterns

### Pattern 1: Salary Income

```json
{
  "Name": "Biweekly Salary",
  "Enabled": true,
  "Amount": 2500,
  "IsPercentage": false,
  "Schedule": {
    "Frequency": "Fortnightly",
    "DayOfWeek": "Friday"
  }
}
```

### Pattern 2: Rental Income (Annual Increase)

```json
{
  "Name": "Rental Income",
  "Enabled": true,
  "Amount": 2000,
  "IsPercentage": false,
  "Schedule": {
    "Frequency": "Monthly",
    "DayOfMonth": 1
  }
}
```

(Pair with a separate AmountFreq for annual rent increases, or use EventItem with percentage increase)

### Pattern 3: Compound Interest (Savings)

```json
{
  "Name": "Savings Interest",
  "Enabled": true,
  "Amount": 3.5,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

### Pattern 4: Credit Card APR

```json
{
  "Name": "Credit Card Interest",
  "Enabled": true,
  "Amount": 18.99,
  "IsPercentage": true,
  "AnnualRateMonthlyCompounding": true
}
```

### Pattern 5: Annual Dividend Yield

```json
{
  "Name": "Dividend Yield",
  "Enabled": true,
  "Amount": 2.5,
  "IsPercentage": true,
  "Schedule": {
    "Frequency": "Annual",
    "DayOfMonth": 30,
    "MonthOfYear": "June"
  }
}
```

### Pattern 6: One-Time Payment

```json
{
  "Name": "Laptop Purchase",
  "Enabled": true,
  "Amount": 1500,
  "IsPercentage": false,
  "Schedule": {
    "Frequency": "Monthly",
    "DayOfMonth": 15,
    "TriggerLimit": 1
  }
}
```

(TriggerLimit=1 ensures this payment only occurs once)

---

## Debugging

### ToString() Format

AmountFreq displays in PropertyGrid and logs with this format:

**Disabled:**
```
Name: (Disabled)
```

**Amount = 0:**
```
Name: N/A
```

**Fixed value:**
```
Name: $100.00 @ Monthly on day 15
```

**Percentage:**
```
Name: 5.00% @ Annual in June on day 30
```

**Annual rate monthly compounding:**
```
Name: 4.50% (Annual Rate, Monthly Compounding)
```

### Debug Logging

AmountFreq logs at INFO and DEBUG levels:

```
[AmountFreq-af123456] Payment count: 2
[AmountFreq-af123456] Using destination value $5000.00 for percentage calculation (PercentageBasis=Destination)
```

**Enable debug logs:**
```bash
Falculator.exe --config "config.json" --run --loglevel DEBUG
```

---

## Related Models

- **TriggerFrequency** - Scheduling system used by AmountFreq.Schedule
- **FinancialItem** - Contains CashIn, CashOut, and Interest AmountFreqs
- **EventItem** - Contains Amount AmountFreq for transfers with flexible PercentageBasis
- **TriggerConditions** - Can reference AmountFreq schedule for timing coordination

---

## JSON Schema Reference

Full schema: `Data/Schema/AmountFreq.schema.json`

**Required properties:**
- Id (auto-generated)
- Enabled
- Amount
- IsPercentage
- Schedule

**Optional properties:**
- Name (auto-generated if missing)
- PercentageBasis (defaults to Source)
- AnnualRateMonthlyCompounding (defaults to false)

---

## Example Configurations

AmountFreq is used extensively in all example configs:

- **Data/examples/HomeLoan.json** - Salary income, mortgage payments, property expenses
- **Data/examples/001.Test.json** - Compound interest demonstration
- **Data/examples/InteractiveMode.Test.json** - Credit card APR
- **Data/examples/Shares.Test.json** - Dividend yield

---

## Implementation Details

**Source file:** `Models/AmountFreq.cs`

**Key methods:**
- `ProcessPayment()` - Lines 89-163: Core calculation logic
- `Sanitize()` - Lines 165-209: Validation and enforcement
- `ToString()` - Lines 211-237: Display formatting

**Dependencies:**
- `TriggerFrequency` - Determines payment timing and count
- `IdGenerator` - ID generation and registration
- `DebugLogger` - Logging infrastructure

**Used by:**
- `FinancialItem.CashIn` - Incoming payments
- `FinancialItem.CashOut` - Outgoing payments
- `FinancialItem.Interest` - Growth/decay rates
- `EventItem.Amount` - Transfer amounts with flexible PercentageBasis

---
