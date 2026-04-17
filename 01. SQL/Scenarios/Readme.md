# 4. CAPSTONE SCENARIOS (15, INCREMENTAL DIFFICULTY)

Each scenario builds on the previous. Success requires correctness, performance, maintainability, compliance awareness, and business alignment.

---

## SCENARIO 1: BASELINE METRIC VALIDATION

**Objective**  
Calculate monthly gross drug spend, payer-paid amount, and member out-of-pocket by plan sponsor type for the last 12 months.

**Constraints**
- Exclude reversed claims  
- Handle NULL `copay_amount` and `coinsurance_amount`  
- Round to two decimals  

**Success Criteria**
- Accurate aggregation  
- Correct date bucketing  
- Zero duplicate inflation from joins  
- Clear output schema  

**Known Pitfall**
Joining `FACT_PHARMACY_CLAIMS` with `DIM_MEMBERS` before aggregation multiplies rows if membership status changes mid-month.

---

## SCENARIO 2: CARDINALITY & FAN-OUT TRAPS

**Objective**  
Calculate average days supply per prescriber specialty for brand vs generic drugs in Q3 2023.

**Constraints**
- Use `FACT_PHARMACY_CLAIMS`, `DIM_DRUG_MASTER`, `DIM_PROVIDERS`  
- Exclude mail order  
- Include only adjudicated claims  

**Success Criteria**
- Correct specialty grouping  
- Accurate brand/generic classification  
- Proper NULL handling for missing prescriber data  

**Known Pitfall**
Using `COUNT(DISTINCT claim_id)` vs `COUNT(claim_id)` when duplicate adjudications exist. Pre-aggregating before filtering distorts averages.

---

## SCENARIO 3: WINDOW FUNCTIONS & ADHERENCE COHORTS

**Objective**  
Identify members who filled a maintenance medication for at least 6 consecutive months and calculate their Proportion of Days Covered (PDC).

**Constraints**
- Include `drug_class` IN (statins, ACE_inhibitors, ARBs, diabetes_oral)  
- Exclude specialty drugs  
- PDC = days covered / 180 days  

**Success Criteria**
- Correct consecutive month logic  
- Accurate PDC calculation  
- Proper handling of overlapping fills  

**Known Pitfall**
- Misaligned `LAG()` / `LEAD()`  
- Overlapping fills inflate coverage  
- Not capping PDC at 1.0  

---

## SCENARIO 4: SLOWLY CHANGING DIMENSIONS & FORMULARY ALIGNMENT

**Objective**  
Calculate total paid amount per formulary tier for Q2 2023 using the tier active at claim fill date.

**Constraints**
- `DIM_FORMULARY` is Type 2 SCD  
- Use `effective_start_date`, `effective_end_date`  

**Success Criteria**
- Historical accuracy  
- Correct date overlap logic  
- No double-counting  

**Known Pitfall**
Using current tier instead of historical tier leads to incorrect allocations.

---

## SCENARIO 5: SARGABILITY & CLAIM REVERSAL HANDLING

**Objective**  
Find December 2023 claims that were later reversed and calculate net impact per member.

**Constraints**
- Use `FACT_PHARMACY_CLAIMS`  
- Query must run under 4 seconds (XS warehouse)  

**Success Criteria**
- Partition pruning works  
- Correct reversal pairing  
- Accurate net calculation  

**Known Pitfall**
Applying functions like `TO_DATE()` on filter columns breaks pruning.

---

## SCENARIO 6: NDC SUPERSESSION & PRODUCT CONTINUITY

**Objective**  
Map superseded NDCs to current ones and calculate quantity by GPI.

**Constraints**
- Use `DIM_NDC_SUPERSESSION`  
- Handle chains like A → B → C  

**Success Criteria**
- Accurate recursive mapping  
- No double counting  

**Known Pitfall**
Assuming one-to-one mapping instead of chains.

---

## SCENARIO 7: DIR FEE LAG & RETROACTIVE ADJUSTMENTS

**Objective**  
Calculate net pharmacy revenue per month for Q1 2024 including late DIR fees.

**Constraints**
- Match DIR fees using `service_month`, not `received_date`  

**Success Criteria**
- Correct retroactive adjustments  
- Clear gross vs net separation  

**Known Pitfall**
Joining on `received_date` instead of `service_month`.

---

## SCENARIO 8: REBATE WATERFALL & CONTRACT ALIGNMENT

**Objective**  
Calculate rebate amounts per manufacturer for Q4 2023.

**Constraints**
- Handle fixed, percentage, and tiered contracts  

**Success Criteria**
- Correct tier evaluation  
- Proper null handling  

**Known Pitfall**
Misapplying tier logic when thresholds aren’t met.

---

## SCENARIO 9: 340B COMPLIANCE & ELIGIBILITY VALIDATION

**Objective**  
Identify non-compliant 340B fills.

**Constraints**
- Match `fill_date` to eligibility window  

**Success Criteria**
- Accurate eligibility validation  
- Clear compliance flags  

**Known Pitfall**
Using current eligibility instead of historical.

---

## SCENARIO 10: PRIOR AUTHORIZATION LATENCY

**Objective**  
Calculate median days from PA request to decision.

**Constraints**
- Use window functions  
- Exclude expired requests  

**Success Criteria**
- Correct median calculation  

**Known Pitfall**
Using `AVG()` instead of `PERCENTILE_CONT()`.

---

## SCENARIO 11: ELT PATTERNS & INCREMENTAL PIPELINES

**Objective**  
Build stream-task pipeline for daily plan summaries.

**Constraints**
- Use Streams, Tasks, MERGE  
- Ensure idempotency  

**Success Criteria**
- Correct upserts  
- Handles late reversals  

**Known Pitfall**
Streams don’t handle deletes without explicit MERGE logic.

---

## SCENARIO 12: COST OPTIMIZATION & OBSERVABILITY

**Objective**  
Reduce credit consumption by 45% without changing output.

**Constraints**
- Use pruning, projection, clustering  

**Success Criteria**
- Measurable cost reduction  
- No logic regression  

**Known Pitfall**
Optimizing performance but breaking correctness.

---

## SCENARIO 13: GOVERNANCE & ROW-LEVEL SECURITY

**Objective**  
Implement row-level security and masking.

**Constraints**
- Use Row Access Policies and Dynamic Masking  

**Success Criteria**
- Role-based visibility  
- No performance degradation  

**Known Pitfall**
Masking returns NULL and breaks dashboards.

---

## SCENARIO 14: REAL-WORLD EVIDENCE COHORT

**Objective**  
Identify GLP-1 initiators with T2DM diagnosis and no insulin switch.

**Constraints**
- Align pharmacy and medical claims  

**Success Criteria**
- Accurate cohort logic  

**Known Pitfall**
Cartesian explosion from improper joins.

---

## SCENARIO 15: PRODUCTION DEBUGGING & POSTMORTEM

**Objective**  
Fix missing DIR fee adjustments in December 2023.

**Constraints**
- Query incorrectly filters on `received_date`  

**Success Criteria**
- Fix using `service_month`  
- Provide postmortem  

**Known Pitfall**
Using `CURRENT_DATE` without delay buffer.
