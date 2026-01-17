# Config Model

Root configuration container for simulation settings, user information, taxation, and financial items.

---

## Overview

Config is the top-level container for a Falculator simulation. It defines:
- **User information**: Birth date for age-based triggers
- **Simulation settings**: Duration, step interval, start date, interactive mode
- **Taxation**: Tax mode and rates
- **Financial items**: List of all FinancialItems in the simulation
- **Main savings account**: Designation for cash flow sweep hub
- **Logging**: Debug level and overdraft handling
- **Backups**: Rolling backup configuration

**File format:** JSON (typically saved as .json files)

**Validation:** Sanitize() method called on load and before simulation start

---

## Quick Start

### Minimal Configuration

```json
{
  "Version": 1,
  "BirthDate": "1990-01-01",
  "SimulationName": "My Retirement Plan",
  "YearsToSim": 30,
  "StepIncrement": "Monthly",
  "StartDateIsToday": true,
  "TaxMode": "NoTax",
  "FinancialItemsList": [
    {
      "Name": "Main Savings Account",
      "Type": "Savings",
      "IsMainSavingsAccount": true,
      "IsLiquidAsset": true,
      "StartEnabled": true,
      "Value": 10000
    }
  ]
}
```

---

## Property Categories

### 1. User Information

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| BirthDate | DateOnly | now - 18 years | User's date of birth for age calculations |
| UserName | string? | "Default" | Optional user name (not required) |

### 2. Simulation Settings

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| Version | uint | 1 | Config file format version (read-only) |
| SimulationName | string? | "Default Simulation Name" | Human-readable simulation name |
| Description | string? | "Default Config" | Optional simulation description |
| YearsToSim | int | 30 | Number of years to simulate |
| StepIncrement | Frequency | Weekly | Simulation step interval (Daily, Weekly, Monthly, Annual) |
| StartDateIsToday | bool | true | If true, StartDate always set to today on load |
| StartDate | DateOnly | now | Simulation start date (ignored if StartDateIsToday=true) |
| InteractiveMode | bool | false | Enable manual fund transfers at each step (GUI only) |

### 3. Taxation

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| TaxMode | TaxMode | NoTax | Taxation mode (NoTax, FlatTax, AustralianComprehensive) |
| TaxPercentage | decimal | 30 | Flat tax rate percentage (0-100) |
| EndOfFinancialYear | DateOnly | June 30 | End of financial year for annual tax calculations |

### 4. Financial Items

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| FinancialItemsList | List<FinancialItem>? | null | List of all financial items in simulation |
| MainSavingsAccountId | uint? | null | Index of main savings account (read-only, set by Sanitize) |

### 5. Logging & Debugging

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| LogLevel | LogLevel | Info | Debug log level (Debug, Info, Warn, Error) |
| FailOnOverDraw | bool | true | Halt simulation if any item balance goes negative |

### 6. Backups

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| RollingBackup | bool | true | Enable automatic rolling backups |
| BackupCount | uint | 10 | Number of backup files to retain |

### 7. Save Statistics

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| DateSaved | DateTime | now | Date and time config last modified |
| SaveCount | uint | 0 | Number of times config has been saved |

---

## TaxMode Enum

| Value | Numeric | Description |
|-------|---------|-------------|
| NoTax | 1 | No tax applied to income or interest (default) |
| FlatTax | 2 | Flat tax rate applied to all income and interest |
| AustralianComprehensive | 3 | Australian progressive tax brackets (not yet implemented) |

**Note:** Currently only NoTax and FlatTax are implemented.

---

## Main Savings Account

**Purpose:** Central hub for all cash flow settlement. CashIn/CashOut from all items flows through this account.

**Requirements:**
- Exactly one FinancialItem must have IsMainSavingsAccount=true
- Type must be Savings
- EvalOrder must be 0 (evaluated first)
- StartEnabled must be true
- DisabledByUser must be false
- IsLiquidAsset must be true
- EndDate must be far in future (>= 95 years)

**Validation:** SetMainSavingsAccountId() finds and validates the main savings account

**Auto-creation:** If no main savings account exists, Sanitize() creates one automatically using FinancialItem.CreateDefaultSavingsAccount()

---

## Sanitization Rules

Config.Sanitize() is called automatically on load and before simulation start.

### Version Enforcement

- Version is always set to 1 (current version)

### ID Tracking Reset

- IdGenerator.Reset() called to ensure unique IDs across all objects

### End of Financial Year

- If EndOfFinancialYear == default: sets to June 30 of current year

### Start Date Handling

- If StartDateIsToday=true: sets StartDate to DateTime.Now

**Warning:** If StartDate < BirthDate, logs warning about negative age

### Years to Simulate

- If YearsToSim <= 0: sets to 1 (minimum)

### Financial Items List

- If null or empty: creates default main savings account
- Calls Sanitize() on all FinancialItems
- Calls SanitizeTargetReference() on all EventItems (validates cross-references)

### Main Savings Account

- Calls SetMainSavingsAccountId() to find and validate
- If not found: creates default main savings account at index 0
- If multiple found: logs ERROR and sets MainSavingsAccountId=null

### EvalOrder Sanitization

- Ensures main savings account has EvalOrder=0
- Validates and corrects EvalOrder conflicts

### Tag Registry

- Builds TagRegistry from all item Tags
- Validates tag references in TagConditionals
- Disables TagConditionals with invalid tags

---

## Validation Workflow

**On Load:**
1. Deserialize JSON to Config object
2. Call Config.Sanitize()
3. Validation complete - ready for simulation

