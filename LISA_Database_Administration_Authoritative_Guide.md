# LISA Database Administration Authoritative Guide

**System:** OSHA Laboratory Information System (LISA / LIMS)  
**Primary database areas covered:** LIMS transactional schema, QC (`QC_DEV2`), OIS staging/transfer support, analyst/user administration  
**Purpose:** Stand-alone operational reference for a human DBA or an LLM assisting a DBA.  
**Basis:** Consolidated from the supplied LISA Maintenance presentations, LIMS DSD/ERD diagrams, and verified live-database administration sessions.

> **Scope and authority.** This guide is authoritative for the administration tasks actually documented in the supplied materials and for the QC unpost/theoretical-value workflow verified interactively against the live database. It does **not** claim to document every object, procedure, form, batch job, or business rule in LISA. Where the source material is silent, this guide says so rather than inventing behavior.

---

## 1. Operating principles

### 1.0 Evidence and authority labels

This guide uses the following evidence labels. When facts conflict, **current live database metadata and verified behavior take precedence over historical diagrams and examples**, but discrepancies must be recorded rather than silently erased.

- **SOURCE-DOCUMENTED** — stated or shown in the supplied Maintenance, DSD, or ERD material.
- **LIVE-VERIFIED 2026-09-01** — observed against the live database during the documented DBA session.
- **SOURCE-DOCUMENTED + LIVE-VERIFIED** — supported by both.
- **HISTORICAL** — useful legacy information that may no longer reflect the current environment.
- **ENVIRONMENT-DEPENDENT** — server names/IPs, paths, schedules, roles, profiles, recipients, or other values that must be verified before use.
- **INFERRED** — a reasoned conclusion not explicitly established by a source; do not treat it as a database rule without verification.
- **SOURCE-DEFECT** — an apparent error or contradiction in inherited documentation; do not execute it without independent verification.

### 1.0.1 Values an operator or LLM must never infer

Never infer an `SMST_ID`, `SMPL_ID`, `ANLY_ID`, `LBST_ID`, `QCSMPL_ID`, `SPKSOL_ID`, `ANALYTE_ID`, physical object behind a synonym, current server address/path, current Oracle role/profile, status-code meaning, foreign-key target, or side effect of an undocumented procedure. Retrieve or verify it first.

### 1.0.2 Preferred mechanism hierarchy

Prefer the supported application workflow when it can safely perform the requested task. Use direct SQL for administrative repair, bulk correction, or operations the application cannot perform. The inherited documentation specifically notes that chemists can change an analyte one sample at a time through the Edit screen, while SQL is useful for multiple samples; QCSM unassignment is preferably performed by IHC supervisors through `QCSMUASGN.fmx`.


### 1.1 Transaction safety

Changes made in SQL Developer are session-local until committed. Use `ROLLBACK;` to abandon unwanted uncommitted changes. After `COMMIT;`, reversing a change requires a compensating `UPDATE`, `INSERT`, or `DELETE`.

This transaction pattern primarily applies to transactional DML such as `UPDATE`, `INSERT`, and `DELETE`. Oracle account/schema administration commands such as `ALTER USER`, `CREATE USER`, and `GRANT` are DDL/security administration and have different transaction semantics; do not rely on `ROLLBACK` to reverse them. Verify their effects through the appropriate account or security checks.

**Recommended DBA pattern for every data correction:**

1. Identify the target rows with a `SELECT`.
2. Record the original values.
3. Make the narrowest possible change, preferably including the expected old value in the `WHERE` clause.
4. Check the reported row count.
5. Re-query the affected rows.
6. Inspect related rows if the change may have derived or parent/child effects.
7. `ROLLBACK;` if anything is unexpected.
8. `COMMIT;` only after final verification.
9. Run a post-commit verification query for important changes.

Never assume that an application label such as *posted*, *assigned*, or *finished* maps to a particular database field until verified from documentation, schema metadata, or stored code.

### 1.2 SQL conventions

A line beginning with `--` is a comment. A `--` appearing within a line comments out the remainder of that line. Some PowerPoint examples contain typographic (“smart”) single quotes; replace them with normal ASCII `'` quotes if SQL Developer reports syntax errors.

**Placeholder convention:** Text enclosed in angle brackets, such as `<USERNAME>`, `<SMST_ID>`, or `<OBJECT_NAME>`, represents a value that must be replaced before execution. The angle brackets themselves are not part of the SQL. Never execute an example containing unresolved `<...>` placeholders.

### 1.3 Schema discovery

LISA uses synonyms extensively. A visible name such as `Q_SETS` may not be the physical table name. When an object is not found where expected, resolve it rather than guessing.

```sql
SELECT owner, object_name, object_type
FROM all_objects
WHERE object_name = UPPER('<OBJECT_NAME>')
ORDER BY owner, object_type;
```

For a public synonym:

```sql
SELECT owner, synonym_name, table_owner, table_name, db_link
FROM all_synonyms
WHERE owner = 'PUBLIC'
  AND synonym_name = UPPER('<SYNONYM_NAME>');
```

Inspect physical table columns with:

```sql
SELECT column_id, column_name, data_type,
       data_length, data_precision, data_scale,
       nullable, data_default
FROM all_tab_columns
WHERE owner = UPPER('<OWNER>')
  AND table_name = UPPER('<TABLE_NAME>')
ORDER BY column_id;
```

Inspect triggers:

```sql
SELECT trigger_name, status, triggering_event
FROM all_triggers
WHERE table_owner = UPPER('<OWNER>')
  AND table_name = UPPER('<TABLE_NAME>')
ORDER BY trigger_name;
```

Search stored source for a column, table, or business rule:

```sql
SELECT owner, name, type, line, text
FROM all_source
WHERE UPPER(text) LIKE UPPER('%<SEARCH_TERM>%')
ORDER BY owner, name, type, line;
```

Inspect constraints and their columns when diagnosing integrity errors:

```sql
SELECT owner, constraint_name, constraint_type, table_name, search_condition
FROM all_constraints
WHERE owner = UPPER('<OWNER>')
  AND constraint_name = UPPER('<CONSTRAINT_NAME>');
```

```sql
SELECT owner, constraint_name, table_name, column_name, position
FROM all_cons_columns
WHERE owner = UPPER('<OWNER>')
  AND constraint_name = UPPER('<CONSTRAINT_NAME>')
ORDER BY position;
```

`NULLABLE = 'N'` in `ALL_TAB_COLUMNS` means a value is required unless another mechanism supplies it. This is especially important before constructing an `INSERT`.

Then retrieve the full source of a discovered program unit before executing it:

```sql
SELECT line, text
FROM all_source
WHERE owner = UPPER('<OWNER>')
  AND name = UPPER('<PROGRAM_NAME>')
  AND type = UPPER('<TYPE>')
ORDER BY line;
```

---

## 2. LISA data model: practical mental model

### 2.0 Application, privilege, and data-entry architecture

**Evidence: SOURCE-DOCUMENTED.** LISA was designed for manual entry through Oracle Forms. Analysts historically have the `SELECT`, `INSERT`, and `UPDATE` privileges needed for transactional processing, while operational restrictions are enforced through LISA Forms; ordinary analyst users do not have SQL*Plus or SQL Developer access. Direct-SQL procedures in this guide assume appropriately privileged DBA/support access and are not analyst-user procedures.

LISA has two documented data-entry paths:

```text
Manual entry
    -> LISA Forms
    -> LIMS transactional tables

OIS electronic request
    -> request XML
    -> SOURCE_RECEIVE
    -> LIMS_STAGE
    -> LISA login form searches staging after the sampling-sheet key is entered
    -> user saves the form
    -> LIMS transactional tables
```

A record's presence in `LIMS_STAGE` therefore does **not** by itself establish that the sampling sheet has been saved into transactional LISA. This distinction is important when troubleshooting an OIS request that appears to have arrived but is not visible as a normal LISA record.


LISA is a relational, normalized database designed around data entry. Parent/child relationships are central to maintenance work. The principal transactional hierarchy described by the documentation is approximately:

```text
Office / Inspection / Sampling Sheet  -> SAMPLE_SETS
                                      |
                                      +-> SAMPLED_EMPLOYEES -> OCCUPATIONS
                                      |
                                      +-> SAMPLES
                                           |
                                           +-> TIME_CHECKS
                                           |
                                           +-> ANALYSIS_TARGETS
                                                |
                                                +-> ANALYTES
                                                +-> ANALYTICAL_METHODS
                                                +-> INSTRUMENTS
                                                +-> LAB_SETS
```

For performance, office, inspection, and sampling-sheet information is combined in `SAMPLE_SETS`; its primary identifier is `SMST_ID`. `SAMPLES.SMST_SMST_ID` links a sample to the sampling sheet. `ANALYSIS_TARGETS.SMPL_SMPL_ID` links an analyte/result target to a sample. `ANALYSIS_TARGETS.LBST_LBST_ID` links it to a laboratory analytical set.

