# About

This document describes the order and logic of the simulation engine.

# Design Principles

## Non-Turing Complete by Design

Deliberately non-turing complete: no loops are supported.

Each simulation step is guaranteed to terminate; infinite loops are not possible.

Each Financial Item has an **EvalOrder** (Evaluation Order) property. Lower numbers are evaluated first.

Duplicate EvalOrder values are not permitted and are automatically corrected during sanitization.

## Config Sanitization

Strict config sanitization is applied on load, save, and before each simulation run.

All corrections are logged via `DebugLogger`.

If sanitization performs any corrections, the user is presented with a dialog showing the corrections. They have the option to accept corrections OR revert and self-correct.

See: `Config.Sanitize()` method in `Models/Config.cs`

## Atomic Financial Items (Separation of Concerns)

A single real-world financial object may be represented by 1 or more **Financial Items** (`FinancialItem` class in `Models/FinancialItem.cs`), each with a specific concern.

This design prevents the need for complex single-item configurations and allows distinct mapping of Debt, Assets, and Liquid Funds.

### Example: Laptop Purchase

**Financial Item 1:** Laptop Purchase (Expenditure). When trigger conditions are met, this item disables itself and enables Item 2.  
**Financial Item 2:** Laptop (Asset). Starts disabled. Once enabled, depreciation is applied to its value at the user-defined rate.

Item 1 waits for trigger conditions (e.g., minimum savings amount reached), creates the expenditure, enables Item 2, and disables itself.

### Example: Investment Rental Property

**Financial Item 1:** Loan (Liability). Compound interest applied to loan amount (e.g., 7.1%). Has loan repayments set as Expenditure or via `EventItem`.  
**Financial Item 2:** Property (Asset). Compound interest applied to property value (e.g., 4.5% appreciation).  
**Financial Item 3:** Rental Income (Revenue). Generates rental income (e.g., $700/week).

Item 1, if enabled, enables Items 2 & 3.  
Item 1 self-disables when loan amount reaches 0 (loan paid off).

# Simulation Execution Flow

The simulation runs from **Step 0** (initial state) and increments by **Step Increment** until **Years to Sim** has elapsed.

## Step 0 (Initial Frame)

Provides base values for all Financial Items. No triggers are evaluated because no time has passed.

The initial `SimFrame` is constructed using the `SimFrame(Config config)` constructor, which copies the state from `Config.FinancialItemsList` into `SimFrame.FinancialItemsState`.

See: `SimFrame.cs` constructor and `Form1.cs` simulation initialization.

## Incremental Steps

For each **Step Increment**, a new **SimFrame** is created.

For example, if **Step Increment** is **Weekly** and simulation duration is 1 year, there will be 52 SimFrames.

### Step Increment Modes

- **Daily:** The simulation steps through every day.
- **Weekly, Fortnightly, Monthly, Annual:** The simulation advances N days, then calculates how many times since the previous step payments (**Revenue**, **Expenditure**, **Interest**) should be applied.

For example, if a Financial Item has daily revenue of $10 and your **Step Increment** is **Weekly**, 7 x $10 = $70 is calculated for that simulation step.

**Important:** Triggers are only checked **once per step**, regardless of how many days are compressed into that step.

## Simulation Frame Calculation

See: `SimFrame.CalculateFrame()` and `SimFrame.CalculateItem()` methods in `Models/SimFrame.cs`

For each **Financial Item** in order of **EvalOrder** (lowest first):

1. **User Disable Check:** If **DisabledByUser** == true, skip this item entirely.

2. **Self-Trigger Evaluation:** Check **SelfTrigger** conditions (see `TriggerConditions` class in `Models/TriggerConditions.cs`). 
   - The `EvaluateTrigger()` method evaluates ALL configured conditions (age min/max, start/end date, liquid assets, main savings balance, tag rules, income tags).
   - Based on `RequireAll` setting, either ALL conditions must pass (AND logic) or ANY condition must pass (OR logic).
   - The result is a **single boolean** that sets **EnabledBySim** to true or false.
   - There are no separate "activation" and "deactivation" conditions; the same conditions are evaluated each step to determine the current enabled state.

3. **Enabled Check:** If **EnabledBySim** == false, skip remaining calculations for this item.

