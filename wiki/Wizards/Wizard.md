# Item Wizard System

**Status:** ? Implemented

Simplified creation of FinancialItems via guided forms instead of raw JSON editing.

---

## Overview

Item Wizards guide users through creating complex FinancialItems with multi-step dialogs.

**Separate system:** Wizard of Wizards (WoW) orchestrates multiple Item Wizards - see `wiki\WizardOfWizards.md`

---

## Quick Start

```bash
# Wizards loaded from: Data/Wizards/*.ini
# Templates loaded from: Data/templates/*.json
```

**Menu location:** `Item Wizard` (top menu bar, after `Items`)

---

## INI File Structure

```ini
[Details]
Name=Home Loan Wizard
Description=Assists in setting up a home loan financial items.
Template=Home Loan.json

[Property 1]
Name=Loan Value
Description=Enter the total value of the loan.
Type=Decimal
Default=400000
JsonPath=$.Value

[Property 2]
Name=Interest Rate
Description=Enter the annual interest rate (in percentage).
Type=Decimal
Default=7.1
JsonPath=$.Interest.Amount
```

### Sections

**[Details]** - Required:
- `Name` - Display name for wizard
- `Description` - Explanation of what wizard creates
- `Template` - JSON template filename (in `Data/templates/`)

**[Property N]** - One or more:
- `Name` - Display name for input field
- `Description` - Help text for user
- `Type` - Data type (String, Integer, Decimal, Boolean, Date)
- `Default` - Pre-filled value
- `JsonPath` - Where to insert value in template (e.g., `$.Value`, `$.Interest.Amount`)

---

## Supported Property Types

- `String` - Text input
- `Integer` - Whole numbers
- `Decimal` - Decimal numbers
- `Boolean` - Checkbox (true/false)
- `Date` - Date picker

---

## Behavior

1. User selects wizard from `Item Wizard` menu
2. Wizard loads template JSON from `Data/templates/`
3. Multi-step dialog collects user input for each property
4. Values are applied to template JSON via JsonPath
5. New FinancialItem added to config with auto-generated IDs
6. Item appears in `Items` menu

---

## Implementation Details

**Code Location:**
- `UI\Form1.TemplatesWizards.cs` - Menu integration, wizard launching
- `WizardDialog.cs` - Multi-step wizard UI
- `Services\WizardLoaderService.cs` - Loads and validates wizard INI files
- `Services\IniFileParser.cs` - Parses INI file format
- `Services\WizardJsonService.cs` - Applies values to template JSON
- `Models\WizardModels.cs` - Data models (WizardConfiguration, WizardProperty, etc.)

**Validation:**
- All wizards validated on load
- Template file existence checked
- JsonPath validation against template JSON
- Invalid wizards logged and skipped (appear in Debug Log)

---

## Testing

```bash
# Test single item wizard
# 1. Falculator.exe (launch GUI)
# 2. File > New
# 3. Item Wizard > Home Loan Wizard
# 4. Enter values, complete wizard
# 5. Verify item appears in Items menu
```

---

## File Naming Conventions

- **Wizard INI:** `<Name>.ini` (e.g., `HomeLoan.ini`, `WeeklySalary.ini`)
- **Template JSON:** `<Name>.json` (e.g., `Home Loan.json`, `Weekly Salary.json`)

**Directory structure:**
```
Data/
??? Wizards/
?   ??? HomeLoan.ini
?   ??? WeeklySalary.ini
??? templates/
    ??? Home Loan.json
    ??? Weekly Salary.json
```

---

## Creating New Wizards

### Method 1: Manual Creation

1. Create template JSON in `Data/templates/`
2. Create wizard INI in `Data/Wizards/`
3. Define properties with JsonPaths
4. Restart Falculator to load new wizard

### Method 2: Save Existing Item as Template

1. Create and configure FinancialItem in GUI
2. `Items` menu > `Save Current Item as Template`
3. Template saved to `Data/templates/`
4. Manually create wizard INI to reference it

---

## Design Notes

**Why separate wizards and templates?**
- Templates are reusable (one template, multiple wizards with different defaults)
- Wizards define UI flow and user experience
- Easy to maintain (change template, all wizards update automatically)

**Why JsonPath?**
- Flexible - can target any property in complex nested JSON
- Standard - well-understood syntax
- Validated - checks paths exist before wizard loads

