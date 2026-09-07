---
name: prd-to-dt-plan
description: "Analyze PRD-style requirement files and produce a structured implementation plan for updating a target Dynamic Table. Use when: onboarding new sources, applying business rules from a PRD/spec to a DT pipeline, planning Silver-layer changes from requirement docs. Triggers: prd to plan, requirements to DT, onboard source from PRD, analyze PRD, plan DT changes, business rules to pipeline."
---

# PRD to Dynamic Table Plan

Turn product requirement documents into a concrete, reviewable implementation plan for a target Dynamic Table.

## When to Use

- A stakeholder hands you a PRD, spec, or requirements file (XLSX, CSV, or PDF) that describes new sources, field mappings, or business rules
- You need to produce a structured plan showing what changes to make to an existing (or new) Dynamic Table
- You want assumptions surfaced explicitly rather than silently baked into SQL

## Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prd_path` | Yes | Path to the requirements file (supports `.xlsx`, `.csv`). May contain multiple sheets/files covering source onboarding, column mappings, and business rules. |
| `target_dynamic_table` | Yes | Fully qualified DT name (e.g. `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`) |
| `source_schema` | No | Schema where bronze/source tables live (default: inferred from PRD) |

## Workflow

### Step 1: Read and Parse the PRD

1. Read the file at `prd_path` using the Read tool (supports XLSX and CSV)
2. Identify distinct sections. Common PRD patterns:
   - **Source Onboarding** — new systems, platforms, regions, contacts, data delivery method
   - **Column Mapping** — source column → target column, data types, transformations
   - **Business Rules** — normalization, deduplication, filtering, derived fields, quality checks
3. If the file has multiple sheets (XLSX), read each sheet and classify it

**If the file format is unsupported or unreadable, STOP and ask the user.**

### Step 2: Inspect the Current Target DT

1. Run `SHOW DYNAMIC TABLES LIKE '<table>' IN SCHEMA <db>.<schema>`
2. Run `DESCRIBE TABLE <target_dynamic_table>` to get current columns
3. If the DT does not exist yet, note this — the plan will be a CREATE rather than ALTER

### Step 3: Analyze and Classify Changes

For each requirement row, classify it into one of:

| Category | Action |
|----------|--------|
| **New Source** | Add a new UNION ALL branch to the DT query |
| **Column Addition** | Add column to SELECT list across all branches |
| **Column Rename/Remap** | Update alias in affected branch |
| **Business Rule (Silver)** | Add CASE/QUALIFY/filter logic to DT query |
| **Business Rule (Gold/Deferred)** | Note as out-of-scope for this DT; log for downstream |
| **Property Change** | ALTER DT (target_lag, warehouse, refresh_mode) |
| **Ambiguous / Needs Decision** | Surface as open question — do NOT guess |

### Step 4: Surface Assumptions and Open Questions

**This is the most important step.** For every decision that is not explicitly stated in the PRD:

- Flag it as an **OPEN QUESTION**
- State what the default assumption would be if no answer is given
- Identify who should own the decision (if the PRD names stakeholders)

Common ambiguity patterns to watch for:
- Status/enum mappings not explicitly defined for all source values
- Normalization deferred to "later" with no clear layer assignment
- Data format inconsistencies across historical vs current data
- Fields that exist in some sources but not others (nullable vs default?)
- Timing dependencies (legal approvals, go-live dates blocking implementation)

**STOP: Present open questions to the user before producing the final plan.**

### Step 5: Produce the Implementation Plan

Output a structured plan with these exact sections:

---

## Output Format

```markdown
# DT Implementation Plan: <target_dynamic_table>
Generated from: <prd_path>
Date: <today>

## 1. Summary of Changes
- Bullet list of what this plan adds/modifies

## 2. New Sources
| Source System | Bronze Table | Delivery Method | Status |
| (one row per new source being onboarded)

## 3. Column Mapping (New/Changed)
| Silver Column | Source | Source Column | Transformation | Notes |
| (only new or changed mappings — not the full existing schema)

## 4. Business Rules to Implement
| Rule ID | Description | Implementation | Layer |
| (CASE statements, QUALIFY, filters, etc. with layer = Silver or Gold)

## 5. Property Changes
| Property | Current | Proposed | Reason |

## 6. Open Questions (BLOCKING)
| # | Question | Default Assumption | Decision Owner | Impact if Wrong |

## 7. Out of Scope (Deferred to Gold/Phase 2)
| Item | Reason | Phase |

## 8. Proposed SQL (Draft)
The CREATE OR ALTER DYNAMIC TABLE statement incorporating all Silver-layer changes.
Mark sections with -- TODO where open questions block final SQL.

## 9. Validation Queries
SQL queries to verify the plan was applied correctly (row counts, distinct values, etc.)
```

---

## Stopping Points

- **Step 1**: If file is unreadable or ambiguous structure — ask user
- **Step 4**: Present open questions — wait for answers before finalizing plan
- **Step 5**: Present full plan for approval before any DDL execution

## Best Practices

1. **Never guess mappings** — if a source column's target isn't explicit, flag it
2. **Preserve raw data at Silver** — transformations that lose information belong in Gold unless the PRD explicitly says otherwise
3. **Quote the PRD** — when surfacing a rule, cite the rule ID or row so the user can trace back
4. **Separate Silver from Gold** — if a rule says "normalize later" or "future phase", put it in Section 7, not Section 4
5. **Validate against existing DT** — compare PRD column names to current DT schema; flag mismatches
6. **One plan per DT** — if the PRD affects multiple DTs, produce separate plans

## Example Usage

**User prompt:**
```
$prd-to-dt-plan prd_path=./requirements/Q3_AP_updates.xlsx target_dynamic_table=COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
```

**Example output (abbreviated):**

```markdown
# DT Implementation Plan: COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
Generated from: ./requirements/Q3_AP_updates.xlsx
Date: 2025-06-25

## 1. Summary of Changes
- Add Baan IV (EMEA) as new source system
- Add Workday Financial Management (Americas) as new source system
- Implement status normalization (BR-001): map all variants to APPROVED/PENDING
- Add deduplication logic for Baan (BR-003)
- Remove SOURCE_ENTITY_ID and SOURCE_DOCUMENT_TYPE columns (BR-008)
- Change TARGET_LAG to DOWNSTREAM (BR-009)

## 2. New Sources
| Source System | Bronze Table | Delivery | Status |
|---|---|---|---|
| Baan IV | BRONZE_BAAN_AP_INVOICES | Nightly CSV to S3 ~02:00 UTC | Approved |
| Workday | BRONZE_WORKDAY_AP_INVOICES | Hourly via connector | Pending legal (DPA-2025-0041) |

## 6. Open Questions (BLOCKING)
| # | Question | Default | Owner | Impact |
|---|---|---|---|---|
| 1 | Normalize payment terms at Silver or Gold? (BR-005) | Leave raw at Silver | Sarah Chen / David Kim | Inconsistent terms in Silver reporting |
| 2 | Drop SOURCE_ENTITY_ID/SOURCE_DOCUMENT_TYPE? (BR-008 vs current design) | Drop per BR-008 | David Kim | Breaks any downstream joins using these columns |
| 3 | Change to DOWNSTREAM lag now or wait for Gold DT? (BR-009) | Keep 1 hour until Gold exists | Engineering | No impact if Gold not yet built |
```