### 2.1 Main transactional tables

- **SAMPLE_SETS** — office, inspection number, sampling number, sampling/shipping dates, 8-hour TWA request, and related sampling-sheet information.
- **SAMPLED_EMPLOYEES** — sampled employee and occupation information.
- **SAMPLES** — sample type, media, CSHO sample identifier (`SUBMISSION_NO`), blank status, sampling time/flow, weight, and laboratory assigned number.
- **ANALYSIS_TARGETS** — analyte, A/B results, units, solution volume, qualifier, reporting/detection information, comments, method, instrument, and lab-set assignment.
- **LAB_SETS** — analytical method/instrument, comments, and chain-of-custody/workflow fields including assigned, posted, checked, and released information.

The DSD/ERD also documents supporting entities including analysts, analytes, analytical methods, instruments, collection media, media/analyte mappings, substances, units of measure, qualifiers, comments, stations, field offices, compliance officers, occupations, PPE, exposure limits, applicable calculations, and historical SAE.

### 2.2 Important LIMS keys and relationships

Common relationships documented in the DSD include:

```text
SAMPLES.SMST_SMST_ID              -> SAMPLE_SETS.SMST_ID
ANALYSIS_TARGETS.SMPL_SMPL_ID     -> SAMPLES.SMPL_ID
ANALYSIS_TARGETS.ANLT_ANALYTE_ID  -> ANALYTES.ANALYTE_ID
ANALYSIS_TARGETS.ANMT_ANMT_ID     -> ANALYTICAL_METHODS.ANMT_ID
ANALYSIS_TARGETS.INST_INST_ID     -> INSTRUMENTS.INST_ID
ANALYSIS_TARGETS.LBST_LBST_ID     -> LAB_SETS.LBST_ID
TIME_CHECKS.SMPL_SMPL_ID          -> SAMPLES.SMPL_ID
SAMPLED_EMPLOYEES.SMST_SMST_ID    -> SAMPLE_SETS.SMST_ID
```

When querying multiple tables, join them through these keys. Most maintenance examples use inner joins.

---

## 3. General sample lookup

A useful sample/sampling-sheet lookup is:

```sql
SELECT lab_assigned_no,
       smpl_id,
       smst_smst_id,
       inspection_no,
       sampling_number,
       flof_office_id,
       submission_no,
       sampled_establishment,
       recvd_in_lab,
       sampling_date,
       shipping_date
FROM l_samples, l_sample_sets
WHERE smst_id = smst_smst_id
  AND lab_assigned_no BETWEEN 'F12346' AND 'F12346'
--AND sampling_number IN ('428893')
--AND inspection_no LIKE '1757190'
ORDER BY 1,5;
```

Choose the identifier available to you and comment/uncomment the appropriate predicate.

---

## 4. Sample assignment and unassignment

### 4.1 Assignment model

When samples are logged into LISA, each sample/requested-analyte combination is inserted into `ANALYSIS_TARGETS` with `LBST_LBST_ID = 0`. Lab set 0 represents unassigned work; only those rows appear in the Assign form. Assigning samples creates/uses a nonzero lab set and updates the selected analysis-target rows to that set ID.

**Do not edit the lab-set-0 record itself.** Doing so can prevent unassigned samples from appearing correctly.

### 4.2 Inspect assignment status

```sql
SELECT anly_id,
       smpl_id,
       lab_assigned_no,
       submission_no,
       lbst_lbst_id,
       labno_range,
       anlt_analyte_id,
       analyte_code IMIS,
       smty_type_code AS type,
       assigned_to,
       date_assigned,
       checked_by,
       checked_on,
       released_by,
       released_on
FROM l_samples a, l_analysis_targets b, l_analytes c, l_lab_sets
WHERE smpl_id = smpl_smpl_id
  AND anlt_analyte_id = analyte_id
  AND lbst_lbst_id = lbst_id
  AND lab_assigned_no BETWEEN 'F24968' AND 'F24973'
ORDER BY lab_assigned_no, anlt_analyte_id;
```

### 4.3 Add an unassigned analyte to an existing set

First find the target set ID from a sample/analyte already in the desired set:

```sql
SELECT lbst_lbst_id
FROM l_analysis_targets
WHERE smpl_smpl_id IN (
    SELECT smpl_id FROM l_samples WHERE lab_assigned_no = 'F24968'
)
AND anlt_analyte_id IN (
    SELECT analyte_id FROM l_analytes WHERE analyte_code = 'G301'
);
```

Then update the desired sample/analyte to that set:

```sql
UPDATE l_analysis_targets
SET lbst_lbst_id = <TARGET_LBST_ID>
WHERE smpl_smpl_id IN (
    SELECT smpl_id FROM l_samples WHERE lab_assigned_no = 'F24969'
)
AND anlt_analyte_id IN (
    SELECT analyte_id FROM l_analytes WHERE analyte_code = '9130'
);
```

> **SOURCE-DEFECT — DO NOT USE WITHOUT LIVE VERIFICATION.** A later/shorter Maintenance slide shows a “simplified” assignment sequence that retrieves `SMPL_ID = 445647` and then uses `445647` as the value assigned to `ANALYSIS_TARGETS.LBST_LBST_ID`. This conflicts with the documented schema: `LBST_LBST_ID` references `LAB_SETS.LBST_ID`, not `SAMPLES.SMPL_ID`. Do not use that example as written. Obtain the actual target `LBST_ID` from an existing analysis target in the desired set, as shown above.

### 4.4 Remove analytes from an analytical set

Example from the maintenance material:

```sql
UPDATE l_analysis_targets
SET lbst_lbst_id = 0
WHERE smpl_smpl_id IN (
    SELECT smpl_id
    FROM l_samples
    WHERE lab_assigned_no BETWEEN 'F47605' AND 'F47607'
)
AND anlt_analyte_id IN (
    SELECT analyte_id
    FROM l_analytes
    WHERE analyte_code IN ('G301','G302')
)
AND lbst_lbst_id IN (
    SELECT lbst_lbst_id
    FROM l_analysis_targets, l_analytes
    WHERE anlt_analyte_id = analyte_id
      AND analyte_code IN ('G301')
);
```

The documentation notes that a sample cannot be truly unassigned from an analyst merely by changing initials/date in the LISA Edit form; doing that leaves it in a separate nonzero set. For QCSMs, unassignment is best performed by IHC supervisors using `QCSMUASGN.fmx`.

---

## 5. Deleting a sample

Because `SAMPLES` has child rows (notably `ANALYSIS_TARGETS` and `TIME_CHECKS`), delete children before parents.

Example sequence from the maintenance documentation:

```sql
DELETE l_analysis_targets
WHERE smpl_smpl_id IN (
    SELECT smpl_id FROM l_samples
    WHERE lab_assigned_no BETWEEN '156484' AND '156484'
);

DELETE l_time_checks
WHERE smpl_smpl_id IN (
    SELECT smpl_id FROM l_samples
    WHERE lab_assigned_no BETWEEN '156484' AND '156484'
);

SELECT smst_smst_id
FROM l_samples
WHERE lab_assigned_no BETWEEN '156484' AND '156484';

-- Save the returned SMST_ID before continuing.

DELETE l_samples
WHERE lab_assigned_no BETWEEN '156484' AND '156484';

DELETE l_sampled_employees
WHERE smst_smst_id IN (<SMST_ID>);

DELETE l_sample_sets
WHERE smst_id IN (<SMST_ID>);
```

**Mandatory blast-radius check before parent deletion:** before deleting from `SAMPLED_EMPLOYEES` or `SAMPLE_SETS`, query all remaining samples and dependent entities sharing the `SMST_ID`. Do not remove a sampling-sheet parent merely because one lab sample was deleted. The source provides the historical sequence but does not establish that every sampling sheet can safely be removed after deleting one lab number. Use `ALL_CONSTRAINTS`/`ALL_CONS_COLUMNS` when dependencies are uncertain.

---

## 6. Changing analytes on samples

For many samples, an analyte can be changed with SQL Developer. If more than one analyte must be changed, perform each analyte replacement separately.

```sql
UPDATE l_analysis_targets
SET anlt_analyte_id = (
    SELECT analyte_id
    FROM l_analytes
    WHERE analyte_code = '9130'
)
WHERE smpl_smpl_id IN (
    SELECT smpl_id
    FROM l_samples
    WHERE lab_assigned_no BETWEEN 'F47605' AND 'F47607'
)
AND anlt_analyte_id IN (
    SELECT analyte_id
    FROM l_analytes
    WHERE analyte_code = 'G301'
);
```

Always query the affected rows before and after the update and check for unique-constraint implications.

---

## 7. Adding missing analysis targets

If a sample was logged but has no analysis target, add the missing analyte individually.

First obtain the analyte ID:

