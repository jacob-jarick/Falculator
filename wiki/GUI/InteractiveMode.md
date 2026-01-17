# Interactive Mode (Drug Wars)

**Inspiration:** Drug Wars (1980s) - pause sim execution, manually transfer funds between accounts

Uses hub & spoke terminology for dope wars/ fiddle UI
 
---

## Quick Testing

```bash
# Test interactive mode with 2-year simulation
Falculator.exe --config "Data\examples\InteractiveMode.Test.json" --run --loglevel DEBUG

# When sim pauses (Interactive Mode tab appears):
# - Click "In"/"Out" buttons to transfer funds
# - Click "Next" to advance simulation
# - Click "End Interactive Mode" to run to completion
# - Click "Stop" to terminate simulation immediately
```

See `CLAUDE.md` section **"Quick Testing & Verification"** for full testing workflow.

---

## Notes

currently hard to automate tesing due to needing GUI interaction, low priority for now until GUI is stable.

---

## Design Decisions

**Q: What if user tries to transfer more than available?**  
A: % buttons calculate from actual value (impossible to overdraw). $100 buttons disabled if insufficient funds.

**Q: What if user manually switches tabs during interactive mode?**  
A: Allowed. Sim auto-switches back to Interactive Mode tab when paused.

**Q: Can user change Enabled state via checkbox?**  
A: No. Checkbox is read-only, displays `EnabledBySim` state only.

**Q: Why disable Main Savings Account transfer buttons?**  
A: Main Savings is the hub - all transfers go through it. No point transferring to/from itself.

**Q: How is loan repayment handled?**  
A: Loan values are negative. "Out" buttons are disabled (cannot withdraw from loan). "In" buttons pay down debt (reduce negative value toward zero). Overpayment is blocked by validation.

---

## Implementation Notes

**Item Row Ordering:**
1. Main Savings Account (always first)
2. Other eligible items (Savings/Loan types, not `DisabledByUser`)
3. Sorted alphabetically by Name
4. Reversed for Dock=Top stacking (last added appears at top)

**Eligible Items:**
- Must be `ItemType.Savings` or `ItemType.Loan`
- Must not be `DisabledByUser`
- Automatically filtered by `UpdateInteractiveModeDisplay()`

**Transfer Amount Calculation:**
```csharp
// Percentage transfers (rounded down)
decimal sourceValue = isInbound ? mainSavingsState.Value : itemState.Value;
decimal amount = Math.Floor(sourceValue * percentage);

// Fixed amount transfers
decimal amount = fixedAmount;  // $100
```

**Loan Transfer Logic:**
```csharp
// For loans/liabilities (negative values)
if (to.Type == ItemType.Loan || to.Type == ItemType.Liability)
{
    to.Value += amount;  // Adding positive reduces debt (moves toward zero)
}
```

**Event Handling:**
- Each transfer logs via `DebugLogger.Log()` with `LogLevel.Info`
- No formal EventItem created (manual transfers bypass event system)
- Mini log displays in real-time