4. **Financial Calculations (if Enabled):**
   - Process **Interest** (applied to current item value before revenue/expenditure)
   - Apply tax to interest (for Savings items only), immediately deducted and added to `SimFrame.TaxPaid`
   - Add net interest (gross interest - tax) to item value
   - Process **Expenditure** (calculated amount, NOT deducted from item value -- **deducted from cashflow instead**)
   - Process **Revenue** (calculated amount, added to cashflow, NOT directly to item value)
   - Apply tax to revenue (all item types), immediately deducted and added to `SimFrame.TaxPaid`
   - Calculate net **CashFlow** = (Net Revenue after tax) - Expenditure
   - **Note:** Tax is applied immediately per-item, not accumulated and applied at frame end

5. **EventItem Processing:** For each **EventItem** in the item's **Events** list (see `EventItem` class in `Models/EventItem.cs`):
   - If **Enabled** == false, skip.
   - Evaluate **EventItem.Triggers** conditions (if any defined).
   - If conditions are met, execute payment transfer from source item to target item.
   - Handle special cases:
     - Loan overpayment prevention (caps payment to remaining balance)
     - State changes (Enable/Disable/Toggle target item via `SetStateOnTrigger`)
   - Record event if payment is conditional/triggered (non-regular payments).

6. **Cash Flow Sweep:** 
   - **Current Implementation:** ALL cashflow from all Financial Items is summed and swept into the **Main Savings Account** (identified by `Config.MainSavingsAccountId`).
   - Code: `var cashFlow = FinancialItemsState.Sum(x => x.CashFlow);`
   - **Intended Future Behavior:** Only non-Savings items should have their cashflow swept. Savings accounts should retain their own cashflow.
   - **Note:** This is a known discrepancy between current and intended behavior.

7. **Overdraw Detection:** If **Config.FailOnOverDraw** == true and **Main Savings Account** balance < 0, end simulation with error.

## Simulation Summary

After all frames are processed:

- **Generate Sim Data CSV** (via export functionality)
- **Render Sim Data Grid** (DataGridView in `Form1.cs`)
- **Plot Graph** (via `SimulationGraphService` in `Services/SimulationGraphService.cs`)
- **Display Events** (activation, deactivation, conditional payments) on graph and in event log

# Key Classes Reference

- **`Config`** (`Models/Config.cs`): Top-level simulation configuration including user settings, tax mode, and list of Financial Items.
- **`FinancialItem`** (`Models/FinancialItem.cs`): Represents a single financial entity (income, asset, liability, savings account, etc.).
- **`SimFrame`** (`Models/SimFrame.cs`): Represents the state of the simulation at a single point in time. Contains snapshot of all Financial Items.
- **`FinancialItemSimple`**: Lightweight runtime state for a Financial Item during simulation (stored in `SimFrame.FinancialItemsState`).
- **`AmountFreq`** (`Models/AmountFreq.cs`): Defines recurring financial amounts (revenue, expenditure, interest) with frequency and schedule rules.
- **`TriggerConditions`** (`Models/TriggerConditions.cs`): Defines activation/deactivation conditions based on age, date, liquid assets, savings balance, or tags.
- **`EventItem`** (`Models/EventItem.cs`): Defines conditional payments from one Financial Item to another, with optional trigger conditions.
- **`SimEvent`** (in `SimFrame.cs`): Records discrete events during simulation (activations, deactivations, conditional payments) for reporting and visualization.

# Terminology

- **Financial Item (FI):** A single entry in `Config.FinancialItemsList` representing income, asset, liability, savings, loan, or expense.
- **Step Increment:** The time interval between simulation frames (Daily, Weekly, Fortnightly, Monthly, Annual).
- **SimFrame:** A snapshot of all Financial Items at a specific point in the simulation timeline.
- **EvalOrder:** The order in which Financial Items are processed within each SimFrame (lower numbers first).
- **Main Savings Account:** The primary liquid account where all cash flow is swept. Identified by `IsMainSavingsAccount` property and `Config.MainSavingsAccountId` index.
- **Trigger:** A condition or set of conditions that controls when a Financial Item or EventItem becomes active.
- **Event:** A discrete occurrence during simulation (activation, deactivation, or conditional payment) recorded in `SimFrame.Events`.

# Notes

## TriggerCount = 1 Behavior

Deactivation showing next frame artifact.

The next SimFrame after Activation you will see the item Deactivated.

This is because the the trigger can no longer be met in the next frame due to having a limit of 1, so the item deactivates.

It will not activate and deactivate on the same frame; the activation happens first, then on the next frame the deactivation is evaluated.

# Advanced Functionality not scoped for MVP

these features are not considered part of the initial MVP and are planned for future version eg 2.0+

- Loan redraw functionality

# EventItem constraints

CashIn is only supported for ItemType.Savings