```sql
SELECT analyte_code, analyte_id
FROM l_analytes
WHERE analyte_code = 'G302';
```

Example result in the source: `G302 -> 1106`.

Then insert the target:

```sql
INSERT INTO l_analysis_targets
    (anly_id,
     requested_analyte_flag,
     result_portion_a,
     detection_limit,
     uom_abbreviation,
     lbst_lbst_id,
     anlt_analyte_id,
     smpl_smpl_id)
SELECT l_anly_seq.NEXTVAL,
       'Y',
       0,
       0,
       'AAAAA',
       0,
       1106,
       smpl_id
FROM l_samples
WHERE lab_assigned_no BETWEEN 'F46843' AND 'F46844';
```

The source states that multiple analytes must be inserted individually.

---

## 8. Changing sampling-sheet “primary key” information

Office ID, inspection number, and sampling number participate in uniqueness rules and may not be editable through the normal LISA form.

Locate the sampling sheet:

```sql
SELECT smst_id, flof_office_id, inspection_no, sampling_number
FROM l_sample_sets
WHERE sampling_number = '445162';
```

Then update the desired field(s) by `SMST_ID`:

```sql
UPDATE l_sample_sets
SET flof_office_id = 3384115
--, inspection_no = '84070-6424'
--, sampling_number = '945162'
WHERE smst_id = 151730;
```

Only uncomment the field(s) actually being changed. Verify uniqueness before committing.

---

## 9. Counting samples analyzed by an analyst

Example:

```sql
SELECT analyte_code, COUNT(DISTINCT smpl_smpl_id)
FROM l_analysis_targets, l_analytes, l_lab_sets
WHERE lbst_lbst_id = lbst_id
  AND anlt_analyte_id = analyte_id
  AND assigned_to = 'DHALTERMAN'
  AND analyte_code = 'S777'
  AND released_on >= DATE '2019-01-01'
  AND released_on <  DATE '2024-07-01'
GROUP BY analyte_code;
```

When using aggregate functions such as `COUNT` or `SUM`, nonaggregated selected expressions generally need to appear in `GROUP BY`.

---

## 10. Resolving the “unique constraint” substance/analyte error

Some LISA analyte codes are substances composed of multiple component analytes. The documented common example is `2587` (Welding Fumes), which returns results for multiple metals. When the Results form decomposes a substance into components, a unique-constraint error occurs if a component analyte already exists for the sample because LISA does not permit duplicate results for the same analyte/sample combination.

Find assigned rows involving analytes that are substance components:

```sql
SELECT lab_assigned_no,
       lbst_id,
       analyte_id,
       analyte_code,
       date_assigned,
       assigned_to
FROM l_samples, l_analysis_targets, l_lab_sets, l_analytes
WHERE smpl_smpl_id = smpl_id
  AND lbst_lbst_id = lbst_id
  AND anlt_analyte_id = analyte_id
  AND lbst_lbst_id > 0
  AND anlt_analyte_id IN (
      SELECT DISTINCT anlt_subs_id FROM l_substances
  )
ORDER BY 1,2;
```

Then inspect all analysis targets for the affected lab numbers and determine which rows duplicate components that will be created from the substance. The source's worked example used lab numbers `A12345`/`A12346`, lab set `235028`, and retained analyte `22365` (`PB-B`) while removing four other rows. The example illustrates the pattern, not a generally safe delete rule.

The documented example contains this broad delete:

```sql
DELETE l_analysis_targets
WHERE lbst_lbst_id = 235028
  AND anlt_analyte_id <> 22365;
```

**POTENTIALLY DESTRUCTIVE HISTORICAL EXAMPLE — do not generalize it.** Identify the substance and component analytes for the actual case and delete only confirmed conflicting rows. Re-query afterward and reopen the Results form to confirm the constraint error is gone.

---

## 11. QC subsystem (`QC_DEV2`)

The QC schema is `QC_DEV2`. QC tables refer to LIMS objects for common elements such as analysts, analytes, instruments, media, and methods. Documented maintenance commonly involves batch expiration dates, set assignment, killing/unposting samples, and theoretical (ASM) values.

### 11.1 Public synonym mapping verified in the live database

During the verified 2026-09-01 maintenance session:

```text
Q_SETS             -> QC_DEV2.SETS
Q_QC_SAMPLES       -> QC_DEV2.QC_SAMPLES
Q_SPIKED_SOLUTIONS -> QC_DEV2.SPIKED_SOLUTIONS
```

Do not assume all QC synonyms follow this exact naming pattern; resolve unfamiliar objects through `ALL_SYNONYMS`.

### 11.2 Relevant verified QC table structures

`QC_DEV2.SETS` columns observed:

```text
SET_ID, QCBTC_QCBTC_ID, ASSIGNED_DT, RESULTS_DT,
ANA_ANALYST_ID_ASGTO, ANA_ANALYST_ID_RESULT,
INST_INST_ID, ANMT_ANMT_ID, COMMENTS, PRECISION,
LBST_LBST_ID, BLK_CORR_IND, CAR_FLAG
```

`QC_DEV2.QC_SAMPLES` columns observed:

```text
QCSMPL_ID, LAB_NUMBER, SPKNT_SPKNT_ID, SET_SET_ID,
SOL_SEQ, SOL_VOL1, SOL_VOL2, SOL_VOL3, BLANK,
SAMPLE_STATUS, BCWT_BCWT_ID
```

`QC_DEV2.SPIKED_SOLUTIONS` columns observed:

```text
SPKSOL_ID, ANA_ANALYST_ID, FOUND, FOUND_THEOR_RATIO,
LCL, MEAN, SAE, SD, THEORETICAL, QCSMPL_QCSMPL_ID,
ANLT_ANALYTE_ID, INST_INST_ID, ANMT_ANMT_ID,
REQUESTED_ANALYTE_FLAG, UCL, UOM_ID, IN_CONTROL,
SET_SET_ID, FLAG, INCLUDED, INCLUDED_DATE, PRECISION,
P_MEAN, P_SD, P_LCL, P_UCL, P_IN_CONTROL,
P_INCLUDED, P_INCLUDED_DATE
```

Important relationships verified:

```text
QC_SAMPLES.QCSMPL_ID          -> SPIKED_SOLUTIONS.QCSMPL_QCSMPL_ID
QC_SAMPLES.SET_SET_ID         -> SETS.SET_ID
SPIKED_SOLUTIONS.SET_SET_ID   -> SETS.SET_ID
```

### 11.3 QC status values

The maintenance documentation defines:

```text
0LQ = logged
0AQ = assigned
0FQ = finished
```

### 11.4 Inspect QC samples

```sql
SELECT lab_number, set_set_id, sample_status
FROM q_qc_samples
WHERE lab_number BETWEEN <FIRST_QC> AND <LAST_QC>
ORDER BY lab_number;
```

For detailed inspection:

```sql
SELECT qcsmpl_id,
       lab_number,
       spknt_spknt_id,
       set_set_id,
       sol_seq,
       sol_vol1,
       sol_vol2,
       sol_vol3,
       blank,
       sample_status,
       bcwt_bcwt_id
FROM q_qc_samples
WHERE lab_number BETWEEN <FIRST_QC> AND <LAST_QC>
ORDER BY lab_number;
```

### 11.5 `LISA-QC-001` — Unpost a finished QC set/sample

**Evidence:** sample-status transition is SOURCE-DOCUMENTED + LIVE-VERIFIED 2026-09-01; set-level `RESULTS_DT` reset is LIVE-VERIFIED 2026-09-01 from the QC 521028–521031 incident and is not stated in the inherited Maintenance slides.  
**Preferred mechanism:** DBA SQL for the verified repair case  
**Objects affected:** `Q_QC_SAMPLES` / `QC_DEV2.QC_SAMPLES` and `Q_SETS` / `QC_DEV2.SETS`  
**Precondition:** requested QC is currently `0FQ` (finished), its parent `SET_ID` has been verified, and the operator has inspected all QC samples in that set.

A finished QC has both sample-level and set-level workflow state. Live inspection of `QC_DEV2.UPDATE_PRECS2` established that finishing a QC set sets `Q_SETS.RESULTS_DT = SYSDATE` and sets all `QC_SAMPLES` in the set to `SAMPLE_STATUS = '0FQ'`. Live verification on 2026-09-01 established that, for the previously finished set investigated, changing only `QC_SAMPLES.SAMPLE_STATUS` was insufficient to restore application visibility. Subsequent investigation identified the retained completion timestamp in `SETS.RESULTS_DT` as an inconsistent set-level state, and it was reset to the active/uncompleted sentinel. Accordingly, the verified unpost procedure treats sample-level and set-level state together. This is a live-verified workflow for the investigated whole-set case, not proof that every possible QC unpost under every workflow condition has identical requirements.

First identify the target QC rows and parent set, and inspect every sample in that set:

