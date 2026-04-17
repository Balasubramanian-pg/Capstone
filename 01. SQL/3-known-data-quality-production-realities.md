## 3. KNOWN DATA QUALITY & PRODUCTION REALITIES

1. **Late-Arriving Claims**: Pharmacy claims adjudicated on day N often arrive in the warehouse 24 to 72 hours later. `adjudication_date` reflects PBM processing. `fill_date` is backdated.
2. **NDC Supersession Drift**: Manufacturers change packaging or reformulate. Old NDCs map to new ones, but historical claims retain old NDCs. Without proper mapping, product-level analytics fragment.
3. **Duplicate Adjudications**: Network outages cause PBM retry logic. Same `claim_id` appears multiple times with identical `gross_amount` but different `created_at`.
4. **Formulary SCD Misalignment**: Tier changes are effective on the first of the month, but claims adjudicated mid-month sometimes pull stale tier values due to batch load timing.
5. **DIR Fee Lag**: DIR fees are reported by payers 14 to 45 days after service month. Late arrivals retroactively adjust net receipts.
6. **Rebate Contract Gaps**: Not all NDCs have active contracts. Some contracts have volume tiers not yet met. Missing contracts default to 0, skewing P&L.
7. **PII/PHI Exposure Risk**: `member_id`, `dob`, `zip_code`, and `prescriber_npi` are regulated. Improper joins or unmasked exports violate HIPAA audit requirements.
8. **Micro-Partition Clumping**: `FACT_PHARMACY_CLAIMS` loads daily without clustering. Queries filtering on `ndc` or `fill_date` scan excessive partitions.
9. **Cost Sensitivity**: Finance enforces a strict monthly compute budget. Queries exceeding 18 minutes or burning >$9.20 in warehouse credits trigger automatic review and dashboard throttling.
10. **340B Compliance**: Covered entities cannot purchase 340B drugs for non-eligible patients. Mixing eligible and ineligible fills triggers audit flags.
