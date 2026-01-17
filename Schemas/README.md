# Falculator JSON Schemas

This directory contains JSON Schema definitions for Falculator's data models. These schemas are automatically generated during build and provide validation and autocompletion support for configuration files.

## Schema Files

- **Config.schema.json** - Root configuration file structure
  - Contains simulation settings, user information, and references to financial items
  - Top-level schema for `.json` config files

- **FinancialItem.schema.json** - Financial instrument definitions
  - Savings accounts, investment properties, income sources, liabilities
  - Core building block of simulations

- **AmountFreq.schema.json** - Amount and frequency specifications
  - Revenue, expenditure, and interest calculations
  - Supports fixed amounts or percentage-based values

- **EventItem.schema.json** - Payment/transfer event definitions
  - Automated transfers between financial items
  - Conditional payment triggers

- **TriggerConditions.schema.json** - Conditional activation rules
  - Date-based, balance-based, income-based triggers
  - Tag-based rules for item activation/deactivation

- **ValueTrigger.schema.json** - Value-based trigger conditions
  - Threshold-based activation logic
  - Used within TriggerConditions

## Usage

### For LLM Agents (Claude, GPT, etc.)

When assisting with Falculator config generation or modification, reference these schemas to:
- Validate config structure before saving
- Provide accurate property names and types
- Understand enum values and constraints
- Generate compliant configuration examples

### For Developers

1. **Validation**: Use schemas to validate config files programmatically
2. **IDE Support**: Configure your IDE to use these schemas for JSON autocompletion
3. **Documentation**: Schemas serve as machine-readable API documentation

### For Users

Most users won't interact with schemas directly, but they enable:
- Better error messages when config files are malformed
- IDE autocompletion when editing JSON configs manually
- Validation tools to check config correctness

## Regeneration

Schemas are automatically regenerated on every build. To manually regenerate:

```bash
Falculator.exe --generate-schemas
```

## Schema Format

Schemas follow [JSON Schema Draft 04](http://json-schema.org/draft-04/schema#) specification and include:
- Property types and descriptions
- Enum value mappings
- Required vs optional fields
- Nested object definitions
- Array item constraints

## Notes

- Schemas reflect the C# model structure at build time
- Property descriptions come from XML doc comments and `[Description]` attributes
- Enum values are documented with their underlying integer values
- Nullable properties are represented with `["null", "type"]` syntax
