# TagRules System

Flexible tag-based conditional system for controlling when FinancialItems become active based on the state of other items.

---

## Overview

TagRules allow you to create sophisticated dependencies between FinancialItems using tags. Instead of hard-coding references to specific items, you can group items with tags and create rules based on whether those tagged items are active or inactive.

**Key Features:**
- Tag-based item dependencies (no hard-coded IDs needed)
- Supports "all" vs "any" matching logic
- Can check for active OR inactive states
- Multiple rules can be combined with AND/OR logic
- Empty tag lists always evaluate to true

---

## Quick Start

```json
"SelfTrigger": {
  "TagRules": [
    {
      "Name": "Require Active Properties",
      "Description": "All property items must be active",
      "Tags": ["property"],
      "MatchType": "All",
      "MatchValue": true
    }
  ],
  "TagMatchType": true
}
```

This item will only activate when ALL items tagged with "property" are active (EnabledBySim == true).

---

## Core Concepts

### Tags

Tags are simple string labels you add to FinancialItems to group them:

```json
{
  "Name": "Rental Property 1",
  "Tags": ["property", "rental", "investment"]
}
```

### TagConditionals

A TagConditional is a single rule that checks the state of items matching one or more tags.

**Properties:**
- `Name` - Human-readable name for debugging
- `Description` - Optional explanation of what the rule does
- `Enabled` - Whether this rule is active (can disable without deleting)
- `Tags` - List of tag names to check (e.g., ["property", "investment"])
- `MatchType` - Defines matching logic: All (all items must match), Any (at least one item must match), or None (no items should match)
- `MatchValue` - The value to match against EnabledBySim: true (items must be enabled) or false (items must be disabled)

### TriggerConditions.TagRules

`TagRules` is a list of TagConditionals attached to a FinancialItem's SelfTrigger or EventItem's Triggers.

**Top-level property:**
- `TagMatchType` - If true, ALL TagConditionals in the list must pass. If false, ANY passing rule is sufficient.

---

## Logic Tables

### MatchType and MatchValue Combinations

The behavior of a TagConditional depends on the MatchType enum and MatchValue boolean:

| MatchType | MatchValue | Behavior |
|-----------|------------|----------|
| All       | true       | ALL items with ANY of the specified tags must be ACTIVE |
| All       | false      | ALL items with ANY of the specified tags must be INACTIVE |
| Any       | true       | ANY item with ANY of the specified tags must be ACTIVE |
| Any       | false      | ANY item with ANY of the specified tags must be INACTIVE |
| None      | true       | NO items with ANY of the specified tags should be ACTIVE |
| None      | false      | NO items with ANY of the specified tags should be INACTIVE |

**Examples:**

**MatchType=All, MatchValue=true:** "All properties must be active"
- Tags: ["property"]
- Items: Property1 (active), Property2 (active) -> PASS
- Items: Property1 (active), Property2 (inactive) -> FAIL

**MatchType=Any, MatchValue=true:** "At least one property must be active"
- Tags: ["property"]
- Items: Property1 (active), Property2 (inactive) -> PASS
- Items: Property1 (inactive), Property2 (inactive) -> FAIL

**MatchType=All, MatchValue=false:** "All investments must be inactive"
- Tags: ["investment"]
- Items: Stock1 (inactive), Stock2 (inactive) -> PASS
- Items: Stock1 (active), Stock2 (inactive) -> FAIL

**MatchType=Any, MatchValue=false:** "At least one investment must be inactive"
- Tags: ["investment"]
- Items: Stock1 (active), Stock2 (inactive) -> PASS
- Items: Stock1 (active), Stock2 (active) -> FAIL

**MatchType=None, MatchValue=true:** "No risky investments should be active"
- Tags: ["risky"]
- Items: Stock1 (inactive), Stock2 (inactive) -> PASS
- Items: Stock1 (active), Stock2 (inactive) -> FAIL
- No items with "risky" tag -> PASS

**MatchType=None, MatchValue=false:** "No safe assets should be inactive"
- Tags: ["safe"]
- Items: Savings1 (active), Savings2 (active) -> PASS
- Items: Savings1 (inactive), Savings2 (active) -> FAIL
- No items with "safe" tag -> PASS