```sql
SELECT lab_number, qcsmpl_id, set_set_id, sample_status, blank
FROM q_qc_samples
WHERE lab_number BETWEEN <FIRST_QC> AND <LAST_QC>
ORDER BY lab_number;

SELECT lab_number, sample_status, blank
FROM q_qc_samples
WHERE set_set_id = <SET_ID>
ORDER BY lab_number;
```

Inspect the exact set-level results timestamp before changing it:

```sql
SELECT set_id,
       results_dt,
       TO_CHAR(results_dt, 'YYYY-MM-DD HH24:MI:SS') AS results_dt_exact,
       ana_analyst_id_asgto
FROM q_sets
WHERE set_id = <SET_ID>;
```

The sample-level transition is:

```sql
UPDATE q_qc_samples
SET sample_status = '0AQ'
WHERE lab_number = <QC_NUMBER>
  AND sample_status = '0FQ';
```

Use the old-status predicate so an unexpected state produces zero updated rows rather than silently overwriting it. Expect exactly one row for each requested QC.

For a set being returned from the finished state, reset the parent set's `RESULTS_DT` to the active/uncompleted sentinel. On 2026-09-01, live comparison against an active `0AQ` set established the sentinel as exactly `1900-01-01 00:00:00`. This value is LIVE-VERIFIED, not source-documented; if the environment has changed or contrary evidence exists, verify the sentinel against a current active `0AQ` set before use. Use the exact old timestamp as a guard:

```sql
UPDATE q_sets
SET results_dt = DATE '1900-01-01'
WHERE set_id = <SET_ID>
  AND results_dt = TO_DATE(
        '<EXPECTED_OLD_RESULTS_TIMESTAMP>',
        'YYYY-MM-DD HH24:MI:SS'
      );
```

Expect exactly one row. If zero rows are updated, stop and re-query the exact timestamp; Oracle `DATE` values may contain a time component even when SQL Developer displays only the date. Do not weaken the guard merely to force a match.

Verify both levels before committing:

```sql
SELECT s.set_id,
       TO_CHAR(s.results_dt, 'YYYY-MM-DD HH24:MI:SS') AS results_dt,
       s.ana_analyst_id_asgto,
       q.lab_number,
       q.sample_status
FROM q_sets s
JOIN q_qc_samples q
  ON q.set_set_id = s.set_id
WHERE s.set_id = <SET_ID>
ORDER BY q.lab_number;
```

For the verified workflow, the intended unposted state is `QC_SAMPLES.SAMPLE_STATUS = '0AQ'` for the requested samples and `SETS.RESULTS_DT = DATE '1900-01-01'` for the parent set. Because `RESULTS_DT` is set-level state, do not reset it until the complete membership and requested scope of the set have been inspected.

> **Scope limitation — partial-set unpost not yet verified.** The 2026-09-01 live verification involved unposting every QC sample in set 27514. The correct set-level behavior when only a subset of samples in a previously finished QC set is to be unposted has not been established. If a request targets fewer than all members of a set, perform the read-only discovery steps, but do not reset `SETS.RESULTS_DT` based solely on this procedure. Investigate the application workflow and/or stored code first and document the verified behavior.

### 11.6 `LISA-QC-002` — Correct an ASM/theoretical value

**Evidence:** SOURCE-DOCUMENTED + LIVE-VERIFIED 2026-09-01  
**Preferred mechanism:** documented source allows SQL Developer table edit; tightly constrained DBA SQL is verified  
**Objects affected:** `Q_SPIKED_SOLUTIONS` / `QC_DEV2.SPIKED_SOLUTIONS`  
**Required identifiers:** QC number, `QCSMPL_ID`, `SPKSOL_ID`, `ANALYTE_ID`, expected old theoretical value  
**Expected modification row count:** exactly one row per requested correction


The maintenance slides state that ASM corrections—often caused by wrong extraction efficiency, density, or similar calculation input—are made in `QC_DEV2.SPIKED_SOLUTIONS`. The slides suggest filtering/editing the table in SQL Developer; SQL `UPDATE` is equally appropriate when tightly constrained.

First identify the exact row and verify the old value:

```sql
SELECT q.lab_number,
       q.qcsmpl_id,
       s.spksol_id,
       s.anlt_analyte_id,
       s.theoretical,
       s.found,
       s.found_theor_ratio,
       s.set_set_id,
       s.in_control,
       s.included
FROM q_qc_samples q
JOIN q_spiked_solutions s
  ON s.qcsmpl_qcsmpl_id = q.qcsmpl_id
WHERE q.lab_number = <QC_NUMBER>
ORDER BY s.anlt_analyte_id;
```

If exactly the expected row/value is found, update narrowly:

```sql
UPDATE q_spiked_solutions
SET theoretical = <NEW_VALUE>
WHERE spksol_id = <SPKSOL_ID>
  AND qcsmpl_qcsmpl_id = <QCSMPL_ID>
  AND anlt_analyte_id = <ANALYTE_ID>
  AND theoretical = <EXPECTED_OLD_VALUE>;
```

Expect one row updated. Do not commit until verified.

### 11.7 `LISA-QC-003` — Complete whole-set “unpost and update theoretical value” workflow

This workflow was verified against the live database on 2026-09-01, including the set-level correction discovered after the initial sample-status-only unpost did not make the QCs visible to the assigned analyst.

1. Query the requested QC numbers and confirm they are `0FQ`.
2. Determine their `QCSMPL_ID`, `SET_SET_ID`, blank status, and corresponding `SPIKED_SOLUTIONS` rows.
3. Inspect **all** QC samples in each affected set before changing set-level state.
4. Inspect and record the exact current `Q_SETS.RESULTS_DT`, including its time component.
5. Confirm each existing `THEORETICAL` value exactly matches the requester's stated old value.
6. Change `QC_SAMPLES.SAMPLE_STATUS` from `0FQ` to `0AQ` for each requested QC, using the expected old status as a guard.
7. For QCs requiring value correction, update only `SPIKED_SOLUTIONS.THEORETICAL` using exact row IDs and the expected old value in the `WHERE` clause.
8. For an “unpost only” QC, do not change its theoretical value.
9. **For the live-verified whole-set case,** reset the affected parent set's `Q_SETS.RESULTS_DT` from its verified completion timestamp to the live-verified active sentinel `DATE '1900-01-01'`, using the exact old timestamp in the `WHERE` clause. If the current environment gives contrary evidence, verify the sentinel against a current active `0AQ` set before use. **If fewer than all members of the set are being unposted, do not perform this step based solely on this procedure; see the partial-set scope limitation in `LISA-QC-001`.**
10. Run a consolidated verification query that includes both set-level `RESULTS_DT` and sample-level status/value state.
11. Do **not** manually update `FOUND_THEOR_RATIO` merely because `THEORETICAL` changed; see the next section.
12. Commit only after all requested rows and the parent set match the intended state.
13. For an important repair, run the same verification after `COMMIT` and have the requester confirm application visibility before making further changes.

**Expected application result:** After commit, the assigned analyst can again locate/open the unposted QC set through the normal QC workflow. Treat success at three distinct levels: (1) SQL success — expected row counts; (2) database-state success — `0AQ`, active `RESULTS_DT` sentinel, and requested `THEORETICAL` values; and (3) application/workflow success — analyst confirmation that the QCs are visible and processable. Do not equate a successful SQL row count with end-to-end resolution.

Consolidated verification query:

```sql
SELECT s.set_id,
       TO_CHAR(s.results_dt, 'YYYY-MM-DD HH24:MI:SS') AS results_dt,
       s.ana_analyst_id_asgto,
       q.lab_number,
       q.sample_status,
       q.blank,
       sp.spksol_id,
       sp.anlt_analyte_id,
       sp.theoretical,
       sp.found,
       sp.found_theor_ratio,
       sp.in_control,
       sp.included
FROM q_sets s
JOIN q_qc_samples q
  ON q.set_set_id = s.set_id
LEFT JOIN q_spiked_solutions sp
  ON sp.qcsmpl_qcsmpl_id = q.qcsmpl_id
WHERE s.set_id = <SET_ID>
ORDER BY q.lab_number, sp.anlt_analyte_id;
```

**2026-09-01 lesson learned:** changing only `QC_SAMPLES.SAMPLE_STATUS` from `0FQ` to `0AQ` was insufficient for QC 521028–521031. It left set 27514 with `RESULTS_DT = 2026-08-28 12:35:20`, producing a mixed state, and the assigned analyst reported that the QCs were not visible. Resetting `SETS.RESULTS_DT` to the verified active sentinel brought the database-side state into alignment with the active `0AQ` state observed in the comparison set. Application visibility should be confirmed by the assigned analyst after commit before treating the repair as end-to-end verified.

### 11.8 `FOUND_THEOR_RATIO` and `UPDATE_PRECS2`

