# Analysis Framework — Thai Clinical Context

This file defines the perspective dimensions applied to every guideline during Step 3 of the skill flow.

Update this file to adjust how guidelines are analysed — e.g. add a dimension for cost-effectiveness or modify the Thai drug availability sources.

---

## Dimension 1 — Delta from Previous Standard

**Question:** What is materially different in this guideline compared to the previous version or the currently active Thai guideline?

Focus on:
- New drug classes or treatment targets added/removed
- Changed first-line therapy
- Updated diagnostic thresholds (labs, imaging criteria)
- New contraindications or safety warnings
- Changes in monitoring frequency

---

## Dimension 2 — Thai Drug & Device Availability

**Question:** Are the recommended drugs, devices, or tests available in Thailand?

Check against:
- National List of Essential Medicines (NLEM) — บัญชียาหลักแห่งชาติ (latest edition)
- UC Scheme (30-บาท) reimbursable drugs and procedures
- CSMBS (สวัสดิการข้าราชการ) benefit coverage
- SSO (ประกันสังคม) formulary
- Typical availability in community hospital (โรงพยาบาลชุมชน) vs tertiary centre

Rate each key recommendation: `Available` / `Partial` / `Not Available / Off-label`.

---

## Dimension 3 — Infrastructure & Workforce Requirements

**Question:** What infrastructure or specialist workforce does this guideline assume?

Consider:
- Need for subspecialty (cardiologist, nephrologist, etc.)
- Monitoring technology (TDM, advanced imaging, genetic testing)
- Procedure capability (electrophysiology lab, TAVI centre)
- Suitable for primary care / district hospital setting?

---

## Dimension 4 — Ethnicity & Population Data

**Question:** Does the guideline cite data from Asian or Thai populations?

- If yes: note which studies and whether Thai patients were included.
- If no: flag potential extrapolation risk, especially for:
  - Drug dosing (e.g., warfarin, clopidogrel — CYP2C19 polymorphism)
  - BMI / cardiovascular risk thresholds (Asian-specific cut-offs)
  - Incidence assumptions

---

## Dimension 5 — Implementation Priority for Thai Hospitals

**Question:** What is the single most actionable change Thai clinicians should make?

Score each guideline:

| Priority | Criteria |
|----------|----------|
| **High** | Change affects first-line therapy for common condition; drugs available; evidence strong |
| **Medium** | Change is evidence-based but drug/device access is partial, or condition is less common |
| **Low** | Aspirational recommendation; significant access/cost barrier; limited Thai-relevant data |

---

## Dimension 6 — Regulatory & Policy Implications

**Question:** Does this guideline change require policy action (FDA registration, NLEM listing, clinical practice guideline revision by Thai medical societies)?

- Tag relevant Thai societies that should be notified (e.g., ราชวิทยาลัยอายุรแพทย์แห่งประเทศไทย, สมาคมความดันโลหิตสูงแห่งประเทศไทย)
- Note if WHO/FIGO/ISH position conflicts with current Thai MOH policy

---

## Article Voice Guidelines

- **Primary audience:** Thai internists, GPs, hospital pharmacists
- **Language:** Thai body text; English for drug names, acronyms, proper nouns
- **Tone:** Collegial, evidence-first, practical
- **Avoid:** Promotional language, unqualified superlatives, first-person hedging ("I think...")
- **Action Items table** must be role-specific (Clinician / Pharmacist / Policy) and ranked by priority