### TagMatchType Behavior

When you have multiple TagConditionals in the TagRules list, TagMatchType controls how their results are combined:

| TagMatchType | Behavior |
|--------------|----------|
| All          | ALL rules in the list must pass (AND logic) |
| Any          | ANY rule in the list can pass (OR logic) |
| None         | NO rules in the list should pass (inverse logic) |

**Example with TagMatchType="All":**
```json
"TagMatchType": "All",
"TagRules": [
  {"Tags": ["property"], "MatchType": "Any", "MatchValue": true},
  {"Tags": ["safe"], "MatchType": "Any", "MatchValue": true}
]
```
Result: Requires ANY property to be active AND ANY safe asset to be active.

**Example with TagMatchType="Any":**
```json
"TagMatchType": "Any",
"TagRules": [
  {"Tags": ["property"], "MatchType": "All", "MatchValue": true},
  {"Tags": ["stocks"], "MatchType": "All", "MatchValue": true}
]
```
Result: Requires (ALL properties active) OR (ALL stocks active).

**Example with TagMatchType="None":**
```json
"TagMatchType": "None",
"TagRules": [
  {"Tags": ["risky"], "MatchType": "Any", "MatchValue": true}
]
```
Result: Activates only when the risky rule does NOT pass (no risky items are active).

### Empty Tags Behavior

| Condition | Result |
|-----------|--------|
| TagRules list is empty ([]) | Always evaluates to true (no tag constraints) |
| TagConditional.Tags is empty ([]) | Always evaluates to true (logs warning) |
| TagConditional.Enabled is false | Rule is skipped (does not contribute to evaluation) |

---

## JSON Schema

### TagConditionals Schema

```json
{
  "Id": "tagr0001",
  "Name": "All Properties Active",
  "Description": "Requires all property items to be active",
  "Enabled": true,
  "Tags": ["property"],
  "MatchType": "All",
  "MatchValue": true
}
```

**Required Properties:**
- `Id` - Auto-generated 8-character identifier (read-only)
- `Name` - Human-readable name
- `Enabled` - Whether this rule is active
- `Tags` - Array of tag strings to match
- `MatchType` - Enum: "All" (all items must match), "Any" (at least one item must match), or "None" (no items should match)
- `MatchValue` - Boolean: true = items must be enabled, false = items must be disabled

**Optional Properties:**
- `Description` - Explanatory text shown in GUI

### TriggerConditions with TagRules

```json
"SelfTrigger": {
  "Id": "trig0001",
  "Name": "Property Dependency Trigger",
  "RequireAll": true,
  "TagMatchType": "All",
  "TagRules": [
    {
      "Id": "tagr0001",
      "Name": "All Properties Active",
      "Description": "Requires all properties to be active",
      "Enabled": true,
      "Tags": ["property"],
      "MatchType": "All",
      "MatchValue": true
    },
    {
      "Id": "tagr0002",
      "Name": "Any Safe Asset Active",
      "Description": "At least one safe asset must be active",
      "Enabled": true,
      "Tags": ["safe"],
      "MatchType": "Any",
      "MatchValue": true
    }
  ]
}
```

**Key Properties:**
- `TagMatchType` - MatchLogic enum (All/Any/None): How to combine TagConditional results
- `TagRules` - Array of TagConditionals objects
- `RequireAll` - Combines tag rules with OTHER trigger conditions (age, date, value triggers)

---

## Common Patterns

### Pattern 1: "Wait for all dependencies"

Activate only when ALL prerequisite items are active.

```json
"TagRules": [
  {
    "Name": "All Prerequisites Active",
    "Tags": ["prerequisite"],
    "MatchType": "All",
    "MatchValue": true
  }
]
```

### Pattern 2: "Activate on any opportunity"

Activate when ANY opportunity item becomes active.

```json
"TagRules": [
  {
    "Name": "Any Opportunity Active",
    "Tags": ["opportunity"],
    "MatchType": "Any",
    "MatchValue": true
  }
]
```

### Pattern 3: "Safety fallback"

Activate when ALL risky investments are inactive (safe mode).

```json
"TagRules": [
  {
    "Name": "All Risky Investments Inactive",
    "Tags": ["risky"],
    "MatchType": "All",
    "MatchValue": false
  }
]
```