Live source inspection established that changing `SPIKED_SOLUTIONS.THEORETICAL` directly does **not** itself recalculate `FOUND_THEOR_RATIO` (no accessible trigger was found on `QC_DEV2.SPIKED_SOLUTIONS`). Stored procedure `QC_DEV2.UPDATE_PRECS2(qcset_id)` contains the normal recalculation:

```sql
update q_spiked_solutions
set found_theor_ratio = decode(theoretical, 0,0,found/theoretical)
where qcsmpl_qcsmpl_id in (
  select qcsmpl_id from q_qc_samples where set_set_id=qcset_id
);
```

However, `UPDATE_PRECS2` also:

- sets `Q_SETS.RESULTS_DT = SYSDATE` for the set;
- sets **all** `QC_SAMPLES` in the set to `SAMPLE_STATUS = '0FQ'`;
- commits internally;
- recalculates `PRECISION` using `F_PRECISION` for each spiked-solution row;
- commits during that loop.

Therefore **do not call `UPDATE_PRECS2` merely to refresh `FOUND_THEOR_RATIO` after an unpost/theoretical correction**. Doing so would finish/repost the entire QC set and commit the transaction. The evidence indicates that ratio/precision recalculation belongs to the normal subsequent QC finishing/posting workflow. The supplied documentation does not specify the application-level repost steps, so do not invent them.

### 11.9 Change QC batch expiration date

Find the QC batch ID:

```sql
SELECT DISTINCT qcbtc_qcbtc_id
FROM q_sets, q_qc_samples
WHERE set_set_id = set_id
  AND lab_number BETWEEN 127164 AND 127211;
```

Then update the batch:

```sql
UPDATE q_qc_batch
SET expiration_date = DATE '2024-12-28'
WHERE qcbtc_id = 59856;
```

Verify before committing.

### 11.10 Unassigning QCSMs

The maintenance documentation states that unassigning QCSMs is best performed by IHC supervisors using the form:

```text
QCSMUASGN.fmx
```

No authoritative SQL procedure for QCSM unassignment is provided in the supplied documentation. Do not substitute the ordinary LIMS `LBST_LBST_ID = 0` procedure without verifying the QC-specific model.

---

## 12. OIS -> LISA transfer

**Procedure ID:** `LISA-OIS-IN-001`  
**Evidence:** SOURCE-DOCUMENTED; environment details are ENVIRONMENT-DEPENDENT.

### 12.0 Inbound troubleshooting decision tree

```text
Analyst says OIS request is missing
        |
        v
Does Request_<sampling>.xml exist in the inbound archive?
   no --+--> investigate upstream/file transfer
        | yes
        v
Does LIMS_STAGE contain the sampling number?
   no --+--> investigate MapForce parsing/load and OIS_am.bat
        | yes
        v
Has the staged record been found and saved through the LISA login form?
   no --+--> investigate application/login-form workflow
        | yes
        v
Investigate the transactional LIMS record and downstream processing
```


If the LISA login form has no data to populate from OIS, a transfer may not have occurred because the database server was down or a nightly MapForce job failed.

The documented inbound process runs nightly at approximately **3:00 AM** using:

```text
D:/lims/shells/OIS_am.bat
```

The process:

1. Copies XML files to directories on the MapForce server.
2. Uses MapForce project `D:\mapforce\v19\labRequests\labRequest1_9_stage_p.mfd` to parse XML into the `SOURCE_RECEIVE` schema in LISA.
3. Runs `lims_stage.p_ois_load_lims_stage8` to load the staging tables.
4. Sends success email(s) as part of the process.

The maintenance slides contain inconsistent MapForce IP references (`10.150.39.15` in one inbound-transfer note and `10.158.39.15` elsewhere). Treat the current server address as an environment value to verify rather than an invariant.

---

## 13. Nightly LISA -> OIS transfers

**Procedure ID:** `LISA-OIS-OUT-001`  
**Evidence:** SOURCE-DOCUMENTED; environment details are ENVIRONMENT-DEPENDENT.

Outbound conceptual flow:

```text
Released in LISA
  -> originated from OIS / staging record exists
  -> ois_list.par parameter row generated
  -> response XML generated
  -> response XML placed where OIS can pick it up
```


The outbound process should include samples released that day that originated from OIS with a request XML.

### 13.1 Check whether a request was received

- Check `LIMS_STAGE.SAMPLE_SETS` for the sampling number.
- Check the MapForce server archive for `Request_<samplingnumber>.xml`.
- Documented archive path: `D:/ois_xchng/receive/OIS/archive`.

### 13.2 Parameter list generation

`ois_list.par` is generated nightly at approximately **9:30 PM** by procedure:

```text
lisarpt.OIS_XML_RELEASE
```

If the database or MapForce server is unavailable, the parameter file may not be generated and no samples will transfer. The documentation notes that server maintenance can interfere with this process.

### 13.3 Daily outbound verification query

```sql
SELECT DISTINCT
       flof_office_id,
       inspection_no,
       sampling_number,
       lbst_id
--   , to_char(released_on, 'DD-MON-YYYY HH24:MI'), labno_range
FROM l_sample_sets, l_samples, l_analysis_targets, l_lab_sets lbst
WHERE lbst_lbst_id = lbst.lbst_id
  AND smpl_smpl_id = smpl_id
  AND smst_smst_id = smst_id
  AND released_on >= TRUNC(SYSDATE)
--AND TRUNC(released_on) = '18-JUN-2024'
--AND TRUNC(released_on) BETWEEN '02-MAR-2024' AND '05-MAR-2024'
  AND flof_office_id BETWEEN 100000 AND 1099999
  AND sampling_number IN (
      SELECT sampling_number
      FROM lims_stage.sample_sets
      WHERE date_created > TRUNC(SYSDATE) - 90
  )
ORDER BY 3,2;
```

Choose the appropriate release-date predicate by commenting/uncommenting. The nested query limits OIS-originated samples to those received in the preceding 90 days; extend that interval if required.

### 13.4 Recover from failed outbound generation

The documented recovery procedure is:

1. Run the appropriate release query to regenerate the parameter rows.
2. Export/copy the result as text.
3. Paste it into `ois_list.par` on the MapForce server.
4. Save it in the documented response parameter directory (the source contains both `D:/ois_xchng/response/OIS/param` and `D:/ois_xchng/responses/OIS/param`; verify the actual path).
5. Execute:

```text
D:/lims/shells/new_ois_run_lab_response_no_gen_list.bat
```

### 13.5 Data missing from OIS

When field users report missing results, generate parameters using an appropriate identifier (inspection number, sampling number, or lab number):

```sql
SELECT DISTINCT
       flof_office_id,
       inspection_no,
       sampling_number,
       lbst_id
--   , sampling_date, recvd_in_lab, released_on
FROM l_sample_sets, l_samples, l_analysis_targets, l_lab_sets
WHERE smst_smst_id = smst_id
  AND smpl_smpl_id = smpl_id
  AND lbst_lbst_id = lbst_id
  AND result_posted_on > DATE '1900-01-01'
--AND lab_assigned_no BETWEEN 'F23499' AND 'F23524'
--AND sampling_number IN ('412400','413982','413983','413984')
  AND inspection_no IN ('1740661')
--AND TRUNC(released_on) BETWEEN '02-MAR-2024' AND '05-MAR-2024'
ORDER BY 2,3;
```

Then use the resulting rows to construct `ois_list.par` and run the no-generation-list batch file as above.

---

## 14. Analyst and Oracle account administration

A new analyst needs both an Oracle user account and an entry in the LISA analyst table to assign, edit, and review samples. Creating users requires Oracle `CREATE USER` privilege and may entail additional security-training requirements.

### 14.1 Oracle user creation example

The maintenance deck contains a historical user-creation pattern. The plaintext historical password and personal example have intentionally been removed from this authoritative guide. Verify current agency identity, password, role, and profile requirements before provisioning:

```sql
SET DEFINE OFF;
CREATE USER <USERNAME>
  IDENTIFIED BY <PROVISIONED_PASSWORD>
  DEFAULT TABLESPACE USERS
  TEMPORARY TABLESPACE TEMP
  PROFILE CHEMIST_ACCOUNTS
  ACCOUNT UNLOCK;

GRANT CONNECT TO <USERNAME>;
GRANT QC_USER TO <USERNAME>;
GRANT QC_USER2 TO <USERNAME>;
GRANT ANALYST2 TO <USERNAME>;
ALTER USER <USERNAME> DEFAULT ROLE ALL;
```

**Security note:** the roles/profile above are historical source material, not guaranteed current configuration. Use the organization's current credential provisioning and password policy and verify every role/profile before granting it.

### 14.2 Add the analyst to LISA

Historical example:

```sql
SET DEFINE OFF;
INSERT INTO LIMS.ANALYSTS
   (ANALYST_ID, PAY_GRADE, TITLE, LAST_NAME, FIRST_NAME,
    DEPT_DEPT_ID, HOME_PHONE, MIDDLE_INITIAL, DATE_HIRED, EXPERTISE,
    E_MAIL, EXTENSION, ANA_INITIALS_ID, INITIALS_ID, STATUS,
    LOGON_ID, END_DATE, BEGIN_DATE)
VALUES
   (l_ana_seq.nextval, '<PAY_GRADE>', 'CHEMIST - ANALYST', '<LAST_NAME>', '<FIRST_NAME>',
    9, NULL, NULL,
    TO_DATE('05/23/2022 00:00:00', 'MM/DD/YYYY HH24:MI:SS'),
    'CHEMISTRY', '<EMAIL>', NULL,
    '<SUPERVISOR_INITIALS>', '<UNIQUE_INITIALS>', NULL, '<LOGON_ID>', NULL, NULL);

COMMIT;
```

Operational rules documented:

- `INITIALS_ID` values must be unique; three initials are preferred to reduce collisions.
- `ANA_INITIALS_ID` represents the analyst's supervisor initials.
- `TITLE` must include the word `ANALYST` for the person to appear in LISA list-of-values popups.
- When someone leaves or should no longer use LISA as an analyst, remove `ANALYST` from the title.

Review available title values with:

```sql
SELECT DISTINCT title
FROM l_analysts
ORDER BY 1;
```


### 14.3 `LISA-USER-001` — Unlock an existing LISA Oracle account

**Purpose:** Restore login access when an existing LISA user's Oracle account has become locked.

**When to use:** A known LISA user reports that the account is locked and the requested action is to unlock the existing Oracle account.

**Do not use when:** The user's Oracle `LOGON_ID` has not been verified, the account does not exist, the reported problem has not been established as an account-lock condition, or the request requires a password reset or another credential change. Unlocking an account and resetting its password are separate administrative actions.

**Preferred mechanism:** Supported GUI administration may be used when available. For appropriately privileged DBA/support personnel, Oracle SQL can unlock the existing account directly.

**Required identifiers:** Verified Oracle username / LISA analyst `LOGON_ID`.

**Objects affected:** Oracle database user account. No LIMS transactional data is modified.

**Evidence / status:** `LIMS.ANALYSTS.LOGON_ID` is SOURCE-DOCUMENTED. `ALTER USER <USERNAME> ACCOUNT UNLOCK` is LIVE-VERIFIED 2026-09-01 for an existing LISA Oracle account. The optional `DBA_USERS` status queries below are standard Oracle administrative guidance but were **not live-verified in the documented LISA session**; use them only when the support account has authorized access to that view.

#### Pre-change identification

Do not infer an Oracle username from a person's name or email address. If the `LOGON_ID` is not already known and verified, retrieve it from the LISA analyst record:

```sql
SELECT analyst_id,
       last_name,
       first_name,
       e_mail,
       logon_id
FROM l_analysts
WHERE UPPER(last_name) = UPPER('<LAST_NAME>')
  AND UPPER(first_name) = UPPER('<FIRST_NAME>');
```

Confirm that the returned analyst is the intended user before continuing. If multiple rows are returned, use additional verified identifying information rather than guessing which account is correct.

> **Placeholder reminder:** `<USERNAME>`, `<LAST_NAME>`, and similar angle-bracketed text in this guide are placeholders. Replace the entire placeholder, including the angle brackets, with the verified value. Do not enter the `<` or `>` characters in executable SQL.

#### Optional pre-change account-status verification

If the DBA/support account has authorized access to Oracle account metadata, the following query can be used to inspect the account before modification. This query was not live-verified during the 2026-09-01 LISA session:

```sql
SELECT username,
       account_status,
       lock_date,
       expiry_date
FROM dba_users
WHERE username = UPPER('<USERNAME>');
```

Expected precondition: exactly one row for the intended Oracle account and a status consistent with the reported lock condition. If `DBA_USERS` is not accessible, do not treat the resulting privilege error as evidence that the user does not exist; use an authorized administrative mechanism instead.

#### Modification SQL

After verifying the Oracle username, unlock the account:

```sql
ALTER USER <USERNAME> ACCOUNT UNLOCK;
```

This statement form was successfully used to unlock an existing LISA Oracle account on 2026-09-01.

**Expected result:** Oracle accepts the account alteration for the verified user. Treat this as SQL/administrative execution success, not by itself as end-to-end LISA access success. When authorized, verify the resulting account status; ultimately, have the user confirm a normal LISA login with the credential they are expected to use. If Oracle reports an error, stop and investigate the actual error rather than assuming another password reset or unlock is required.

#### Verification

When authorized access to `DBA_USERS` is available, re-run the account-status query and confirm that the account is no longer locked. Because this metadata verification was not exercised during the documented 2026-09-01 session, treat it as useful verification guidance rather than a live-verified LISA step. The practical end-to-end verification is for the user to attempt a normal LISA login with the credential they are expected to use.

#### Password-reset distinction

`ACCOUNT UNLOCK` does not itself reset the user's password. If the password has already been reset and the subsequent problem is only an account lock, perform the narrowest requested operation: unlock the account without changing the password again. If authentication still fails after a successful unlock, obtain the exact resulting error before performing additional credential changes.

#### Commit point / rollback / recovery

`ALTER USER` is Oracle DDL and is not part of the transactional LIMS data-correction workflow described in Section 1.1. Do not rely on `ROLLBACK` to reverse the operation. Verify the resulting account state and/or user login. If an unintended account was altered, investigate and use the appropriate authorized account-management action rather than attempting a transactional rollback.

#### Known side effects

The live-verified operation changes the Oracle account lock state. No LIMS transactional records are modified. No additional LISA-specific side effects were established during the 2026-09-01 verification.

#### Placeholder troubleshooting note

If Oracle returns `ORA-01935: missing user or role name` while following an `ALTER USER` example, first verify that an angle-bracket placeholder was not entered literally. `<USERNAME>` means substitute the verified Oracle username; the `<` and `>` characters are not executable SQL syntax. This placeholder mistake was observed during the 2026-09-01 session.

**Date live-verified:** 2026-09-01.

---

## 15. Diagnostic and verification patterns

### 15.1 Prefer exact old-value guards

For administrative corrections, prefer:

```sql
UPDATE <table>
SET <column> = <new_value>
WHERE <primary_key> = <id>
  AND <column> = <expected_old_value>;
```

This converts an unexpected database state into `0 rows updated`, which is safer than overwriting silently.

### 15.2 Verify row counts

Treat any row count other than the expected count as a stop condition. Investigate before continuing or committing.

### 15.3 Inspect the whole set/group

When an application groups samples into a set, inspect all members before invoking set-level procedures:

```sql
SELECT ...
FROM ...
WHERE <set_fk> = <set_id>
ORDER BY ...;
```

This was essential in the QC workflow because `UPDATE_PRECS2` acts on the entire QC set.

### 15.4 Read stored code before executing unfamiliar procedures

A procedure name is not sufficient evidence of its side effects. Check source for updates, deletes, inserts, calls to other routines, and especially internal `COMMIT` statements.

---

## 16. Known documentation gaps and inconsistencies

The supplied documentation does **not** fully document every DBA operation. Important known gaps include:

- The Maintenance material contains a limited QC entity diagram, but not a complete current physical QC DSD comparable to DSD12. The QC relationships and column structures used operationally in this guide were therefore confirmed from live metadata/data on 2026-09-01.
- No application-level instructions were supplied for reposting/re-finishing a QC after DBA correction. `UPDATE_PRECS2` reveals database behavior but should not be treated as a substitute for the normal application workflow.
- The documentation says theoretical/ASM values can be edited in `SPIKED_SOLUTIONS` but does not name `THEORETICAL`; that column was verified live.
- ERD5 contains a legacy `SAMPLES.FLOW_CC` attribute that is not present in the later DSD12 physical `SAMPLES` structure. Treat `FLOW_CC` as historical/legacy unless live metadata proves otherwise.
- The shorter outbound-OIS slide says `lims.sample_sets`, while the longer slide and the actual later query use `lims_stage.sample_sets`; treat the shorter reference as an apparent source typo unless live verification shows otherwise.
- The documentation does not describe `FOUND_THEOR_RATIO` behavior after an ASM correction; this was established by live observation and stored-source inspection.
- The two maintenance presentations differ in some details and length. One includes additional material such as changing sampling-sheet primary-key information, analyst counting, and user creation.
- MapForce server IP/path references are inconsistent across the supplied slides. Verify current environment values before operational use.
- Historical examples may contain obsolete usernames, roles, passwords, dates, server addresses, or paths. Treat examples as structural guidance, not current credentials/configuration.

When a requested task is not covered here, use the schema-discovery method in Section 1.3, inspect existing data and stored code, make no assumptions about side effects, and document the verified procedure afterward.

---

## 17. Human/LLM execution protocol

When this guide is used by an LLM assisting a DBA, follow these rules:

