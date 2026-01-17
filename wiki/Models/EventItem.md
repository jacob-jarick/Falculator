# EventItem Model

Conditional payment and transfer system for inter-item cash flows with trigger-based activation and state changes.

---

## Overview

EventItem represents a payment or transfer from one FinancialItem to another, executed when specified trigger conditions are met. It enables sophisticated financial modeling scenarios:
- **Conditional transfers**: "Transfer 10% of savings to investments when age >= 65"
- **State changes**: "Enable pension when retirement age reached"
- **Asset liquidation**: "Sell property when value exceeds $500k"
- **Scheduled payments**: "Monthly transfer from salary to savings"
- **Rebalancing**: "Top up emergency fund to $10k monthly"

EventItems are attached to FinancialItems via the `EventItems` property (list of EventItem objects).

**Key Features:**
- Trigger-based execution (age, balance, tags, dates - see TriggerConditions.md)
- Bidirectional transfers (CashOut = push to target, CashIn = pull from target)
- Percentage-based transfers with Source or Destination basis
- Target state changes (Enable, Disable, Toggle)
- Asset liquidation support
- Self-reference protection

---

## Quick Start

### Monthly Savings Transfer

```json
{
  "Name": "Monthly Savings",
  "Enabled": true,
  "TargetName": "Savings Account",
  "CashOut": {
    "Enabled": true,
    "Amount": 500,
    "IsPercentage": false,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 15 }
  }
}
```

Transfers $500 from this item to "Savings Account" on the 15th of each month.

### Age-Triggered Pension

```json
{
  "Name": "Activate Pension",
  "Enabled": true,
  "TargetName": "Pension Income",
  "SetStateOnTrigger": true,
  "TargetStateOnTrigger": "Enable",
  "Triggers": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 67,
      "TriggerLimit": 1
    }
  }
}
```

Enables "Pension Income" item once when age reaches 67.

---

## Properties

### Identity Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Id | string | auto-generated | 8-character unique identifier (read-only) |
| Name | string | "EventItem-{Id}" | Human-readable name for debugging |
| Description | string | "" | Optional description of purpose |
| Enabled | bool | true | Controls whether this EventItem is active |

### Trigger Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Triggers | TriggerConditions | empty | Criteria for execution (see TriggerConditions.md) |

### Target Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| TargetId | string | "" | Target item ID (auto-synchronized with TargetName) |
| TargetName | string | "" | Target item name (user-configurable, dropdown in GUI) |
| SetStateOnTrigger | bool | false | Change target's EnabledBySim on trigger |
| TargetStateOnTrigger | StateAction | Disable | Enable, Disable, or Toggle target state |

### Payment Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| CashOut | AmountFreq | disabled | Push payment from source to target (see amountFreq.md) |
| CashIn | AmountFreq | disabled | Pull payment from target to source (see amountFreq.md) |
| Liquidate | bool | false | Liquidate target asset to main savings (mutually exclusive with CashOut/CashIn) |

---

## StateAction Enum

Controls what happens to the target item when the EventItem triggers.

| Value | Numeric | Behavior |
|-------|---------|----------|
| Enable | 0 | Set target's EnabledBySim = true |
| Disable | 1 | Set target's EnabledBySim = false |
| Toggle | 2 | Invert target's EnabledBySim state |

**Default:** Disable

**Note:** Only applies if SetStateOnTrigger = true

---

## Transfer Mechanics

### CashOut (Push Payment)

Transfers funds FROM the source item TO the target item.

**Fixed amount:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 1000,
  "IsPercentage": false,
  "Schedule": { "Frequency": "Monthly" }
}
```

Transfers $1,000 per month from source to target.

**Percentage of source:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 10,
  "IsPercentage": true,
  "PercentageBasis": "Source",
  "Schedule": { "Frequency": "Monthly" }
}
```

Transfers 10% of source item's balance per month to target.

**Percentage of destination:**
```json
"CashOut": {
  "Enabled": true,
  "Amount": 5,
  "IsPercentage": true,
  "PercentageBasis": "Destination",
  "Schedule": { "Frequency": "Monthly" }
}
```

Transfers an amount equal to 5% of target item's balance per month (useful for "top-up" scenarios).

### CashIn (Pull Payment)

Transfers funds FROM the target item TO the source item.

**Example: Dividend from investment**
```json
"CashIn": {
  "Enabled": true,
  "Amount": 2.5,
  "IsPercentage": true,
  "PercentageBasis": "Destination",
  "Schedule": { "Frequency": "Annual" }
}
```

Receives 2.5% of target's value annually (dividend simulation).

### Bidirectional Transfers

