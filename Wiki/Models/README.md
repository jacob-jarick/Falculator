# Model Documentation

User-facing model documentation for Falculator's configuration and simulation system.

---

## Overview

This directory contains comprehensive documentation for all user-facing models in Falculator. Each model corresponds to a JSON schema in `Data/Schema/` and represents a component of the simulation configuration.

---

## Core Models

### Configuration

- **[Config](config.md)** - Root configuration container for simulation settings, taxation, and financial items

### Financial Items

- **[FinancialItem](FinancialItem.md)** - Core financial entity (income, expenses, savings, assets, liabilities, loans, shares, credit cards)

### Payment System

- **[AmountFreq](amountFreq.md)** - Payment and frequency system for recurring financial transactions

### Inter-Item Transfers

- **[EventItem](EventItem.md)** - Conditional payment and transfer system with trigger-based activation

---

## Trigger System

### Comprehensive Triggers

- **[TriggerConditions](TriggerConditions.md)** - Comprehensive conditional activation combining tags, dates, ages, and values

### Value Comparisons

- **[ValueTrigger](ValueTrigger.md)** - Simple value comparison for balance, age, and asset thresholds

### Tag-Based Dependencies

- **[TagRules](TagRules.md)** - Flexible tag-based conditional system for item dependencies

---

## Schema Correspondence

All models documented here have corresponding JSON schemas in `Data/Schema/`:

| Model | Schema File |
|-------|-------------|
| Config | Config.schema.json |
| FinancialItem | FinancialItem.schema.json |
| AmountFreq | AmountFreq.schema.json |
| EventItem | EventItem.schema.json |
| TriggerConditions | TriggerConditions.schema.json |
| ValueTrigger | ValueTrigger.schema.json |

**Regenerate schemas:**
```bash
Falculator.exe --generate-schemas
```

Schemas update automatically on build (MSBuild post-build target).

---

## Related Documentation

- **ItemType Documentation:** See `wiki/ItemTypes/` for ItemType-specific patterns and behavior
- **Cash Flow Semantics:** See `wiki/Logic/Flows.md` for CashIn/CashOut/Interest overview
- **Example Configs:** See `Data/examples/` for real-world configuration examples
- **Testing:** See `CLAUDE.md` for testing and validation commands

---