**On Save:**
1. Update DateSaved to DateTime.Now
2. Increment SaveCount
3. Call Config.Sanitize() (optional but recommended)
4. Serialize to JSON
5. Save to file
6. Create backup (if RollingBackup=true)

**Before Simulation:**
1. Call Config.Sanitize() (ensures fresh state)
2. Validate MainSavingsAccountId is set
3. Validate all items have valid EvalOrder
4. Build TagRegistry
5. Start simulation

---

## Common Patterns

### Pattern 1: Simple 30-Year Retirement Plan

```json
{
  "BirthDate": "1990-01-01",
  "SimulationName": "Retirement Plan",
  "YearsToSim": 30,
  "StepIncrement": "Monthly",
  "TaxMode": "NoTax",
  "FinancialItemsList": [
    {
      "Name": "Main Savings Account",
      "Type": "Savings",
      "IsMainSavingsAccount": true,
      "Value": 10000
    },
    {
      "Name": "Salary",
      "Type": "Income",
      "StartEnabled": true,
      "CashIn": {
        "Enabled": true,
        "Amount": 5000,
        "Schedule": { "Frequency": "Monthly" }
      }
    }
  ]
}
```

### Pattern 2: Weekly Simulation with Flat Tax

```json
{
  "BirthDate": "1985-06-15",
  "SimulationName": "Short-Term Cash Flow",
  "YearsToSim": 2,
  "StepIncrement": "Weekly",
  "TaxMode": "FlatTax",
  "TaxPercentage": 25,
  "StartDateIsToday": true
}
```

### Pattern 3: Interactive Mode for Manual Adjustments

```json
{
  "InteractiveMode": true,
  "StepIncrement": "Monthly",
  "YearsToSim": 5,
  "StartDateIsToday": true
}
```

**Note:** InteractiveMode only works in GUI, not CLI.

### Pattern 4: Daily High-Frequency Simulation

```json
{
  "StepIncrement": "Daily",
  "YearsToSim": 1,
  "StartDateIsToday": true,
  "FailOnOverDraw": false
}
```

**Note:** Daily simulations with YearsToSim > 5 will generate large datasets.

---

## CLI Override Flags

Several config properties can be overridden via command-line flags:

| CLI Flag | Overrides | Example |
|----------|-----------|---------|
| --years-override N | YearsToSim | `--years-override 1` |
| --loglevel LEVEL | LogLevel | `--loglevel DEBUG` |
| --savepath PATH | Output directory | `--savepath "testoutput"` |
| --sanitize-config | Forces sanitization | `--sanitize-config --config "config.json"` |

**See:** CLAUDE.md for full CLI command reference.

---

## Debugging

### SetMainSavingsAccountId() Validation

Logs at ERROR level if:
- Multiple items have IsMainSavingsAccount=true
- No main savings account found (before auto-creation)

### Sanitize() Warnings

Logs at WARN level for:
- StartDate < BirthDate (negative age)
- YearsToSim <= 0 (corrected to 1)
- EndOfFinancialYear == default (set to June 30)
- No items present (creates default savings account)
- No main savings account (creates one)

### Debug Logging

Set LogLevel=Debug for detailed sanitization output:

```bash
Falculator.exe --config "config.json" --loglevel DEBUG
```

---

## Related Models

- **FinancialItem** - See FinancialItem.md for financial item structure
- **TaxMode** - Enum defining taxation modes (NoTax, FlatTax, AustralianComprehensive)
- **Frequency** - Enum defining step intervals (Daily, Weekly, Monthly, Annual)
- **IdGenerator** - ID generation and uniqueness validation
- **TagRegistry** - Tag validation and cross-reference checking

---

## JSON Schema Reference

Full schema: `Data/Schema/Config.schema.json`

**Required properties:**
- Version (defaults to 1)
- BirthDate (defaults to now - 18 years)
- YearsToSim (defaults to 30)
- StepIncrement (defaults to Weekly)
- FinancialItemsList (created with default savings if missing)

**Optional properties:**
- UserName, SimulationName, Description
- TaxMode (defaults to NoTax)
- TaxPercentage (defaults to 30)
- LogLevel (defaults to Info)
- FailOnOverDraw (defaults to true)
- RollingBackup (defaults to true)
- BackupCount (defaults to 10)

---

## Example Configurations

Example configs demonstrating various scenarios:

- **Data/examples/HomeLoan.json** - Comprehensive scenario (salary, loan, property, events)
- **Data/examples/001.Test.json** - Compound interest testing
- **Data/examples/InteractiveMode.Test.json** - Interactive mode configuration
- **Data/examples/Shares.Test.json** - Shares ItemType demonstration
- **Data/examples/006.Test.json** - Complex trigger scenarios

---

## Implementation Details

**Source file:** `Models/Config.cs`

**Key methods:**
- `SetMainSavingsAccountId()` - Lines 157-184: Finds and validates main savings account
- `Sanitize()` - Lines 190-350+: Comprehensive validation and correction
- Proxy properties (BirthDateProxy, StartDateProxy, EndOfFinancialYearProxy): GUI integration for DateTimePicker

**Processing:**
- Loaded from JSON via System.Text.Json
- Sanitize() called after deserialization
- Passed to SimFrame for simulation execution

**Dependencies:**
- `FinancialItem` - FinancialItemsList property
- `TaxMode` enum (Config.cs) - Taxation configuration
- `Frequency` enum (Models/GlobalEnums.cs) - Step interval
- `IdGenerator` - ID tracking and validation
- `DebugLogger` - Logging infrastructure
- `TagRegistry` - Tag validation

**Used by:**
- `SimFrame` - Simulation execution engine
- `Form1` - GUI configuration editor
- CLI - Command-line simulation runner

---
