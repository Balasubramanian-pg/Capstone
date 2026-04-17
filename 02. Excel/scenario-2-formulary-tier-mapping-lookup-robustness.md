## SCENARIO 2: FORMULARY TIER MAPPING & LOOKUP ROBUSTNESS

**Assumptions**:
- `Formulary` sheet contains `NDC`, `Tier`, `Eff_Start`, `Eff_End`.
- Claims table has normalized 11-digit NDC.
- Supersession mapping in `NDC_Map` sheet with `Old_NDC`, `New_NDC`.

**Formula**:
```excel
=LET(
  ndc, [@NDC],
  mapped, IFERROR(XLOOKUP(ndc, NDC_Map[Old_NDC], NDC_Map[New_NDC], ndc, 0), ndc),
  tier, XLOOKUP(mapped, Formulary[NDC], Formulary[Tier], "Not on Formulary", 0),
  tier
)
```

**Explanation**:
- `LET` improves readability and prevents repeated NDC lookups.
- `XLOOKUP` handles supersession fallback. Second `XLOOKUP` maps to tier. `0` ensures exact match.
- Validation: Test with known superseded NDCs. Verify "Not on Formulary" appears for unmapped.

**Excel Performance Note**:
- Convert lookup tables to Excel Tables (`Ctrl+T`). Enables dynamic range expansion. Avoid `VLOOKUP` for left-column mismatches.