1. **Never invent schema details.** If a table, column, synonym, key, procedure, or status meaning is not in this guide, discover it with read-only metadata queries.
2. **Work one change at a time when risk is nontrivial.** Present a `SELECT`, ask for/inspect its result, then propose the next statement.
3. **Separate observation from inference.** Say explicitly when a conclusion comes from documentation, live metadata, live data, or stored source.
4. **Do not commit early.** Keep a reversible transaction until all requested changes have been verified.
5. **Do not execute an unfamiliar procedure merely because its name sounds appropriate.** Read its source first and identify commits and set-wide side effects.
6. **Use exact identifiers and old-value predicates in corrective updates.**
7. **Protect parent/child integrity.** Discover dependent rows before deletes.
8. **Do not broaden a request.** An “unpost only” request means do not alter analytical/theoretical values.
9. **After success, capture the reusable procedure.** Add newly verified behavior to this guide with date/context and distinguish it from original documentation.
10. **For credentials and infrastructure, verify current policy/configuration.** Never rely on historical passwords or assume old IP addresses and paths remain valid.

---

## 18. Quick-reference task index

Stable procedure identifiers should be used in tickets/change logs where applicable.

| Task | Primary objects / mechanism |
|---|---|
| `LISA-SAMPLE-001` Find a sample | `L_SAMPLES`, `L_SAMPLE_SETS` |
| `LISA-ASSIGN-001` Inspect analytical assignment | `L_ANALYSIS_TARGETS`, `L_LAB_SETS`, `L_ANALYTES` |
| `LISA-ASSIGN-002` Assign sample/analyte to set | update `L_ANALYSIS_TARGETS.LBST_LBST_ID` |
| `LISA-ASSIGN-003` Unassign sample/analyte | set `LBST_LBST_ID = 0` |
| `LISA-DELETE-001` Delete sample | delete children, then `L_SAMPLES`, then parent rows when appropriate |
| Change analyte | update `L_ANALYSIS_TARGETS.ANLT_ANALYTE_ID` |
| Add missing analyte | insert `L_ANALYSIS_TARGETS` row |
| Change office/inspection/sampling number | update `L_SAMPLE_SETS` by `SMST_ID` |
| Count analyst work | aggregate `L_ANALYSIS_TARGETS` + `L_LAB_SETS` |
| Resolve substance unique constraint | inspect component analytes; remove confirmed duplicate targets |
| `LISA-QC-001` QC unpost | set-aware whole-set workflow: `Q_QC_SAMPLES.SAMPLE_STATUS: 0FQ -> 0AQ` plus `Q_SETS.RESULTS_DT` reset to the **live-verified** active sentinel after whole-set inspection; partial-set behavior is not yet verified |
| `LISA-QC-002` QC ASM/theoretical correction | `Q_SPIKED_SOLUTIONS.THEORETICAL` |
| QC batch expiration | `Q_QC_BATCH.EXPIRATION_DATE` |
| QCSM unassignment | `QCSMUASGN.fmx` (preferred documented method) |
| `LISA-OIS-IN-001` Inbound OIS troubleshooting | staging tables, MapForce inbound job |
| `LISA-OIS-OUT-001` Outbound OIS troubleshooting | `lisarpt.OIS_XML_RELEASE`, `ois_list.par`, response batch file |
| Add analyst | Oracle account + `LIMS.ANALYSTS` row |
| `LISA-USER-001` Unlock existing LISA account | verify `LIMS.ANALYSTS.LOGON_ID`; `ALTER USER ... ACCOUNT UNLOCK` |
| Discover unknown object | `ALL_OBJECTS`, `ALL_SYNONYMS`, `ALL_TAB_COLUMNS`, `ALL_SOURCE` |

---

## 19. Source provenance

This guide consolidates the following supplied materials:

- **LISA Maintenance** presentation, longer version (622 extracted lines).
- **LISA Maintenance** presentation, shorter/earlier version (503 extracted lines).
- **DSD12.pdf**, LIMS physical/data-structure diagram showing tables, columns, constraints, and relationships.
- **ERD5.pdf**, LIMS entity-relationship diagram (modified 2001) showing the conceptual relational model.
- **Verified DBA session, 2026-09-01:** QC 521028–521031 unpost/theoretical-value correction, including live synonym/table discovery and inspection of `QC_DEV2.UPDATE_PRECS2`.
- **Verified DBA session, 2026-09-01:** an existing LISA Oracle account was successfully unlocked using `ALTER USER ... ACCOUNT UNLOCK`.

### Verified 2026-09-01 QC findings retained as reusable knowledge

- `Q_SETS -> QC_DEV2.SETS`
- `Q_QC_SAMPLES -> QC_DEV2.QC_SAMPLES`
- `Q_SPIKED_SOLUTIONS -> QC_DEV2.SPIKED_SOLUTIONS`
- `QC_SAMPLES.QCSMPL_ID -> SPIKED_SOLUTIONS.QCSMPL_QCSMPL_ID`
- `0FQ -> 0AQ` is the sample-level portion of QC unpost/return-to-assigned state. Live verification on 2026-09-01 established that changing sample status alone left the previously finished set in a mixed state and did not restore application visibility. Subsequent investigation identified the retained `Q_SETS.RESULTS_DT` completion timestamp as inconsistent with the active `0AQ` comparison state, and it was reset to the live-verified sentinel `1900-01-01 00:00:00`. End-to-end necessity remains subject to application confirmation by the assigned analyst.
- ASM correction is a change to `SPIKED_SOLUTIONS.THEORETICAL`.
- Direct theoretical-value updates do not automatically refresh `FOUND_THEOR_RATIO` through a table trigger.
- `UPDATE_PRECS2` recalculates `FOUND_THEOR_RATIO`, marks the entire set `0FQ`, updates `RESULTS_DT`, recalculates precision, and commits internally; it is therefore not an appropriate helper for an unpost-only/correction transaction.
- QC unpost is set-aware in the live-verified 2026-09-01 whole-set case: inspect all set members, change requested finished samples to `0AQ`, and reset the previously finished set's `SETS.RESULTS_DT` to the live-verified active sentinel using a guarded update. Partial-set unpost behavior has not yet been verified and must not be inferred from this case.
- End-to-end application visibility after the `RESULTS_DT` correction was not established within the documented session unless separately confirmed by the assigned analyst. Until such confirmation is recorded, distinguish database-state verification from application/workflow verification.
- Oracle `DATE` values can contain a time component even when SQL Developer displays only the date; retrieve `TO_CHAR(..., 'YYYY-MM-DD HH24:MI:SS')` before constructing an exact old-value guard.

---

## 20. Change-control recommendation

Keep this document under version control. For every newly discovered DBA procedure, add:

- the date verified;
- the request/use case;
- read-only discovery queries;
- exact objects/relationships involved;
- safe update/delete/insert sequence;
- expected row counts;
- verification queries;
- transaction/commit behavior;
- application or stored-procedure side effects;
- rollback considerations;
- whether the procedure is **source-documented**, **live-verified**, or both.

That practice turns future one-off troubleshooting into durable LISA operational knowledge while keeping observed facts separate from assumptions.


## 21. Compact LIMS data dictionary and relationship reference

**Evidence:** primarily DSD12 physical schema; ERD5 is used for historical/conceptual semantics. Current live metadata wins if it differs. This appendix intentionally converts the dense diagrams into searchable text.

