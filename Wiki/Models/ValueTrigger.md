# ValueTrigger Model

Simple value comparison system for triggering financial events based on balance, age, or asset thresholds.

---

## Overview

ValueTrigger represents a single comparison rule that evaluates to true or false based on a value (balance, age, asset value, etc.) and a comparison operator. ValueTriggers are used within TriggerConditions to create sophisticated conditional logic for:
- **SelfTrigger**: When should this item activate/deactivate?
- **EventItem Triggers**: When should this transfer/payment execute?
- **Age-based events**: Retirement age, pension eligibility
- **Balance thresholds**: Emergency fund minimums, credit limit maximums
- **Asset milestones**: "Sell when property value exceeds $500k"

**Key Features:**
- Six comparison operators (==, !=, >, >=, <, <=)
- Optional trigger limits (fire once, fire N times, or unlimited)
- Trigger count tracking for analytics
- Backward compatibility with legacy Min/Max properties

---

## Quick Start

### Minimum Balance Threshold

```json
{
  "Name": "Emergency Fund Threshold",
  "Enabled": true,
  "Operator": "GreaterThanOrEqual",
  "ComparisonValue": 10000,
  "TriggerLimit": 0
}
```

This trigger evaluates to true when the value is >= $10,000.

### Age-Based Retirement

```json
{
  "Name": "Retirement Age",
  "Enabled": true,
  "Operator": "GreaterThanOrEqual",
  "ComparisonValue": 65,
  "TriggerLimit": 1
}
```

This trigger fires once when age reaches 65.

---

## Properties

### Identity Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Id | string | auto-generated | 8-character unique identifier (read-only) |
| Name | string | "ValueTrigger-{Id}" | Human-readable name for debugging |

### Comparison Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Enabled | bool | false | Enable or disable this trigger |
| Operator | Comparator | GreaterThanOrEqual | Comparison operator (see table below) |
| ComparisonValue | decimal | 0 | Value to compare against |

### Trigger Limiting Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| TriggerLimit | uint | 0 | Maximum times this trigger can fire (0 = unlimited) |
| TriggerCount | int | 0 | Number of times this trigger has fired (read-only) |
| LastTriggerDate | DateOnly | MinValue | Date when trigger last fired (read-only) |

### Deprecated Properties (Backward Compatibility)

| Property | Type | Status | Migration |
|----------|------|--------|-----------|
| MinEnabled | bool? | Obsolete | Migrates to Operator=GreaterThanOrEqual |
| MinValue | decimal? | Obsolete | Migrates to ComparisonValue |
| MaxEnabled | bool? | Obsolete | Migrates to Operator=LessThanOrEqual |
| MaxValue | decimal? | Obsolete | Migrates to ComparisonValue |

**Migration is automatic:** Sanitize() detects legacy properties and converts them to the new Operator/ComparisonValue system.

---

## Comparator Enum

| Value | Numeric | Symbol | Behavior |
|-------|---------|--------|----------|
| Equal | 0 | == | Value must exactly equal ComparisonValue |
| NotEqual | 1 | != | Value must NOT equal ComparisonValue |
| GreaterThan | 2 | > | Value must be strictly greater than ComparisonValue |
| GreaterThanOrEqual | 3 | >= | Value must be greater than or equal to ComparisonValue |
| LessThan | 4 | < | Value must be strictly less than ComparisonValue |
| LessThanOrEqual | 5 | <= | Value must be less than or equal to ComparisonValue |

**Default:** GreaterThanOrEqual (>= comparison)

---

## Evaluation Logic

### TriggerCheck Method

Returns true if the trigger conditions are met.

**Signature:**
```csharp
bool TriggerCheck(decimal value)
```

**Parameters:**
- `value` - The value to compare (balance, age, asset value, etc.)

**Return value:** true if comparison passes AND trigger limit not exceeded

### Evaluation Steps

1. **Check if enabled:**
   - If `Enabled == false`: return false

2. **Perform comparison:**
   - Apply `Operator` to `value` vs `ComparisonValue`
   - If comparison fails: return false

3. **Check trigger limit:**
   - If `TriggerLimit > 0` AND `TriggerCount >= TriggerLimit`: return false
   - Otherwise: return true

**Important:** TriggerCheck does NOT increment TriggerCount. That happens only when RecordTrigger() is called after the overall trigger evaluation succeeds.