### Pattern 4: "No risky investments allowed"

Activate only when NO risky investments are active (stricter than Pattern 3).

```json
"TagRules": [
  {
    "Name": "No Risky Investments Active",
    "Tags": ["risky"],
    "MatchType": "None",
    "MatchValue": true
  }
]
```

### Pattern 5: "Complex dependencies"

Requires both properties AND income sources to be active.

```json
"TagMatchType": "All",
"TagRules": [
  {
    "Name": "At Least One Property",
    "Tags": ["property"],
    "MatchType": "Any",
    "MatchValue": true
  },
  {
    "Name": "At Least One Income",
    "Tags": ["income"],
    "MatchType": "Any",
    "MatchValue": true
  }
]
```

### Pattern 6: "Flexible activation"

Activates if EITHER all properties are active OR all stocks are active.

```json
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
]
```

---

## Validation and Sanitization

TagRules are validated automatically at several points:

### Load Time (Config.Sanitize)
- Generates missing IDs and Names
- Removes duplicate tags from TagConditionals.Tags
- Logs warnings for empty Tags lists

### Validation (ValidateReferences)
- Checks that all referenced tags exist in the TagRegistry
- Disables TagConditionals with invalid tag references
- Logs errors for missing tags

**When ValidateReferences is called:**
- On config load
- On config save
- Before simulation start

**Validation guarantees:** If a simulation starts, all tag references are valid and no broken dependencies exist.

---

## Debugging

### PropertyGrid Display

TagConditionals display in the GUI PropertyGrid with this format:

```
Name [Description] (N tags, All|Any|None, Active|Inactive)
```

**Examples:**
- `All Properties Active [Main dependencies] (1 tags, All, Active)`
- `Any Safe Asset [Fallback condition] (2 tags, Any, Active)`
- `No Risky Investments [Safety condition] (1 tags, None, Active)`
- `TagConditional-tagr0001 (3 tags, All, Inactive)` (if Name is empty)

### Log Output

Tag evaluation is logged at DEBUG level:

```
[Step 5, 2026-05-01] TagConditionals 'All Properties Active' (Id: tagr0001) evaluating...
[Step 5, 2026-05-01] 'property' is satisfied
[Step 5, 2026-05-01] Owner 'Tech Stocks': Final result - 1/1 conditions passed, Result: ACTIVE
```

**To enable debug logs:**
```bash
Falculator.exe --config "config.json" --run --loglevel DEBUG
```

**Check logs in:**
- `savepath/simulation.log` (when using --savepath)
- Console output (when not redirected)

### Common Issues

**Issue:** TagConditional always fails even when items are active.

**Solution:** Check that:
1. Tags match exactly (case-sensitive)
2. MatchType is set correctly (All/Any/None)
3. MatchValue is set correctly (true = items must be enabled, false = items must be disabled)
4. EnabledBySim is true for the target items (not just StartEnabled)

**Issue:** Item never activates despite tags being correct.

**Solution:** Check:
1. TagMatchType setting (may require multiple rules to pass)
2. Other trigger conditions (age, date, value triggers) that must also pass
3. RequireAll setting in TriggerConditions (combines all condition types)

---

## Performance Notes

- Tag evaluation is O(N*M) where N = number of items, M = number of tags
- Self-triggers exclude the owner item to prevent circular dependencies
- Disabled TagConditionals are skipped without evaluation
- Empty TagRules lists short-circuit to true (no iteration)

---

## Test Coverage

See `Data/examples/006.Test.json` for comprehensive test cases covering:
- All four MatchAll/MatchOnTrue combinations
- Multiple TagConditionals with TagMatchType=true
- Multiple TagConditionals with TagMatchType=false
- Empty TagRules behavior
- Disabled TagConditionals behavior

**Run test:**
```bash
dotnet test --filter "FullyQualifiedName~006.Test"
```

---

## Related Documentation

- **Trigger System:** See TriggerConditions.cs for complete trigger evaluation logic
- **Tag Registry:** See Services/TagRegistry.cs for tag validation
- **Item Wizards:** See wiki/wizards/wizard.md for creating items with tags via wizards
- **Validation Flow:** See wiki/devop.md for sanitization and validation process