| Object | Primary identifier / key | Important relationships / columns | Operational purpose |
|---|---|---|---|
| `SAMPLE_SETS` | `SMST_ID` | office, inspection, sampling number, dates, CSHO, SIC, processing expedience | Sampling-sheet/header record |
| `SAMPLES` | `SMPL_ID` | `SMST_SMST_ID`, sample type, media, `LAB_ASSIGNED_NO`, `SUBMISSION_NO`, blank, flow, weight | Physical laboratory sample |
| `TIME_CHECKS` | `TMCK_ID` | `SMPL_SMPL_ID`, `TIME_ON`, `TIME_OFF` | Sample timing child rows |
| `SAMPLED_EMPLOYEES` | `EMP_ID` | `SMST_SMST_ID`, occupation | Employee sampled on sampling sheet |
| `ANALYSIS_TARGETS` | `ANLY_ID` | sample, analyte, method, instrument, lab set, qualifier, UOM; result A/B, detection limit, solution volume | Requested/result analyte row |
| `LAB_SETS` | `LBST_ID` | assigned analyst, method, instrument; assigned/posted/checked/released dates/users | Analytical work-set workflow |
| `ANALYTES` | `ANALYTE_ID` | `ANALYTE_CODE`, description, sampling rate, conversion factor, station | Analyte master |
| `ANALYTICAL_METHODS` | `ANMT_ID` | description | Analytical method master |
| `METHOD_ANALYTE` | `MTAN_ID` | analyte + method | Valid/associated analyte-method mapping |
| `INSTRUMENTS` | `INST_ID` | barcode, name, station | Instrument master |
| `COLLECTION_MEDIA` | `MEDIA_ID` / physical schema identifier | description, short description, used flag | Collection media master |
| `COLLECTION_MEDIA_SYNONYMS` | `CMSN_ID` | media synonym -> collection media | Alternate media naming |
| `MEDIA_ANALYTES` | `MDAN_ID` | collection media, analyte, supplier | Media/analyte compatibility/mapping |
| `MEDIA_SUPPLIERS` | `MDSP_ID` | supplier contact fields | Media supplier master |
| `SUBSTANCES` | `SUBS_ID` | substance analyte -> component analyte | Composite/substance decomposition |
| `UNITS_OF_MEASURE` | `ABBREVIATION` | description | Result/unit master |
| `QUALIFIER` | `TYPE` | description | Result qualifier master |
| `COMMENTS` | `CMMT_ID` | comment text/type/number | Reusable comments |
| `SET_COMMENTS` | `STCT_ID` | comment + lab set | Comments attached to analytical sets |
| `SMPL_COMMENTS` | `SMCT_ID` | analysis target + comment | Comments attached to sample/analysis work |
| `FIELD_OFFICES` | `OFFICE_ID` | region, scope, status, contact fields | OSHA office master |
| `COMPLIANCE_OFFICERS` | `CSHO_ID` | office, name/contact, begin/end dates | CSHO master |
| `OCCUPATIONS` | `JOB_CODE` | description | Occupation master |
| `PROTECTIVE_EQUIPMENTS` | `DEVICE_CODE` | description | PPE master |
| `TYPE_OF_PROTECTIONS` | `TYPR_ID` | employee + protective equipment | Employee/PPE relationship |
| `STATIONS` | `STTN_NUMBER` | name/comment | Laboratory station master |
| `STATION_ANALYTE` | `STAN_ID` | station + analyte | Station/analyte mapping |
| `ASSIGNMENTS` | `ASGN_ID` | month/year, analyst, station | Analyst/station assignment |
| `APPLICABLE_CALCULATIONS` | `APCL_ID` | analyte, exposure limit, UOM, value, posted date | Exposure/calculation support |
| `EXPOSURE_LIMITS` | `EXPL_ID` | short/long descriptions | Exposure limit master |
| `HISTORICAL_SAE` | `HSAE_ID` | analyte, method, media, instrument, date/value | Historical SAE values |
| `SIC_CODES` | `SIC_ID` | SIC code/description | Industry classification |
| `REGIONS` | `RGN_NUMBER` | name/phone | OSHA region master |
| `REGION_PERSONNEL` | `REGION_PERS_ID` | region, name, extension/title | Regional personnel |
| `PROCESSING_EXPEDIANCES` | `PROC_ID` | description | Processing expedience/priority support |
| `EXPOSURE_SUMMARY` | derived/reporting object | office, inspection, sampling, analyte, unit, exposure levels, release/severity fields | Exposure reporting/summary |
| `ASSIGNA` | view/reporting object | sample, inspection, analyst/analyte/station/media/priority fields | Assignment-oriented query/view |

### 21.1 Critical physical constraint families

DSD12 documents primary, unique, check, and foreign-key constraints. Examples include `ANLY_PK`, `ANLY_UK`, `ANLY_CK` for `ANALYSIS_TARGETS`, plus foreign keys to sample, analyte, method, instrument, lab set, qualifier, and UOM. `SAMPLES` similarly has `SMPL_PK`, `SMPL_UK`, `SAMPLES_CK`, and its foreign keys. When Oracle reports a named constraint, query `ALL_CONSTRAINTS` and `ALL_CONS_COLUMNS` rather than guessing its meaning.

### 21.2 LAB_SETS workflow fields

The supplied physical documentation shows workflow fields including `LBST_ID`, `ASSIGNED_TO`, `DATE_ASSIGNED`, `OPEN_COMMENT`, `ANMT_ANMT_ID`, `INST_INST_ID`, `RELEASED_BY`, `RELEASED_ON`, `RESULT_POSTED_BY`, `RESULT_POSTED_ON`, `CHECKED_BY`, `CHECKED_ON`, `CHECKERS_HOURS`, and `LOAD_COMMENTS`. Historical diagrams/screenshots may show additional legacy fields; use live `ALL_TAB_COLUMNS` for authoritative current nullability and column existence before writing DML.

---

## 22. Known Oracle errors and source defects

| Signature / issue | Meaning / action |
|---|---|
| `ORA-00001` on the analysis-target unique constraint during substance conversion | Match the situation to Section 10 before deleting anything; inspect actual constraint metadata and component analytes first. The inherited source associates this failure with `LIMS.CHANGE_SUBSTANCE_TO_IMIS`. |
| Short-deck assignment example uses a returned `SMPL_ID` as `LBST_LBST_ID` | **SOURCE-DEFECT. Do not use as written.** `LBST_LBST_ID` must reference a lab-set ID. |
| `FLOW_CC` appears in ERD5 but not DSD12 physical `SAMPLES` | Historical schema discrepancy; verify live metadata. |
| `lims.sample_sets` vs `lims_stage.sample_sets` in OIS slides | Apparent source inconsistency; later query and longer source use `LIMS_STAGE.SAMPLE_SETS`. Verify live environment. |
| MapForce IP/path variants | ENVIRONMENT-DEPENDENT; verify current server/path. |

---

## 23. Glossary

| Term | Meaning supported by source/context |
|---|---|
| **LISA / LIMS** | Laboratory Information System / database naming used by the OSHA laboratory application and schema artifacts. The inherited materials use both terms. |
| **OIS** | OSHA Information System; source/destination for electronic request/response integration in these materials. |
| **CSHO** | Compliance Safety and Health Officer; represented in the compliance-officer and sampling data. |
| **TWA** | Time-weighted average; `EIGHT_HR_TWA` appears on sampling-sheet data. |
| **QCSM** | QC sample terminology used by the maintenance material; QCSM unassignment is associated with `QCSMUASGN.fmx`. |
| **ASM** | Term used by the Maintenance slides for values corrected in `SPIKED_SOLUTIONS`; the source does not expand the acronym. Do not invent an expansion. |
| **SAE** | Term used in `HISTORICAL_SAE` and QC calculations; the supplied sources do not provide an authoritative expansion. |
| **IMIS** | Historical OSHA/legacy terminology used as an analyte-code alias and in `CHANGE_SUBSTANCE_TO_IMIS`; the supplied sources do not define the acronym, so this guide does not assert an expansion. |
| **91A** | Sampling-sheet/form terminology used by the Maintenance documentation; `SAMPLE_SETS` corresponds to its top section. |
| **MapForce** | Integration tooling/server used to parse OIS XML and generate response XML. |

---

## 24. Environment-dependent configuration and operations

These values are useful clues but **must be verified before operational use**.

| Item | Historical/source value | Verification note |
|---|---|---|
| Inbound job | `D:/lims/shells/OIS_am.bat` | Verify current server and path |
| MapForce project | `D:\mapforce\v19\labRequests\labRequest1_9_stage_p.mfd` | Verify version/path |
| MapForce server | conflicting `10.150.39.15` / `10.158.39.15` | Do not assume either is current |
| Inbound archive | `D:/ois_xchng/receive/OIS/archive` | Verify current path |
| Outbound parameter generation | about 9:30 PM; `lisarpt.OIS_XML_RELEASE` | Verify current schedule/job |
| Response parameter directory | both `response/OIS/param` and `responses/OIS/param` appear | Verify actual path |
| Response batch | `D:/lims/shells/new_ois_run_lab_response_no_gen_list.bat` | Verify current path |
| Maintenance window | source notes Friday evening around 21:00 Eastern can disrupt transfers | Historical operational note; verify current maintenance schedule |
| Success email recipients | inbound/outbound jobs historically send success email; recipient configuration is maintained in batch files | Personnel names in historical slides intentionally omitted; inspect current batch/configuration |
| Oracle roles/profile | `CONNECT`, `QC_USER`, `QC_USER2`, `ANALYST2`, `CHEMIST_ACCOUNTS` | Historical; verify current security policy |

---

## 25. Standard procedure template for future additions

Every new procedure added to this guide should use this structure:

```text
PROCEDURE ID / TASK
Purpose
When to use
Do not use when
Preferred mechanism (LISA Form / QC Form / DBA SQL / infrastructure)
Required identifiers
Objects affected
Evidence / status
Pre-change SELECT
Expected preconditions
Modification SQL or application steps
Expected row count
Verification SELECT
Commit point
Rollback / recovery
Known side effects
Related application form/job/procedure
Known source discrepancies
Date live-verified (if applicable)
```

For new SQL authored by this guide, prefer explicit `JOIN ... ON` syntax and ANSI date literals (`DATE 'YYYY-MM-DD'`) where practical. Historical SQL may be retained for provenance but should be labeled as historical when it uses older conventions.