### RecordTrigger Method

Records that the trigger has fired.

**Signature:**
```csharp
void RecordTrigger()
```

**Behavior:**
- Increments `TriggerCount`
- Sets `LastTriggerDate` to current date
- Called by TriggerConditions.ProcessTrigger() when overall evaluation succeeds

---

## Comparison Examples

### Example 1: Minimum Savings Balance

```json
{
  "Operator": "GreaterThanOrEqual",
  "ComparisonValue": 5000
}
```

**Evaluation:**
- value = 4999 -> false
- value = 5000 -> true
- value = 5001 -> true

### Example 2: Maximum Credit Card Balance

```json
{
  "Operator": "LessThan",
  "ComparisonValue": 10000
}
```

**Evaluation:**
- value = 9999 -> true
- value = 10000 -> false
- value = 10001 -> false

### Example 3: Exact Age Check

```json
{
  "Operator": "Equal",
  "ComparisonValue": 65
}
```

**Evaluation:**
- age = 64.9 -> false
- age = 65.0 -> true
- age = 65.1 -> false

**Warning:** Exact equality (==) is fragile with decimal values and simulation step intervals. Prefer >= for age-based triggers.

### Example 4: "Not Zero" Check

```json
{
  "Operator": "NotEqual",
  "ComparisonValue": 0
}
```

**Evaluation:**
- value = 0 -> false
- value = 0.01 -> true
- value = -100 -> true

---

## Sanitization Rules

Sanitize is called automatically during config load and before simulation start.

### Identity Sanitization

- **Id registration:** Registers ID with IdGenerator, checks for duplicates
- **Name generation:** If Name is empty, sets to `"ValueTrigger-{Id}"`

### Legacy Property Migration

If deprecated properties are detected:

| Legacy Property | New Property | Migration Logic |
|-----------------|--------------|-----------------|
| MinEnabled=true, MinValue=X | Operator=GreaterThanOrEqual, ComparisonValue=X | Auto-migrates and logs INFO message |
| MaxEnabled=true, MaxValue=X | Operator=LessThanOrEqual, ComparisonValue=X | Auto-migrates and logs INFO message |

**After migration:**
- Legacy properties are set to null (prevents serialization)
- Enabled is set to true
- Migration is logged at INFO level

**Idempotent:** Running Sanitize multiple times has no effect after initial migration.

---

## Common Patterns

### Pattern 1: Age-Based Retirement

```json
{
  "Name": "Retirement Age",
  "Enabled": true,
  "Operator": "GreaterThanOrEqual",
  "ComparisonValue": 67,
  "TriggerLimit": 1
}
```

Fire once when age >= 67.

### Pattern 2: Emergency Fund Minimum

```json
{
  "Name": "Emergency Fund Minimum",
  "Enabled": true,
  "Operator": "LessThan",
  "ComparisonValue": 10000,
  "TriggerLimit": 0
}
```

Trigger is active whenever savings drop below $10,000 (unlimited occurrences).

### Pattern 3: Property Sale Threshold

```json
{
  "Name": "Property Value Target",
  "Enabled": true,
  "Operator": "GreaterThan",
  "ComparisonValue": 500000,
  "TriggerLimit": 1
}
```

Fire once when property value exceeds $500,000.

### Pattern 4: Loan Paid Off

```json
{
  "Name": "Loan Balance Zero",
  "Enabled": true,
  "Operator": "Equal",
  "ComparisonValue": 0,
  "TriggerLimit": 1
}
```

Fire once when loan balance reaches exactly zero.

**Note:** Loan ItemType has auto-disable logic that makes this pattern unnecessary for most use cases.

### Pattern 5: Credit Card Over Limit

```json
{
  "Name": "Credit Over Limit",
  "Enabled": true,
  "Operator": "GreaterThan",
  "ComparisonValue": 15000,
  "TriggerLimit": 0
}
```

Trigger activates every time credit card balance exceeds $15,000.

---

## Usage in TriggerConditions

ValueTriggers are embedded in TriggerConditions for FinancialItems and EventItems:

### BalanceTrigger (Self Balance)

Checks the item's own balance (Value property).

```json
"SelfTrigger": {
  "BalanceTrigger": {
    "Name": "Low Balance Alert",
    "Enabled": true,
    "Operator": "LessThan",
    "ComparisonValue": 1000
  }
}
```

