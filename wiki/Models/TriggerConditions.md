# TriggerConditions Model

Comprehensive conditional activation system combining tag-based dependencies, value thresholds, age requirements, and date ranges.

---

## Overview

TriggerConditions is a powerful orchestration layer that determines when a FinancialItem or EventItem should activate. It combines multiple trigger types with flexible logic to create sophisticated conditional behavior:
- **TagRules**: Activate based on other items' states (see TagRules.md)
- **ValueTriggers**: Age thresholds, balance requirements, liquid asset levels (see ValueTrigger.md)
- **Date Ranges**: Start and end date boundaries
- **Match Logic**: Combine conditions with All/Any/None logic

TriggerConditions is used in two contexts:
1. **FinancialItem.SelfTrigger**: Controls when an item activates/deactivates
2. **EventItem.Triggers**: Controls when a transfer/payment executes

**Key Features:**
- Combine multiple trigger types with AND/OR logic
- Tag-based dependencies on other items (no hard-coded IDs)
- Value-based thresholds (age, balance, liquid assets, main savings)
- Date range filtering
- Flexible match logic (All, Any, None)
- Backward compatibility with legacy MinAge/MaxAge properties

---

## Quick Start

### Age-Based Activation (Retirement)

```json
"SelfTrigger": {
  "Name": "Retirement Trigger",
  "Age": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 67
  }
}
```

Activates when user reaches age 67.

### Balance + Tag Dependency

```json
"SelfTrigger": {
  "Name": "Investment Purchase",
  "TriggerMatchType": "All",
  "MainSavingsBalance": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 50000
  },
  "TagRules": [
    {
      "Name": "Property Must Be Active",
      "Tags": ["property"],
      "MatchType": "All",
      "MatchValue": true
    }
  ]
}
```

Activates when savings >= $50,000 AND all items tagged "property" are active.

### Date Range + Age

```json
"SelfTrigger": {
  "Name": "Temporary Pension",
  "TriggerMatchType": "All",
  "StartDate": "2030-07-01",
  "EndDate": "2040-06-30",
  "Age": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 65
  }
}
```

Active between 2030-2040 AND when age >= 65.

---

## Properties

### Identity Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Id | string | auto-generated | 8-character unique identifier (read-only) |
| Name | string | "TriggerConditions-{Id}" | Human-readable name for debugging |

### Match Logic Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| TriggerMatchType | MatchLogic | All | How to combine ALL condition types (tags, dates, values) |
| TriggerMatchValue | bool | true | Value to match against condition results |
| TagMatchType | MatchLogic | All | How to combine multiple TagRules |
| MatchValue | bool | true | Value to match for tag conditionals (deprecated, use TagRules[].MatchValue) |

### Tag-Based Conditions

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| TagRules | List<TagConditionals> | [] | Tag-based dependencies on other items (see TagRules.md) |

### Date-Based Conditions

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| StartDate | DateOnly? | null | Item active on or after this date (null = no restriction) |
| EndDate | DateOnly? | null | Item active on or before this date (null = no restriction) |
| StartDateProxy | DateTime? | null | GUI proxy for StartDate |
| EndDateProxy | DateTime? | null | GUI proxy for EndDate |

### Value-Based Conditions

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Age | ValueTrigger | disabled | Age-based activation (see ValueTrigger.md) |
| LiquidAssets | ValueTrigger | disabled | Total liquid assets threshold |
| MainSavingsBalance | ValueTrigger | disabled | Main savings account balance threshold |

### Deprecated Properties (Backward Compatibility)

| Property | Type | Status | Migration |
|----------|------|--------|-----------|
| MinAge | int? | Obsolete | Migrates to Age.Operator=GreaterThanOrEqual |
| MaxAge | int? | Obsolete | Migrates to Age.Operator=LessThanOrEqual |

---

## MatchLogic Enum

Controls how multiple conditions are combined.

| Value | Numeric | Behavior |
|-------|---------|----------|
| All | 0 | ALL conditions must pass (AND logic) |
| Any | 1 | ANY condition can pass (OR logic) |
| None | 2 | NO conditions should pass (inverse logic) |

**Default:** All (AND logic)

---

## Evaluation Logic

### EvaluateTrigger Method

