## SCENARIO 1: BASELINE METRIC VALIDATION

**Assumptions**:
- `gross_amount` = total claim amount before member/payer split.
- `payer_paid_amount` + `copay_amount` + `coinsurance_amount` = `gross_amount`.
- Reversed claims excluded via `claim_status != 'reversed'`.
- Month bucketing uses `DATE_TRUNC('MONTH', fill_date)`.
- Missing copay/coinsurance treated as 0.

**Query**:
```sql
WITH monthly_claims AS (
    SELECT 
        DATE_TRUNC('MONTH', fill_date) AS fill_month,
        p.sponsor_type,
        SUM(gross_amount) AS gross_spend,
        SUM(payer_paid_amount) AS payer_paid,
        SUM(COALESCE(copay_amount, 0) + COALESCE(coinsurance_amount, 0)) AS member_oop,
        COUNT(claim_id) AS claim_count
    FROM FACT_PHARMACY_CLAIMS c
    JOIN DIM_PLANS p ON c.plan_id = p.plan_id
    WHERE c.claim_status != 'reversed'
      AND c.fill_date >= DATEADD(MONTH, -12, CURRENT_DATE)
    GROUP BY 1, 2
)
SELECT 
    fill_month,
    sponsor_type,
    gross_spend,
    payer_paid,
    member_oop,
    claim_count,
    ROUND(payer_paid / NULLIF(claim_count, 0), 2) AS avg_payer_paid
FROM monthly_claims
ORDER BY fill_month DESC, sponsor_type;
```

**Explanation**:
- Aggregation happens at fact level. Join to `DIM_PLANS` is safe because `plan_id` is stable per claim.
- `COALESCE` prevents NULL propagation in member OOP calculation.
- `NULLIF` prevents division by zero.
- Date filter uses `DATEADD` for dynamic range. No hard-coded months.
- Performance: Single scan with predicate pushdown on `fill_date` and `claim_status`. Snowflake prunes micro-partitions efficiently.
- Validation: Cross-check `SUM(gross_spend)` against PBM accounting export for sample month. Row count should match distinct `(fill_month, sponsor_type)`.

**Cost/Performance Note**:
- If `FACT_PHARMACY_CLAIMS` lacks clustering on `fill_date`, Snowflake still prunes via metadata. Expect < 3 seconds on XS warehouse.
- Tag query: `ALTER SESSION SET QUERY_TAG = 'capstone_scenario1_baseline';`

**Production Note**:
- Avoid joining `DIM_MEMBERS` at this stage. Membership status changes introduce fan-out if not aligned to claim date.