CashOut and CashIn can both be enabled (processed sequentially: CashOut first, then CashIn).

**Warning:** This is rarely useful and will generate a WARN log message during sanitization.

### Liquidate Target

Sells the target asset, transfers all proceeds to main savings, and disables the target.

```json
{
  "Name": "Sell Property",
  "TargetName": "Rental Property",
  "Liquidate": true,
  "Triggers": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 70
    }
  }
}
```

**Requirements:**
- Target must be an Asset ItemType
- Mutually exclusive with CashOut/CashIn (sanitization will disable them)

**Behavior:**
- Transfers target's Value to main savings
- Sets target's EnabledBySim = false
- Target's Value = 0

---

## Sanitization Rules

Sanitize is called automatically during config load and before simulation start.

### Identity Sanitization

- **Id registration:** Registers ID with IdGenerator, checks for duplicates
- **Name generation:** If Name is empty, sets to `"EventItem-{Id}"`

### Child Object Sanitization

- Calls Sanitize() on CashOut, CashIn, Triggers

### Percentage Basis Logging

If CashOut or CashIn uses PercentageBasis=Destination:
- Logs INFO message for visibility
- This is a valid but advanced feature

### Bidirectional Transfer Warning

If both CashOut and CashIn are enabled with non-zero amounts:
- Logs WARN message
- Both will be processed sequentially (CashOut first)

### Liquidate Mutual Exclusion

If Liquidate = true:
- If CashOut is configured: disables it, sets Amount=0, logs WARN
- If CashIn is configured: disables it, sets Amount=0, logs WARN

### Target Reference Validation

SanitizeTargetReference() is called separately after all items are loaded.

**CASE 1: TargetId is set**
- Looks up item by ID
- If found: syncs TargetName to actual item name
- If not found: clears TargetId, TargetName, disables EventItem, logs WARN

**CASE 2: TargetId is empty, TargetName is set**
- Looks up item by name
- If found: sets TargetId for future reference, logs INFO
- If not found: clears TargetName, disables EventItem, logs WARN

**CASE 3: Both empty**
- Logs DEBUG if Enabled=true
- EventItem will not execute (valid state for disabled items)

**CASE 4: Self-reference check**
- If TargetId == owner.Id: clears both, disables EventItem, logs WARN

---

## Trigger Evaluation

EventItem execution depends on:
1. **Enabled = true**
2. **Valid target configured** (TargetId not empty)
3. **Triggers.EvaluateTrigger() returns true**

If all conditions are met:
- CashOut is processed (if configured)
- CashIn is processed (if configured)
- Liquidate is processed (if configured)
- Target state change is applied (if SetStateOnTrigger=true)

**Note:** See TriggerConditions.md for full trigger evaluation logic.

---

## Common Patterns

### Pattern 1: Monthly Savings Contribution

```json
{
  "Name": "Monthly Savings",
  "TargetName": "Savings Account",
  "CashOut": {
    "Enabled": true,
    "Amount": 1000,
    "Schedule": { "Frequency": "Monthly", "DayOfMonth": 15 }
  }
}
```

No triggers needed - executes every month unconditionally.

### Pattern 2: Emergency Fund Top-Up

```json
{
  "Name": "Top Up Emergency Fund",
  "TargetName": "Emergency Fund",
  "CashOut": {
    "Enabled": true,
    "Amount": 500,
    "Schedule": { "Frequency": "Monthly" }
  },
  "Triggers": {
    "TargetBalanceTrigger": {
      "Enabled": true,
      "Operator": "LessThan",
      "ComparisonValue": 10000
    }
  }
}
```

Transfers $500 monthly only when emergency fund balance < $10,000.

### Pattern 3: Retirement Pension Activation

```json
{
  "Name": "Activate Pension",
  "TargetName": "Pension Income",
  "SetStateOnTrigger": true,
  "TargetStateOnTrigger": "Enable",
  "Triggers": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 67,
      "TriggerLimit": 1
    }
  }
}
```

Enables pension income item once at age 67 (no cash transfer).

### Pattern 4: Investment Rebalancing

```json
{
  "Name": "Rebalance to Stocks",
  "TargetName": "Stock Portfolio",
  "CashOut": {
    "Enabled": true,
    "Amount": 10,
    "IsPercentage": true,
    "PercentageBasis": "Source",
    "Schedule": { "Frequency": "Annual" }
  },
  "Triggers": {
    "MainSavingsBalance": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 50000
    }
  }
}
```

Transfers 10% of source balance to stocks annually when savings >= $50k.

### Pattern 5: Property Liquidation at Retirement