Evaluates all configured conditions and returns true if the overall trigger should activate.

**Signature:**
```csharp
bool EvaluateTrigger(
    List<FinancialItem> allItems,
    FinancialItem owner,
    DateOnly simDate,
    int currentAge,
    decimal currentLiquidAssets,
    decimal currentMainSavingsBalance
)
```

**Parameters:**
- `allItems` - All items in the simulation (for tag lookup)
- `owner` - The item that owns this trigger (excluded from self-trigger checks)
- `simDate` - Current simulation date
- `currentAge` - User's current age
- `currentLiquidAssets` - Total liquid assets across all items
- `currentMainSavingsBalance` - Main savings account balance

**Return value:** true if trigger conditions are met

### Evaluation Algorithm

1. **Initialize results list** (stores boolean result for each condition type)

2. **Evaluate TagRules** (if any are configured):
   - For each TagConditional in TagRules:
     - Skip if Enabled=false
     - For each tag in the TagConditional.Tags list:
       - Find all items (except owner) matching the tag
       - Evaluate based on MatchType (All/Any/None) and MatchValue
     - All tags in the TagConditional must pass (AND logic within a single rule)
   - Combine all TagConditional results using TagMatchType (All/Any/None)
   - Add combined result to results list

3. **Evaluate Date Conditions**:
   - If StartDate is set: add (simDate >= StartDate) to results
   - If EndDate is set: add (simDate <= EndDate) to results

4. **Evaluate Value Triggers**:
   - If Age.Enabled: evaluate Age.TriggerCheck(currentAge), add to results
   - If LiquidAssets.Enabled: evaluate LiquidAssets.TriggerCheck(currentLiquidAssets), add to results
   - If MainSavingsBalance.Enabled: evaluate MainSavingsBalance.TriggerCheck(currentMainSavingsBalance), add to results

5. **Check for empty results**:
   - If no conditions are configured (results.Count == 0): return false

6. **Combine all results using TriggerMatchType**:
   - All: return true if all results match TriggerMatchValue
   - Any: return true if any result matches TriggerMatchValue
   - None: return true if no results match TriggerMatchValue

7. **Record triggers** (only if overall result is true):
   - Call RecordTrigger() on Age, LiquidAssets, MainSavingsBalance (if enabled)

### Example Evaluation

**Configuration:**
```json
{
  "TriggerMatchType": "All",
  "Age": { "Enabled": true, "Operator": ">=", "ComparisonValue": 65 },
  "MainSavingsBalance": { "Enabled": true, "Operator": ">=", "ComparisonValue": 100000 },
  "TagRules": [
    { "Tags": ["property"], "MatchType": "Any", "MatchValue": true }
  ]
}
```

**Evaluation:**
1. Age check: currentAge=66 >= 65 -> true
2. MainSavingsBalance check: currentMainSavingsBalance=120000 >= 100000 -> true
3. TagRules check: Any item with "property" tag is active -> true
4. Combine with TriggerMatchType=All: true AND true AND true -> **TRUE**

---

## Match Logic Combinations

### TriggerMatchType vs TagMatchType

**TriggerMatchType**: Combines ALL condition types (tags, dates, values)

**TagMatchType**: Combines multiple TagRules (within the tag condition block)

**Example 1: All tags AND age**
```json
{
  "TriggerMatchType": "All",
  "TagMatchType": "All",
  "TagRules": [
    { "Tags": ["property"], "MatchType": "Any", "MatchValue": true },
    { "Tags": ["income"], "MatchType": "Any", "MatchValue": true }
  ],
  "Age": { "Enabled": true, "Operator": ">=", "ComparisonValue": 65 }
}
```

Result: (Any property active AND Any income active) AND age >= 65

**Example 2: Any tag OR age**
```json
{
  "TriggerMatchType": "Any",
  "TagMatchType": "Any",
  "TagRules": [
    { "Tags": ["property"], "MatchType": "All", "MatchValue": true },
    { "Tags": ["stocks"], "MatchType": "All", "MatchValue": true }
  ],
  "Age": { "Enabled": true, "Operator": ">=", "ComparisonValue": 65 }
}
```

Result: (All properties active OR All stocks active) OR age >= 65

---

## Sanitization Rules

