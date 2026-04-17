## SCENARIO 14: REAL-WORLD EVIDENCE COHORT GENERATION

**Assumptions**:
- GLP-1 agonist: `drug_class` IN (glp1_receptor_agonist).
- T2DM diagnosis: `icd_10_code LIKE 'E11%'`.
- Insulin switch: `drug_class = 'insulin'` within 180 days of GLP-1 initiation.
- Initiation = first GLP-1 fill in 2023.

**Query**:
```sql
WITH glp1_initiation AS (
    SELECT 
        c.member_id,
        MIN(c.fill_date) AS init_date,
        d.drug_class
    FROM FACT_PHARMACY_CLAIMS c
    JOIN DIM_DRUG_MASTER d ON c.ndc = d.ndc
    WHERE c.fill_date BETWEEN '2023-01-01' AND '2023-12-31'
      AND c.claim_status = 'adjudicated'
      AND d.drug_class = 'glp1_receptor_agonist'
    GROUP BY 1, 3
),
t2dm_diagnosis AS (
    SELECT 
        m.member_id,
        m.service_date
    FROM FACT_MEDICAL_CLAIMS m
    WHERE m.icd_10_code LIKE 'E11%'
),
insulin_switch AS (
    SELECT 
        c.member_id,
        c.fill_date AS insulin_date
    FROM FACT_PHARMACY_CLAIMS c
    JOIN DIM_DRUG_MASTER d ON c.ndc = d.ndc
    WHERE d.drug_class = 'insulin'
      AND c.claim_status = 'adjudicated'
)
SELECT 
    g.member_id,
    g.init_date,
    COUNT(DISTINCT t.service_date) AS t2dm_diagnosis_count,
    MIN(i.insulin_date) AS earliest_insulin_date
FROM glp1_initiation g
LEFT JOIN t2dm_diagnosis t ON g.member_id = t.member_id
  AND t.service_date BETWEEN DATEADD(DAY, -365, g.init_date) AND g.init_date
LEFT JOIN insulin_switch i ON g.member_id = i.member_id
  AND i.insulin_date BETWEEN g.init_date AND DATEADD(DAY, 180, g.init_date)
WHERE t.member_id IS NOT NULL
  AND i.member_id IS NULL
GROUP BY 1, 2
ORDER BY g.init_date;
```

**Explanation**:
- Initiation = first GLP-1 fill in 2023. `MIN(fill_date)` captures it.
- T2DM diagnosis window: 365 days prior to initiation. Ensures prior history.
- Insulin switch excluded if found within 180 days. `LEFT JOIN` + `WHERE i.member_id IS NULL` enforces exclusion.
- Validation: Cross-check cohort against clinical trial criteria. Verify diagnosis codes align with T2DM definitions.
- Edge case: Overlapping fills. Handled by date boundaries.

**Cost/Performance Note**:
- Multiple joins bounded by member and date. Snowflake prunes efficiently.
- If `FACT_MEDICAL_CLAIMS` large, cluster on `icd_10_code, service_date`.
- Tag: `ALTER SESSION SET QUERY_TAG = 'capstone_scenario14_rwe';`

**Production Note**:
- RWE cohorts used for outcomes research. Ensure IRB compliance and data de-identification.
