# Wizard of Wizards (WoW)

**Purpose:** Orchestrate multiple Item Wizards to create related FinancialItems in a single workflow

---

## Overview

Wizard of Wizards (WoW) allows you to chain multiple Item Wizards together to create complex multi-item setups.

**Example use cases:**
- First home buyer package (mortgage + salary)
- Retirement setup (pension + investments + expenses)
- Business startup (revenue + loan + operating costs)

**How it works:** Each WoW file references existing Item Wizard INI files and launches them sequentially with optional value overrides.

**Separate system:** Item Wizards are documented in `wiki/Wizards/Wizard.md`

---

## Quick Start

```bash
# WoW files: Data/wizardsOfWizards/*.ini
# References: Data/wizards/*.ini (Item Wizards)
```

**Menu location:** `Wizard of Wizards` (top menu bar, after `Item Wizard`)

**Example:** `Data/wizardsOfWizards/FirstHomeBuyer.ini`

---

## Implementation Notes

**Override system (simplified - no type changes):**
- WoW uses `Value` property to override wizard's `Default` value
- Type conversion handled automatically by existing wizard (no `OverrideType` needed)
- Per-template `Value` in `[Template N]` sections overrides global value
- Priority: Template Value > Global OverrideValue > Original Wizard Default

**Template files:**
- WoW files reference Item Wizard INI files (not JSON templates directly)
- Item Wizards located in `Data/wizards/`
- WoW files located in `Data/wizardsOfWizards/`

---

## INI File Structure

```ini
[Settings]
Name=First Home Buyer Package
Description=Creates house asset, mortgage loan, and related items
OverrideValue=500000             # (Optional) Global default for all sub-wizards

[Template 1]
file=HomeLoan.ini                # (Required) Item Wizard filename from Data/wizards/
Name=Home Mortgage Loan          # (Optional) Override wizard name
Description=Mortgage for home    # (Optional) Override wizard description
Value=400000                     # (Optional) Override value for this wizard

[Template 2]
file=WeeklySalary.ini
Name=Weekly Income
Description=Your primary income source
Value=1500
```

### Sections

**[Settings]** - Required:
- `Name` - Display name for WoW
- `Description` - Explanation of what this WoW creates
- `OverrideValue` - (Optional) Global default value for all sub-wizards

**[Template N]** - One or more:
- `file` - (Required) Item Wizard INI filename (e.g., `HomeLoan.ini`)
- `Name` - (Optional) Override wizard's display name
- `Description` - (Optional) Override wizard's description
- `Value` - (Optional) Override default value for this wizard's properties

---

## Value Override Priority

When a wizard property has multiple default values, priority is:

1. **Template-specific `Value`** (highest priority)
2. **Global `OverrideValue`** from Settings
3. **Original wizard `Default`** (lowest priority)

**Example:**
```ini
[Settings]
OverrideValue=100000      # Global default

[Template 1]
file=HomeLoan.ini
Value=400000              # This overrides global (400k, not 100k)

[Template 2]
file=WeeklySalary.ini
# No Value specified, uses global OverrideValue (100k)
```

**Note:** Type conversion is automatic - the wizard knows its property types, WoW just provides string values.

---

## Behavior

1. User selects WoW from `Wizard of Wizards` menu
2. WoW loads INI file from `Data/wizardsOfWizards/`
3. Validates all referenced Item Wizard files exist in `Data/wizards/`
4. For each `[Template N]` section (in order):
   - Load Item Wizard configuration
   - Apply name/description/value overrides
   - Launch wizard dialog
   - User completes wizard (or cancels)
   - If OK: FinancialItem added to config
   - If Cancel: Stop WoW, previous items remain in config
5. All items created in sequence (step-by-step)
6. Success message shows count of items created

---

## Implementation Details

**Code Location:**
- `UI\Form1.TemplatesWizards.cs` - WoW menu and orchestration (`LaunchWizardOfWizards()`)
- `Services\WizardLoaderService.cs` - Loads WoW files (`LoadAllWizardsOfWizards()`)
- `Services\IniFileParser.cs` - Parses WoW INI format (`ParseWoWFile()`)
- `Models\WizardModels.cs` - Data models (WizardOfWizardsConfiguration, WizardTemplate, etc.)

**Validation:**
- All WoWs validated on load
- Referenced wizard files checked for existence
- Invalid WoWs logged and skipped (appear in Debug Log)

---

## Error Handling

**Missing wizard files:**
- Validates all referenced wizards exist during WoW load
- Skips WoW if any wizard missing
- Error logged to DebugLogger
- WoW doesn't appear in menu

**Missing wizard files at runtime:**
- Stops WoW execution
- Shows error message with count of items created
- Previous items remain in config

**User cancellation:**
- User can cancel at any wizard step
- Shows info message: "N of M items were created"
- Previous items remain in config
- User can manually delete unwanted items if needed

---

## Testing

```bash
# Manual GUI testing
# 1. Falculator.exe (launch GUI)
# 2. File > New
# 3. Wizard of Wizards > First Home Buyer Package
# 4. Complete each sub-wizard
# 5. Verify all items created in Items menu

# Test cancellation
# 1. Launch same WoW
# 2. Complete first wizard
# 3. Cancel second wizard
# 4. Verify only first item was created
```

See `CLAUDE.md` section **"Quick Testing & Verification"** for build/test workflow.

---

## File Structure

**Directory layout:**
```
Data/
+-- wizards/
|   +-- HomeLoan.ini            # Item Wizard (referenced by WoW)
|   +-- WeeklySalary.ini        # Item Wizard (referenced by WoW)
+-- wizardsOfWizards/
|   +-- FirstHomeBuyer.ini      # WoW file (orchestrates multiple wizards)
+-- templates/
    +-- Home Loan.json          # JSON template (used by HomeLoan.ini)
    +-- Weekly Salary.json      # JSON template (used by WeeklySalary.ini)
```

**File naming:**
- **WoW files:** `<Name>.ini` (e.g., `FirstHomeBuyer.ini`)
- No special suffix needed - directory separates them from Item Wizards

---

## Creating New WoW Files

1. Identify which Item Wizards you want to chain
2. Create new `.ini` file in `Data/wizardsOfWizards/`
3. Add `[Settings]` section with Name, Description
4. Add `[Template N]` sections for each wizard
5. Optionally override names, descriptions, values
6. Restart Falculator to load new WoW

**Example:**
```ini
[Settings]
Name=Retirement Setup
Description=Creates pension, investments, and living expenses

[Template 1]
file=Pension.ini
Value=2000

[Template 2]
file=InvestmentAccount.ini
Value=500000

[Template 3]
file=LivingExpenses.ini
Value=3000
```

---

## Design Notes

**Why separate Item Wizards and WoW?**
- Item Wizards are reusable components
- WoW composes existing wizards without duplication
- Easy to maintain (change wizard, all WoWs update automatically)
- Clear separation of concerns

**Why step-by-step execution instead of batch?**
- User can review each item before proceeding
- Easier to debug issues
- User can cancel mid-WoW if they realize they need different values
- Follows wizard UX conventions (sequential, guided)

**Why separate directory for WoW files?**
- Prevents confusion in file system
- No special file extension needed (`.ini` vs `.wow.ini`)
- Clear organizational structure
- Easy to find and manage WoW files

**Why allow partial completion on cancel?**
- Partial progress better than losing all work
- User can manually delete unwanted items
- More flexible workflow




