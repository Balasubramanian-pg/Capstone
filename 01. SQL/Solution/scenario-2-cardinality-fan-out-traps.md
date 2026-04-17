## SCENARIO 2: CARDINALITY & FAN-OUT TRAPS

**Assumptions**:
- Brand vs generic derived from `DIM_DRUG_MASTER.is_controlled` and `generic_name` presence, but simplified here to `drug_class` or `formulation` flag. Assume `is_brand` boolean exists or derived.
- Prescriber specialty from `DIM_PROVIDERS`.
- Exclude mail order (`practice_type != 'mail_order'`).
- Q3 2023 = 2023-07-01 to 2023-09-30.
- Use `COUNT(DISTINCT claim_id)` to protect against duplicates.

**Query**:
```sql
WITH q3_claims AS (
    SELECT 
        c.claim_id,
        c.prescriber_npi,
        c.days_supply,
        c.ndc,
        CASE WHEN d.generic_name IS NULL OR d.generic_name = d.brand_name THEN 'brand' ELSE 'generic' END AS brand_generic
    FROM FACT_PHARMACY_CLAIMS c
    JOIN DIM_DRUG_MASTER d ON c.ndc = d.ndc
    WHERE c.claim_status = 'adjudicated'
      AND c.fill_date BETWEEN '2023-07-01' AND '2023-09-30'
),
provider_joined AS (
    SELECT 
        qc.claim_id,
        qc.brand_generic,
        qc.days_supply,
        pr.specialty
    FROM q3_claims qc
    LEFT JOIN DIM_PROVIDERS pr ON qc.prescriber_npi = pr.npi
    WHERE pr.practice_type != 'mail_order'
),
specialty_agg AS (
    SELECT 
        COALESCE(specialty, 'unknown') AS prescriber_specialty,
        brand_generic,
        COUNT(DISTINCT claim_id) AS claim_count,
        AVG(days_supply) AS avg_days_supply
    FROM provider_joined
    GROUP BY 1, 2
)
SELECT * FROM specialty_agg ORDER BY avg_days_supply DESC;
```

**Explanation**:
- CTE isolates Q3 claims early. Reduces join size.
- `LEFT JOIN` preserves claims with missing prescriber data. `COALESCE` handles NULL specialty.
- `COUNT(DISTINCT claim_id)` prevents duplicate inflation.
- `AVG(days_supply)` calculated at aggregation level. Snowflake optimizes hash aggregation.
- Validation: Spot-check 5 claims. Verify brand/generic classification matches master data. Confirm specialty alignment.

**Cost/Performance Note**:
- Join on `ndc` and `prescriber_npi` is selective. Snowflake uses hash join.
- If `DIM_PROVIDERS` is large, consider broadcasting or materializing small dimension.
- Tag: `ALTER SESSION SET QUERY_TAG = 'capstone_scenario2_cardinality';`

**Production Note**:
- Mail order exclusion applied after join to avoid filtering out claims before specialty assignment.
