## SCENARIO 3: DYNAMIC ARRAY FILTERING & MULTI-CRITERIA AGGREGATION

**Assumptions**:
- `Claims` table contains `DrugClass`, `PlanType`, `Status`, `Payer_Paid`, `Fill_Date`.
- Q3 2023 = 2023-07-01 to 2023-09-30.
- Exclude `Status = "Reversed"`.

**Formula**:
```excel
=LET(
  q3, FILTER(Claims, (Claims[Fill_Date]>=DATE(2023,7,1))*(Claims[Fill_Date]<DATE(2023,10,1))),
  ma, FILTER(q3, q3[PlanType]="Medicare Advantage"),
  rev, FILTER(ma, ma[Status]<>"Reversed"),
  spend, SUM(rev[Payer_Paid]),
  count, COUNTA(rev[Claim_ID]),
  {"Q3 MA Specialty Spend", spend, "Claim Count", count}
)
```

**Explanation**:
- Boolean arrays multiply to `AND` logic. `FILTER` returns structured array. `LET` caches intermediate steps.
- Output spills automatically. No helper columns needed.
- Validation: Cross-check spend against pivot table. Verify count matches filtered rows.

**Excel Performance Note**:
- `FILTER` recalculates on source change. Keep arrays under 500K rows for smooth performance. Use `.xlsb` if larger.