Sanitize is called automatically during config load and before simulation start.

### Identity Sanitization

- **Id registration:** Registers ID with IdGenerator, checks for duplicates
- **Name generation:** If Name is empty, sets to `"TriggerConditions-{Id}"`

### Legacy Property Migration

If deprecated MinAge/MaxAge properties are detected:

| Legacy Property | New Property | Migration Logic |
|-----------------|--------------|-----------------|
| MinAge=X | Age.Operator=GreaterThanOrEqual, Age.ComparisonValue=X, Age.Enabled=true | Auto-migrates, clamps to [0,120], logs INFO |
| MaxAge=X | Age.Operator=LessThanOrEqual, Age.ComparisonValue=X, Age.Enabled=true | Auto-migrates, clamps to [1,120], logs INFO |

**After migration:**
- Legacy properties are set to null (prevents serialization)
- Migration is logged at INFO level
- Age clamping warnings logged at WARN level

### Date Range Validation

- If both StartDate and EndDate are set:
  - If StartDate > EndDate: StartDate is set to EndDate (logged at WARN level)

### Child Object Sanitization

- Calls Sanitize() on Age, LiquidAssets, MainSavingsBalance (ValueTriggers)
- Calls Sanitize() on each TagConditional in TagRules

---

## Common Patterns

### Pattern 1: Age-Only Retirement

```json
{
  "Name": "Retirement Trigger",
  "Age": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 67,
    "TriggerLimit": 1
  }
}
```

Fire once when age reaches 67.

### Pattern 2: Savings Threshold with Emergency Fund

```json
{
  "Name": "Investment Purchase Trigger",
  "TriggerMatchType": "All",
  "MainSavingsBalance": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 50000
  },
  "LiquidAssets": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 20000
  }
}
```

Activate when savings >= $50k AND total liquid assets >= $20k.

### Pattern 3: Date-Bounded Income

```json
{
  "Name": "Contract Work",
  "StartDate": "2026-01-01",
  "EndDate": "2028-12-31"
}
```

Active only between 2026-2028.

### Pattern 4: Complex Tag Dependencies

```json
{
  "Name": "Investment Opportunity",
  "TriggerMatchType": "All",
  "TagMatchType": "Any",
  "TagRules": [
    {
      "Name": "All Properties Active",
      "Tags": ["property"],
      "MatchType": "All",
      "MatchValue": true
    },
    {
      "Name": "All Stocks Active",
      "Tags": ["stocks"],
      "MatchType": "All",
      "MatchValue": true
    }
  ],
  "MainSavingsBalance": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 100000
  }
}
```

Activate when savings >= $100k AND (all properties active OR all stocks active).

### Pattern 5: Temporary Pension Window

```json
{
  "Name": "Bridging Pension",
  "TriggerMatchType": "All",
  "StartDate": "2030-01-01",
  "EndDate": "2035-12-31",
  "Age": {
    "Enabled": true,
    "Operator": "GreaterThanOrEqual",
    "ComparisonValue": 60
  }
}
```

Active between 2030-2035 AND age >= 60.

---

## Debugging

### ToString() Format

TriggerConditions displays in PropertyGrid and logs with this format:

```
3 tag rules, 2026-01-01, 2030-12-31
0 tag rules, No Start Date, No End Date
```

### Debug Logging

TriggerConditions logs detailed evaluation at DEBUG level:

```
[TriggerConditions-tc123456] TagConditionals 'All Properties Active' (Id: tagc0001) evaluating...
[TriggerConditions-tc123456] 'property' is satisfied (MatchType: All, MatchValue: true, Items: 2)
[TriggerConditions-tc123456] Owner 'Investment Purchase': Age check - Current: 66, Required: >= 65, Result: PASS
[TriggerConditions-tc123456] Owner 'Investment Purchase': MainSavingsBalance check - Current: $120000.00, Required: Value >= 100000, No Limit, Result: PASS
[TriggerConditions-tc123456] Owner 'Investment Purchase': Final result - 3/3 conditions passed, TriggerMatchType=All, TriggerMatchValue=True, TagMatchType=All, Result: ACTIVE
```

**Enable debug logs:**
```bash
Falculator.exe --config "config.json" --run --loglevel DEBUG
```