### AgeTrigger

Checks the simulation age (Config.DOB vs current date).

```json
"SelfTrigger": {
  "AgeTrigger": {
    "Name": "Pension Age",
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 67,
    "TriggerLimit": 1
  }
}
```

### TargetBalanceTrigger (EventItem Only)

Checks the target item's balance for EventItem transfers.

```json
"Triggers": {
  "TargetBalanceTrigger": {
    "Name": "Top Up Investment",
    "Enabled": true,
    "Operator": "LessThan",
    "ComparisonValue": 50000
  }
}
```

---

## Debugging

### ToString() Format

ValueTrigger displays in PropertyGrid and logs with this format:

**Disabled:**
```
Disabled
```

**Enabled:**
```
Value >= 10000, No Limit
Value > 5000, Limit 1
Value == 0, Limit 5
```

### Debug Logging

ValueTrigger creation and migration are logged:

```
[ValueTrigger-vt123456] ValueTrigger created with Operator: GreaterThanOrEqual, ComparisonValue: 10000
[ValueTrigger-vt123456] Sanitize: Migrated from MinEnabled/MinValue to Operator=GreaterThanOrEqual, ComparisonValue=10000
```

---

## Helper Methods

### SetComparison

Configure the trigger in one call.

```csharp
void SetComparison(Comparator op, decimal value)
```

**Example:**
```csharp
valueTrigger.SetComparison(Comparator.GreaterThanOrEqual, 10000);
```

Sets Operator, ComparisonValue, enables the trigger, and calls Sanitize().

### Disable

Disable the trigger and optionally reset the comparison value.

```csharp
void Disable(bool zeroValue = false)
```

**Example:**
```csharp
valueTrigger.Disable(zeroValue: true);
```

Sets Enabled=false, optionally sets ComparisonValue=0, and calls Sanitize().

### ResetTriggerCount

Reset trigger count and last trigger date.

```csharp
void ResetTriggerCount()
```

**Example:**
```csharp
valueTrigger.ResetTriggerCount();
```

Sets TriggerCount=0 and LastTriggerDate=MinValue. Useful for testing or resetting one-time triggers.

---

## Related Models

- **TriggerConditions** - Contains ValueTriggers for BalanceTrigger, AgeTrigger, TargetBalanceTrigger
- **FinancialItem.SelfTrigger** - Uses TriggerConditions to control item activation
- **EventItem.Triggers** - Uses TriggerConditions to control transfer execution
- **Comparator** - Enum defining comparison operators (Models/GlobalEnums.cs)

---

## JSON Schema Reference

Full schema: `Data/Schema/ValueTrigger.schema.json`

**Required properties:**
- Id (auto-generated)
- Enabled
- Operator
- ComparisonValue

**Optional properties:**
- Name (auto-generated if missing)
- TriggerLimit (defaults to 0 = unlimited)
- TriggerCount (read-only, tracked during simulation)
- LastTriggerDate (read-only, tracked during simulation)

**Deprecated properties (still accepted for backward compatibility):**
- MinEnabled, MinValue, MaxEnabled, MaxValue

---

## Example Configurations

ValueTrigger is used in all configs with triggers:

- **Data/examples/006.Test.json** - Comprehensive trigger testing (age, balance, tag combinations)
- **Data/examples/HomeLoan.json** - Age-based retirement triggers
- **Data/examples/005.Test.json** - Loan payoff triggers

---

## Implementation Details

**Source file:** `Models/ValueTrigger.cs`

**Key methods:**
- `TriggerCheck()` - Lines 181-203: Comparison evaluation logic
- `RecordTrigger()` - Lines 206-210: Trigger count tracking
- `Sanitize()` - Lines 110-144: Legacy property migration
- `SetComparison()` - Lines 169-176: Helper for configuration
- `ToString()` - Lines 94-105: Display formatting

**Dependencies:**
- `Comparator` enum (Models/GlobalEnums.cs) - Comparison operators
- `IdGenerator` - ID generation and registration
- `DebugLogger` - Logging infrastructure

**Used by:**
- `TriggerConditions` - Contains BalanceTrigger, AgeTrigger, TargetBalanceTrigger (ValueTrigger instances)
- `FinancialItem.SelfTrigger` - Controls item activation
- `EventItem.Triggers` - Controls transfer execution

---