```json
{
  "Name": "Sell Rental Property",
  "TargetName": "Rental Property",
  "Liquidate": true,
  "Triggers": {
    "Age": {
      "Enabled": true,
      "Operator": "GreaterThanOrEqual",
      "ComparisonValue": 70,
      "TriggerLimit": 1
    }
  }
}
```

Liquidates rental property once at age 70.

### Pattern 6: Conditional Loan Payment

```json
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
```

Makes extra $1,000 monthly loan payments only when savings >= $20k.

---

## Shares ItemType Support

### Buying Shares

EventItem with CashOut targeting a Shares item:
- Calculates shares affordable: `Floor(Amount / UnitPrice)`
- Purchases whole shares only
- Leftover cash stays in source item
- Updates target's TotalCostBase for CGT tracking

### Selling Shares

EventItem with CashIn from a Shares source:
- Calculates shares to sell: `Ceiling(Amount / UnitPrice)`
- Capped at available shares
- Sale proceeds transferred to source item

**Note:** See Shares.md for detailed Shares ItemType documentation.

---

## Debugging

### Debug Logging

EventItem execution is logged at INFO and DEBUG levels:

```
[EventItem-ev123456] Evaluating trigger for 'Monthly Savings'
[EventItem-ev123456] CashOut: Transferring $1000.00 from 'Salary' to 'Savings Account'
[EventItem-ev123456] Target state changed: Enabling 'Pension Income'
```

**Enable debug logs:**
```bash
Falculator.exe --config "config.json" --run --loglevel DEBUG
```

**Check logs in:**
- `savepath/simulation.log` (when using --savepath)
- Console output (when not redirected)

### Common Issues

**Issue:** EventItem never executes.

**Solution:** Check:
1. Enabled = true
2. TargetId is not empty (check logs for "Target ID not found" warnings)
3. Triggers.HasAnyConditions() returns true
4. Trigger conditions are met (check trigger evaluation logs)

**Issue:** Transfer amount is zero.

**Solution:** Check:
1. CashOut/CashIn.Enabled = true
2. CashOut/CashIn.Amount != 0
3. Schedule frequency is triggering (check payment count logs)
4. Source has sufficient balance

**Issue:** Target not found during simulation.

**Solution:** Check:
1. TargetName matches exactly (case-sensitive)
2. Target item exists in config
3. No circular dependencies (source != target)
4. Check sanitization logs for warnings

---

## Related Models

- **TriggerConditions** - See TriggerConditions.md for trigger evaluation
- **AmountFreq** - See amountFreq.md for payment/transfer calculations
- **FinancialItem** - Contains EventItems list
- **ValueTrigger** - See ValueTrigger.md for value-based conditions
- **TagRules** - See TagRules.md for tag-based dependencies

---

## JSON Schema Reference

Full schema: `Data/Schema/EventItem.schema.json`

**Required properties:**
- Id (auto-generated)
- Name (auto-generated if missing)
- Enabled (defaults to true)
- Triggers (TriggerConditions object)
- CashOut, CashIn (AmountFreq objects, disabled by default)

**Optional properties:**
- Description
- TargetId, TargetName (synced automatically)
- SetStateOnTrigger (defaults to false)
- TargetStateOnTrigger (defaults to Disable)
- Liquidate (defaults to false)

---

## Example Configurations

EventItem is used extensively in:

- **Data/examples/HomeLoan.json** - Mortgage payments, salary to savings transfers
- **Data/examples/006.Test.json** - Complex trigger scenarios with state changes
- **Data/examples/InteractiveMode.Test.json** - Credit card payments
- **Data/examples/Shares.Test.json** - Share purchase/sale transactions

---

## Implementation Details

**Source file:** `Models/EventItem.cs`

**Key methods:**
- `Sanitize()` - Lines 122-193: Validation, child sanitization, mutual exclusion
- `SanitizeTargetReference()` - Lines 201-290: Target lookup, sync, self-reference check

**Processing:**
- Executed in `SimFrame.CalculateItem()` - STAGE 3 (EventItem processing)
- Trigger evaluation via `Triggers.EvaluateTrigger()`
- Transfer execution via `CashOut.ProcessPayment()` and `CashIn.ProcessPayment()`

**Dependencies:**
- `TriggerConditions` - Trigger evaluation
- `AmountFreq` - Payment calculation
- `StateAction` enum (EventItem.cs) - State change actions
- `IdGenerator` - ID generation and registration
- `DebugLogger` - Logging infrastructure

**Used by:**
- `FinancialItem.EventItems` - List of EventItem objects attached to each FinancialItem
- `SimFrame.CalculateItem()` - Processes EventItems during simulation

---