**Check logs in:**
- `savepath/simulation.log` (when using --savepath)
- Console output (when not redirected)

### HasAnyConditions Method

Check if any triggers are configured (useful for optimization).

```csharp
bool HasAnyConditions()
```

Returns true if at least one of:
- Age.Enabled
- StartDate.HasValue
- EndDate.HasValue
- LiquidAssets.Enabled
- MainSavingsBalance.Enabled
- TagRules.Any()

---

## EventItem-Specific Triggers

EventItem uses TriggerConditions with an additional ValueTrigger:

### TargetBalanceTrigger (EventItem Only)

**Not available in FinancialItem.SelfTrigger**

Checks the target item's balance for EventItem transfers.

```json
"Triggers": {
  "Name": "Top Up Investment Account",
  "TargetBalanceTrigger": {
    "Enabled": true,
    "Operator": "LessThan",
    "ComparisonValue": 50000
  }
}
```

Transfer executes when target item balance < $50,000.

**Note:** See EventItem.md for full EventItem trigger documentation.

---

## Performance Notes

- Tag evaluation is O(N*M) where N = number of items, M = number of tags
- Self-triggers exclude the owner item to prevent circular dependencies
- Disabled TagConditionals are skipped without evaluation
- Empty TagRules lists short-circuit (no tag evaluation)
- No conditions configured (HasAnyConditions() == false) returns false immediately
- ValueTrigger.TriggerCheck() does NOT increment counters (only RecordTrigger() does)

---

## Test Coverage

See `Data/examples/006.Test.json` for comprehensive test cases covering:
- All MatchLogic combinations (All/Any/None)
- Multiple TagConditionals with TagMatchType
- Age + TagRules combinations
- Date ranges
- Value trigger combinations
- TriggerMatchType with multiple condition types

**Run test:**
```bash
dotnet test --filter "FullyQualifiedName~006.Test"
```

---

## Related Models

- **TagRules** - See TagRules.md for tag-based conditional system
- **ValueTrigger** - See ValueTrigger.md for value comparison logic
- **FinancialItem.SelfTrigger** - Uses TriggerConditions to control item activation
- **EventItem.Triggers** - Uses TriggerConditions to control transfer execution
- **TagConditionals** - Individual tag rule within TriggerConditions.TagRules

---

## JSON Schema Reference

Full schema: `Data/Schema/TriggerConditions.schema.json`

**Required properties:**
- Id (auto-generated)
- TriggerMatchType (defaults to All)
- TriggerMatchValue (defaults to true)
- TagMatchType (defaults to All)
- MatchValue (defaults to true)

**Optional properties:**
- Name (auto-generated if missing)
- TagRules (defaults to empty list)
- StartDate, EndDate (null = no restriction)
- Age, LiquidAssets, MainSavingsBalance (ValueTriggers, disabled by default)

**Deprecated properties (still accepted for backward compatibility):**
- MinAge, MaxAge

---

## Example Configurations

TriggerConditions is used extensively in:

- **Data/examples/006.Test.json** - Comprehensive trigger testing
- **Data/examples/HomeLoan.json** - Age-based retirement triggers
- **Data/examples/005.Test.json** - Loan payoff triggers
- **Data/examples/InteractiveMode.Test.json** - Tag-based dependencies

---

## Implementation Details

**Source file:** `Models/TriggerConditions.cs`

**Key methods:**
- `EvaluateTrigger()` - Lines 190-315: Core evaluation logic
- `HasAnyConditions()` - Lines 321-329: Check if any triggers are configured
- `Sanitize()` - Lines 339-403: Legacy migration, validation, child sanitization
- `ToString()` - Lines 331-333: Display formatting

**Dependencies:**
- `ValueTrigger` - Age, LiquidAssets, MainSavingsBalance properties
- `TagConditionals` - TagRules list
- `MatchLogic` enum (Models/GlobalEnums.cs) - Combination logic
- `IdGenerator` - ID generation and registration
- `DebugLogger` - Logging infrastructure

**Used by:**
- `FinancialItem.SelfTrigger` - Controls item activation (TriggerConditions instance)
- `EventItem.Triggers` - Controls transfer execution (TriggerConditions instance with TargetBalanceTrigger)

---
