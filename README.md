# Data Sentinel for Snowflake

**Installation, Administration, Security, and User Guide**

| Document control | Value |
|---|---|
| Product | Data Sentinel |
| Provider | Infojini, Inc. |
| Document version | 1.0 |
| Product release | Initial marketplace release |
| Last updated | 17 September 2026 |
| Intended audience | Snowflake administrators, security engineers, data governance teams, data stewards, and application reviewers |
| Deployment model | Snowflake Native App with Streamlit in Snowflake |

> **Documentation status:** This guide is based on the supplied Data Sentinel application package structure, `setup.sql`, and `streamlit_app.py`. The package references `manifest.yml`, `environment.yml`, `README.md`, and `fix_procs.sql`; those files were not included in the documentation source set. Validate package-specific privileges, procedure signatures, support details, and uninstall behavior against the final release before marketplace publication.

---

## Table of contents

1. [Product overview](#1-product-overview)
2. [Key features](#2-key-features)
3. [Architecture](#3-architecture)
4. [Prerequisites](#4-prerequisites)
5. [Required Snowflake privileges](#5-required-snowflake-privileges)
6. [Installation](#6-installation)
7. [First-launch configuration](#7-first-launch-configuration)
8. [Database access grants](#8-database-access-grants)
9. [Running sensitive-data classification](#9-running-sensitive-data-classification)
10. [Reviewing classification results](#10-reviewing-classification-results)
11. [Assigning authorized roles](#11-assigning-authorized-roles)
12. [Applying bulk governance](#12-applying-bulk-governance)
13. [Applying column-level governance](#13-applying-column-level-governance)
14. [Managing and removing policies](#14-managing-and-removing-policies)
15. [Replacing masking policies](#15-replacing-masking-policies)
16. [Metadata tables](#16-metadata-tables)
17. [Security and privacy](#17-security-and-privacy)
18. [Limitations](#18-limitations)
19. [Troubleshooting](#19-troubleshooting)
20. [Support contact](#20-support-contact)
21. [Uninstallation](#21-uninstallation)

---

## 1. Product overview

Data Sentinel is a Snowflake Native App that helps organizations identify sensitive columns and implement role-aware data protection within Snowflake. The application presents a Streamlit user interface for classification, review, role assignment, masking-policy deployment, policy administration, and governance monitoring.

The principal workflow is:

```text
Select databases and tables
          ↓
Classify potentially sensitive columns
          ↓
Review PII, PHI, and PCI results
          ↓
Assign authorized Snowflake roles
          ↓
Create or reuse governance tags
          ↓
Create and apply masking policies
          ↓
Monitor and maintain implementation
```

Data Sentinel is intended to assist governance operations. It does not independently certify regulatory compliance, replace legal review, or remove the consumer's responsibility to validate classification and masking behavior.

### Intended users

- **Snowflake administrators:** Install the application, provision access, and manage infrastructure privileges.
- **Security and governance administrators:** Configure roles, tags, and masking controls.
- **Data stewards:** Review classifications and verify business context.
- **Platform operators:** Monitor implementation and troubleshoot failures.
- **Auditors and reviewers:** Inspect governance metadata and implementation status, subject to granted access.

### In-scope data classifications

The current user interface reports three high-level tag classes:

- **PII:** Personally Identifiable Information
- **PHI:** Protected Health Information
- **PCI:** Payment card or payment-related information

Classification output also records semantic and privacy categories returned by the backend classification procedure.

---

## 2. Key features

### Governance dashboard

- Counts sensitive and masked columns.
- Reports the number of governed tables.
- Summarizes PII, PHI, and PCI classifications.
- Displays implementation status and progress.
- Shows distribution by semantic category, privacy category, tag, and table.

### Sensitive-data classification

- Select a database, schema, and one or more tables.
- Run classification through the governance backend.
- Store results in a temporary preview table.
- Review results before implementing controls.

### Bulk governance

- Group results by classification tag.
- Assign one or more authorized Snowflake roles per tag.
- Reuse existing role assignments when an existing tag is detected.
- Load approved results into governance metadata.
- Create tags and apply tag-based masking policies.

### Attribute governance

- Select an individual database column.
- Assign authorized roles.
- Apply direct column-level masking.
- Infer a generalized masking-policy type of string, number, or date from Snowflake column metadata.

### Policy lifecycle management

- View implemented policies by governed table.
- Remove governance from a selected column.
- Remove a tag and associated policies through the Metadata Hub.
- Add roles by replacing existing masking-policy definitions while retaining tags.

### Metadata administration

- Filter records by database, schema, table, tag, semantic category, privacy category, status, or column name.
- View tag-to-role assignments.
- Delete selected metadata records or truncate metadata, subject to confirmation and privileges.

---

## 3. Architecture

### Logical architecture

```text
┌─────────────────────────────────────────────────────────────┐
│ Consumer Snowflake account                                  │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Installed Data Sentinel Native App                    │  │
│  │  core.datasentinel                                    │  │
│  │  Streamlit UI + Snowpark session                      │  │
│  └───────────────────────┬───────────────────────────────┘  │
│                          │ SQL / procedure calls             │
│  ┌───────────────────────▼───────────────────────────────┐  │
│  │ GOVERNANCE_DB.GOVERNANCE                             │  │
│  │  • MD_DATA_GOVERNANCE                                │  │
│  │  • MD_CLASSIFICATION_PREVIEW                         │  │
│  │  • SP_* governance procedures                        │  │
│  │  • tags and masking policies                         │  │
│  └───────────────────────┬───────────────────────────────┘  │
│                          │ granted visibility and control    │
│  ┌───────────────────────▼───────────────────────────────┐  │
│  │ Consumer-authorized databases, schemas, and tables    │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Application components

| Component | Purpose |
|---|---|
| `manifest.yml` | Declares the Native App, artifacts, and requested privileges. Validate against the release package. |
| `scripts/setup.sql` | Creates application role `app_user`, versioned schema `core`, and Streamlit object `core.datasentinel`. |
| `streamlit/streamlit_app.py` | Presents the dashboard and invokes Snowflake SQL and procedures. |
| `streamlit/fix_procs.sql` | Expected to define 11 governance procedures. Required for operational workflows. |
| `streamlit/environment.yml` | Declares Streamlit Python dependencies. |
| `GOVERNANCE_DB.GOVERNANCE` | Consumer-side governance schema used by the current implementation. |

### Runtime behavior

The application uses the Streamlit Snowflake connection and current Snowpark session. It checks that the metadata table is accessible and that at least 11 procedures whose names start with `SP_` exist in the governance schema. If the readiness check fails, the application opens the first-launch setup page.

### Data flow

1. Object discovery reads accessible Snowflake databases, schemas, tables, columns, and roles.
2. Classification executes against explicitly selected tables.
3. Results are written to `MD_CLASSIFICATION_PREVIEW`.
4. Approved results and role assignments are loaded into `MD_DATA_GOVERNANCE`.
5. Governance procedures create tags and masking policies and associate policies with selected objects.
6. The dashboard reads governance metadata to report implementation status.

---

## 4. Prerequisites

### Provider prerequisites

Before publishing:

- A Snowflake provider account capable of creating application packages and listings.
- A complete release bundle preserving the documented folder structure.
- A valid `manifest.yml` with only the required privileges and references.
- A tested application package version or patch.
- Public product documentation and a support process.
- Marketplace listing content, terms, availability, and access model.

### Consumer prerequisites

- A supported Snowflake account and permission to install Native Apps.
- A user authorized to accept requested application privileges.
- An administrative or dedicated governance role able to create governance objects and apply tags and masking policies.
- Access to each database, schema, and table selected for governance.
- Existing Snowflake roles that can be assigned access to unmasked values.
- Test data and non-production validation before applying controls broadly.

### Package completeness checklist

```text
manifest.yml
README.md
scripts/setup.sql
streamlit/streamlit_app.py
streamlit/fix_procs.sql
streamlit/environment.yml
```

Do not publish a version until every artifact is present and its path matches the manifest.

---

## 5. Required Snowflake privileges

> **Least-privilege requirement:** Use a dedicated governance owner role rather than a broad administrative role for normal operations whenever the final procedure design permits. `ACCOUNTADMIN` should be used only for narrowly scoped installation tasks that cannot be delegated.

### Application-level access

The supplied setup script creates:

```sql
CREATE APPLICATION ROLE IF NOT EXISTS app_user;
CREATE OR ALTER VERSIONED SCHEMA core;
GRANT USAGE ON SCHEMA core TO APPLICATION ROLE app_user;

CREATE OR REPLACE STREAMLIT core.datasentinel
  FROM '/streamlit'
  MAIN_FILE = '/streamlit_app.py';

GRANT USAGE ON STREAMLIT core.datasentinel
  TO APPLICATION ROLE app_user;
```

The consumer must grant the installed application's application role to appropriate account roles according to the final package instructions.

### Governance backend access

The current first-launch workflow expects the installed application to receive access comparable to:

```sql
GRANT USAGE ON DATABASE GOVERNANCE_DB
  TO APPLICATION DATASENTINEL_APP;
GRANT USAGE ON SCHEMA GOVERNANCE_DB.GOVERNANCE
  TO APPLICATION DATASENTINEL_APP;
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE
  ON ALL TABLES IN SCHEMA GOVERNANCE_DB.GOVERNANCE
  TO APPLICATION DATASENTINEL_APP;
GRANT USAGE ON ALL PROCEDURES IN SCHEMA GOVERNANCE_DB.GOVERNANCE
  TO APPLICATION DATASENTINEL_APP;
```

Replace `DATASENTINEL_APP` with the actual installed application name.

### Governed database access

For every in-scope database, the first-launch page generates access similar to:

```sql
GRANT USAGE ON DATABASE <DATABASE_NAME>
  TO APPLICATION DATASENTINEL_APP;
GRANT USAGE ON ALL SCHEMAS IN DATABASE <DATABASE_NAME>
  TO APPLICATION DATASENTINEL_APP;
GRANT SELECT ON ALL TABLES IN DATABASE <DATABASE_NAME>
  TO APPLICATION DATASENTINEL_APP;
```

These statements cover existing objects only. If the product must govern future schemas or tables, design and approve separate future grants in accordance with organizational policy and the final Native App security model.

### Administrative operations

The backend owner requires appropriate Snowflake privileges to perform the operations implemented by `fix_procs.sql`, which may include:

- Creating and using the governance database and schema.
- Creating tables, stored procedures, tags, and masking policies.
- Applying or removing tags and masking policies.
- Reading selected table metadata and data needed for classification.
- Executing governance procedures.

The exact privilege list must be derived from the final `manifest.yml` and `fix_procs.sql` during security review.

---

## 6. Installation

### Provider packaging workflow

1. Create an application package, for example `DATASENTINEL_PKG`.
2. Upload all release files to the package stage while preserving `scripts/` and `streamlit/` directories.
3. Add a version or patch that points to the staged artifacts.
4. Validate the version in a clean test account.
5. Attach the application package to the marketplace listing.
6. Complete listing review and publish through the approved access model.

### Consumer installation workflow

1. Open the Data Sentinel listing in Snowflake Marketplace.
2. Review the provider, requested privileges, terms, documentation, and support policy.
3. Install the application using an approved application name, for example `DATASENTINEL_APP`.
4. Accept only reviewed privilege requests.
5. Grant `app_user` to approved account roles according to the release instructions.
6. Open the Data Sentinel Streamlit application.
7. Complete first-launch setup before using governance features.

### Installation validation

After installation, validate:

- The Streamlit UI opens.
- The installed application role is granted only to intended operators.
- The first-launch page appears if the backend is absent.
- The governance database and schema are not created unexpectedly under an inappropriate owner.
- Audit history captures installation and setup actions.

---

## 7. First-launch configuration

On startup, Data Sentinel checks whether the governance metadata table is accessible and whether at least 11 `SP_` procedures exist. If not, the application displays **DataSentinel - First Launch Setup**.

### Step 1: Create the governance backend

1. Open a Snowflake SQL worksheet.
2. Use the approved installation or governance-owner role.
3. Run the SQL generated in **Step 1: Create the Governance Backend**.
4. Confirm creation of:
   - `GOVERNANCE_DB`
   - `GOVERNANCE_DB.GOVERNANCE`
   - `MD_DATA_GOVERNANCE`
   - `MD_CLASSIFICATION_PREVIEW`
   - All required governance procedures from `fix_procs.sql`
5. Review procedure ownership and execution context.

### Step 2: Grant controlled access

1. Enter the database names to be governed.
2. Copy the generated grant statements.
3. Review every database and privilege before execution.
4. Run the approved grants using an authorized role.
5. Avoid granting access to databases outside the intended scope.

### Step 3: Verify

Select **I've completed setup - Verify & Continue**. If validation succeeds, the application reloads. **Skip for now** bypasses readiness for the current session but does not create the backend.

### Post-setup validation

```sql
SHOW DATABASES LIKE 'GOVERNANCE_DB';
SHOW SCHEMAS IN DATABASE GOVERNANCE_DB;
SHOW TABLES IN SCHEMA GOVERNANCE_DB.GOVERNANCE;
SHOW PROCEDURES IN SCHEMA GOVERNANCE_DB.GOVERNANCE;
```

Confirm exactly the expected procedures, not merely the number of procedures.

---

## 8. Database access grants

### Granting a database

Use the setup-generated SQL after replacing placeholders and confirming the installed application name:

```sql
USE ROLE <APPROVED_ADMIN_ROLE>;

GRANT USAGE ON DATABASE <DATABASE_NAME>
  TO APPLICATION DATASENTINEL_APP;
GRANT USAGE ON ALL SCHEMAS IN DATABASE <DATABASE_NAME>
  TO APPLICATION DATASENTINEL_APP;
GRANT SELECT ON ALL TABLES IN DATABASE <DATABASE_NAME>
  TO APPLICATION DATASENTINEL_APP;
```

### Recommended grant process

1. Maintain an approved inventory of in-scope databases.
2. Start with a non-production database.
3. Grant the minimum visibility required for classification.
4. Validate that unauthorized databases do not appear in selectors.
5. Record grant approval and execution in the change-management system.
6. Reassess access after schema changes, ownership changes, or application upgrades.

### Revoking access

```sql
REVOKE SELECT ON ALL TABLES IN DATABASE <DATABASE_NAME>
  FROM APPLICATION DATASENTINEL_APP;
REVOKE USAGE ON ALL SCHEMAS IN DATABASE <DATABASE_NAME>
  FROM APPLICATION DATASENTINEL_APP;
REVOKE USAGE ON DATABASE <DATABASE_NAME>
  FROM APPLICATION DATASENTINEL_APP;
```

Before revoking access, remove or transfer governance controls according to organizational policy. Revoking application visibility does not necessarily remove tags or policies already applied to data objects.

---

## 9. Running sensitive-data classification

1. Open **Bulk Governance**.
2. Select a database.
3. Select a schema.
4. Select one or more tables.
5. Select **Run Classification**.
6. Wait for each selected table to complete.

For each table, Data Sentinel calls:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_CLASSIFY_AND_PREVIEW(
  '<DATABASE>.<SCHEMA>.<TABLE>'
);
```

### Operational guidance

- Begin with representative non-production tables.
- Avoid running broad classification during peak workloads without performance validation.
- Confirm the selected tables are within the approved governance scope.
- Treat classification as a recommendation requiring review.
- Re-run classification after material schema or data-profile changes.

### Expected outcome

Successful classification populates `MD_CLASSIFICATION_PREVIEW` and advances the interface to the review stage. Failures are displayed by table.

---

## 10. Reviewing classification results

Results are grouped into tabs by `TAG_NAME`, such as PII, PHI, or PCI. Each row can include:

- Table name
- Column name
- Semantic category
- Privacy category
- Data type

### Review checklist

For every proposed classification:

- Confirm the column belongs to the expected table.
- Verify the classification fits the business meaning.
- Check false positives and false negatives.
- Confirm the proposed data type is compatible with masking behavior.
- Identify columns requiring a different control or exception.
- Obtain data-owner approval when required.

Do not implement governance until the preview is acceptable. The current UI does not provide an inline editor for individual preview rows, so corrections may require adjusting the backend process or preview data under an approved procedure.

---

## 11. Assigning authorized roles

After review, Data Sentinel displays each classification group and an **Allowed Roles** selector.

### Procedure

1. Review existing role assignments shown for each tag.
2. If no assignment exists, select at least one authorized Snowflake role.
3. Ensure service roles, break-glass roles, and administrative roles are handled according to policy.
4. Confirm each classification group has an assignment.
5. Record the business owner and approval source.

### Access-design guidance

- Assign roles based on business purpose, not individual users.
- Prefer dedicated functional roles over broad administrative roles.
- Use role hierarchy deliberately and test inherited access.
- Avoid using `PUBLIC` as an authorized unmasked role.
- Review assignments periodically and after organizational changes.
- Test both an authorized and unauthorized role before production rollout.

The current metadata model stores comma-separated role names in `ROLE_NAME`. Changes to role names or hierarchy should trigger a governance review.

---

## 12. Applying bulk governance

Select **Implement Governance** after every classification group has at least one role.

### Processing sequence

For each selected table, the application performs:

1. Load approved preview records and role assignments:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_LOAD_GOVERNANCE_FROM_PREVIEW(
  '<DATABASE>.<SCHEMA>.<TABLE>',
  '<TAG>',
  '<COMMA_SEPARATED_ROLES>',
  'GOVERNANCE_DB.GOVERNANCE.MD_DATA_GOVERNANCE',
  'Yes'
);
```

2. Create required tags:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_CREATE_TAGS(
  'GOVERNANCE_DB.GOVERNANCE.MD_DATA_GOVERNANCE',
  'GOVERNANCE_DB',
  'GOVERNANCE'
);
```

3. Apply tag-based masking policies:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_APPLY_MASKING_POLICY_BY_TAG(
  'GOVERNANCE_DB.GOVERNANCE.MD_DATA_GOVERNANCE',
  'GOVERNANCE_DB',
  'GOVERNANCE',
  'MP'
);
```

### Validation

- Review the execution summary and table status.
- Confirm metadata records show `IS_IMPLEMENTED = 1` where appropriate.
- Query governed data using authorized and unauthorized test roles.
- Verify masking output for string, number, and date columns.
- Review Snowflake query and access history.
- Investigate partial failures before rerunning.

The UI executes multiple steps sequentially. Do not assume automatic rollback unless the final stored procedures explicitly provide transactional or compensating behavior.

---

## 13. Applying column-level governance

Use **Attribute Governance** for a single column.

### Procedure

1. Select database, schema, table, and column.
2. Select one or more roles authorized to view unmasked values.
3. Optionally enter a tag name.
4. Select **Apply Governance**.
5. Validate behavior using authorized and unauthorized roles.

The application maps discovered types into generalized groups:

- Numeric Snowflake types → `NUMBER`
- Date, time, or timestamp types → `DATE`
- Other types → `STRING`

### Important current behavior

The current implementation sets `cg_tag_flag = "No"` without a user control. Consequently, the direct-masking procedure is used:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_APPLY_MASKING(
  'GOVERNANCE_DB.GOVERNANCE.MD_DATA_GOVERNANCE',
  'GOVERNANCE_DB',
  'GOVERNANCE'
);
```

The optional tag-name field does not make this workflow tag-based in the supplied release. Validate and correct this behavior before advertising tag-based attribute governance.

### Duplicate prevention

Before repeated submissions, verify whether a metadata record already exists for the selected column. The supplied UI inserts a new metadata row directly and does not visibly enforce uniqueness.

---

## 14. Managing and removing policies

### Viewing a governed table

1. Open **Policy Management**.
2. Select database and schema.
3. Select a table from **Table (Governed Only)**.
4. Review governed columns, tags, semantic categories, privacy categories, data types, roles, and creation timestamps.

### Removing governance from one column

1. Select a governed column.
2. Select the confirmation checkbox.
3. Select **Remove Policy**.

The UI calls:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_REMOVE_TAGS_FROM_COLUMN(
  '<DATABASE>', '<SCHEMA>', '<TABLE>', '<COLUMN>'
);
```

Because the implementation resides in `fix_procs.sql`, confirm whether this action removes the tag, masking-policy association, metadata row, or all three.

### Removing a tag and its policies

In **Metadata Hub**, expand **Remove Tag & Policies**, select a tag, review the warning, confirm, and execute. The UI calls:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_REMOVE_TAG_AND_POLICIES('<TAG>');
```

This is intended as a global destructive action. Require change approval, impact analysis, and a tested recovery plan.

### Metadata cleanup warning

Deleting rows or truncating `MD_DATA_GOVERNANCE` may not remove tags or policies from Snowflake objects. Use metadata cleanup only when the actual object state has been reconciled.

---

## 15. Replacing masking policies

Use **Policy Replacement** to add roles to existing implemented policies.

### Procedure

1. Review current tag-based and attribute-governed policy summaries.
2. Select one or more tags.
3. Review current roles.
4. Select additional roles.
5. Review the merged role preview.
6. Confirm the replacement.
7. Select **Replace Policies**.

The application calls:

```sql
CALL GOVERNANCE_DB.GOVERNANCE.SP_REPLACE_MASKING_POLICY(
  'GOVERNANCE_DB.GOVERNANCE.MD_DATA_GOVERNANCE',
  'GOVERNANCE_DB',
  'GOVERNANCE',
  'MP',
  '<NEW_COMMA_SEPARATED_ROLES>',
  '<COMMA_SEPARATED_TAGS>'
);
```

### Post-change controls

- Verify all existing authorized roles remain present.
- Verify new roles receive intended access.
- Verify unauthorized roles remain masked.
- Confirm tags remained attached.
- Check that no column was left without an applicable policy.
- Record the change and approval.

Test replacement in non-production first. The exact atomicity and rollback behavior depend on the stored procedure implementation.

---

## 16. Metadata tables

### `MD_DATA_GOVERNANCE`

The durable governance registry expected by the dashboard.

| Column | Purpose |
|---|---|
| `TABLE_CATALOG` | Governed database name |
| `TABLE_SCHEMA` | Governed schema name |
| `TABLE_NAME` | Governed table name |
| `TABLE_COLUMN` | Governed column name |
| `TAG_NAME` | Classification or governance tag |
| `SEMANTIC_CATEGORY` | Classification semantic category |
| `PRIVACY_CATEGORY` | Classification privacy category |
| `DATA_TYPE` | Generalized or discovered data type |
| `ROLE_NAME` | Comma-separated authorized roles |
| `TAG_FLAG` | Indicates tag-based (`Yes`) or direct (`No`) governance |
| `IS_IMPLEMENTED` | `1` for implemented and `0` for pending |
| `POLICY_CREATION_TIMESTAMP` | Policy or metadata creation time |

### `MD_CLASSIFICATION_PREVIEW`

A staging table for classification review.

| Column | Purpose |
|---|---|
| `TABLE_CATALOG` | Classified database name |
| `TABLE_SCHEMA` | Classified schema name |
| `TABLE_NAME` | Classified table name |
| `TABLE_COLUMN` | Classified column name |
| `TAG_NAME` | Proposed tag such as PII, PHI, or PCI |
| `SEMANTIC_CATEGORY` | Proposed semantic category |
| `PRIVACY_CATEGORY` | Proposed privacy category |
| `DATA_TYPE` | Classified data type |
| `CLASSIFIED_AT` | Classification timestamp |

### Data-management recommendations

- Restrict metadata tables to approved administrators and the application.
- Define retention and cleanup rules for preview records.
- Reconcile metadata with actual Snowflake tags and policies.
- Protect role-assignment metadata as security configuration.
- Consider uniqueness controls for database, schema, table, and column combinations.
- Back up configuration through approved Snowflake mechanisms before destructive changes.

---

## 17. Security and privacy

### Security model

Data Sentinel operates inside the consumer's Snowflake environment and uses consumer-granted access. The application should be evaluated under shared responsibility: the provider secures application logic and package updates, while the consumer controls installation, granted privileges, roles, governed databases, classification approval, and operational monitoring.

### Security recommendations

- Apply least privilege to application roles and procedure owners.
- Review `manifest.yml` and every requested privilege before installation.
- Inspect `fix_procs.sql` for dynamic SQL, owner-rights execution, and input validation.
- Validate object identifiers and escape string values.
- Avoid broad `ACCOUNTADMIN` ownership for ongoing runtime operations.
- Separate installers, operators, approvers, and auditors where practical.
- Enable change management for tag and policy removal.
- Review Snowflake query, access, and login history according to policy.
- Test upgrades and patches before production deployment.
- Rotate or remove obsolete role assignments.

### Data privacy

The supplied UI does not show an external data-transfer integration. Nevertheless, consumers must review the final package for external access integrations, secrets, telemetry, logging, and support diagnostics. Do not enter personal or confidential values into free-text fields such as tag names.

### Compliance statement

Data Sentinel can support technical control implementation, but installation or use does not itself establish GDPR, HIPAA, PCI DSS, SOC 2, ISO 27001, or other compliance. Compliance depends on organizational policy, architecture, operation, evidence, legal requirements, and independent assessment.

---

## 18. Limitations

1. The documentation source set did not include `manifest.yml`, `fix_procs.sql`, `environment.yml`, or the package README.
2. Exact classification logic and masking expressions cannot be verified without `fix_procs.sql`.
3. The first-launch readiness check counts `SP_` procedures but does not verify exact names, signatures, ownership, or behavior.
4. Several SQL statements are assembled through string interpolation. Special characters and quoted identifiers require careful validation.
5. Procedure-creation errors may be suppressed during the application's automatic backend attempt.
6. Bulk governance performs sequential steps, and rollback behavior is not visible in the UI code.
7. Attribute Governance currently forces direct masking and does not activate the tag-based branch.
8. Metadata cleanup can diverge from actual Snowflake object state.
9. Grants shown by the setup page apply to existing schemas and tables; future objects require a separate approved strategy.
10. Role assignments are stored as comma-separated values rather than a normalized relationship.
11. The UI does not visibly provide row-level editing of classification preview results.
12. Object selectors depend on the installed application's visibility and may include system or irrelevant objects unless privileges are tightly scoped.
13. Product certification and regulatory compliance are not established by the application files.

---

## 19. Troubleshooting

### Backend not detected

**Symptoms:** The first-launch setup page continues to appear.

**Actions:**

```sql
SHOW TABLES IN SCHEMA GOVERNANCE_DB.GOVERNANCE;
SHOW PROCEDURES IN SCHEMA GOVERNANCE_DB.GOVERNANCE;
```

- Confirm `MD_DATA_GOVERNANCE` exists and is accessible.
- Confirm all required procedures were created successfully.
- Confirm procedure names begin with `SP_` where expected.
- Review grants to the actual installed application name.
- Do not rely solely on procedure count.

### Database, schema, or table does not appear

- Confirm `USAGE` on the database and schema.
- Confirm `SELECT` on the table.
- Confirm grants target the installed application rather than the package name.
- Use **Refresh Data** or wait for the selector cache to expire.
- Check object name case and quoted identifiers.

### Classification fails

- Verify the selected table exists and is accessible.
- Confirm `SP_CLASSIFY_AND_PREVIEW` exists and is executable.
- Review procedure logs and Snowflake query history.
- Check unsupported data types or malformed dynamic SQL.
- Test one table at a time to isolate the failure.

### Preview is empty

- Verify classification completed successfully.
- Query `MD_CLASSIFICATION_PREVIEW` for the selected database, schema, and tables.
- Confirm names and case match the filter used by the application.
- Determine whether the table contains detectable sensitive patterns.

### Implement Governance is disabled

Assign at least one role to every classification group. Existing assignments are reused only when the application detects both an existing tag and non-empty role metadata.

### Masking policy application fails

- Review tag and policy creation privileges.
- Confirm the procedure owner is authorized for the target objects.
- Verify policy data-type signatures match target columns.
- Check for an existing incompatible policy.
- Inspect earlier metadata or tag-creation failures.

### Authorized role still sees masked data

- Verify the active primary and secondary roles.
- Check whether the policy tests the expected role semantics.
- Confirm the role name stored in metadata exactly matches Snowflake.
- Review role hierarchy and policy definition.
- Retest in a new session after changes.

### Unauthorized role sees clear data

Treat this as a security incident:

1. Suspend further rollout.
2. Record the role, query, object, and timestamp.
3. Review the masking policy attached to the tag or column.
4. Review role hierarchy and policy conditions.
5. Apply a corrective policy using the approved emergency process.
6. Notify the security and data owner teams.

### Metadata and actual policies disagree

- Inventory tags and masking policies directly in Snowflake.
- Compare attachments with `MD_DATA_GOVERNANCE`.
- Do not delete more metadata until reconciliation is complete.
- Restore or correct records only through an approved procedure.

### Collecting diagnostic information

When opening a support case, include:

- Data Sentinel application version and patch
- Installed application name
- Snowflake region and account locator, if approved for sharing
- Timestamp and query ID
- Page and operation performed
- Sanitized error message
- Affected object names, redacted where required
- Whether the issue reproduces in non-production

Never send table data, credentials, private keys, session tokens, or secrets in a support ticket.

---

## 20. Support contact

> **Provider action required:** Replace the placeholders below before marketplace publication.

| Support channel | Required value |
|---|---|
| Support organization | Infojini, Inc. |
| Support email | `REPLACE_WITH_SUPPORT_EMAIL` |
| Support portal | `REPLACE_WITH_SUPPORT_PORTAL_URL` |
| Documentation URL | `REPLACE_WITH_PUBLIC_DOCUMENTATION_URL` |
| Support hours | `REPLACE_WITH_SUPPORT_HOURS_AND_TIME_ZONE` |
| Initial response target | `REPLACE_WITH_SERVICE_LEVEL_TARGET` |
| Security contact | `REPLACE_WITH_SECURITY_EMAIL` |

### Suggested severity model

- **Severity 1:** Confirmed exposure of unmasked sensitive data, widespread production outage, or destructive governance failure.
- **Severity 2:** Major feature unavailable with no reasonable workaround.
- **Severity 3:** Limited functional issue with a workaround.
- **Severity 4:** Documentation question, enhancement request, or general guidance.

Support commitments must match the marketplace listing and contractual terms.

---

## 21. Uninstallation

> **Warning:** Dropping the installed Native App may not remove governance objects created outside the application, including tags, masking policies, metadata tables, or controls attached to consumer tables. Perform reconciliation first.

### Pre-uninstallation checklist

1. Stop new classification and governance changes.
2. Inventory governed databases, tables, columns, tags, and policies.
3. Export approved configuration metadata if retention is required.
4. Decide whether masking must remain after uninstallation.
5. Remove or transfer policies through approved procedures.
6. Validate governed data using authorized and unauthorized roles.
7. Revoke the application's database access.
8. Capture audit evidence and change approval.

### Revoke governed-database access

```sql
REVOKE SELECT ON ALL TABLES IN DATABASE <DATABASE_NAME>
  FROM APPLICATION DATASENTINEL_APP;
REVOKE USAGE ON ALL SCHEMAS IN DATABASE <DATABASE_NAME>
  FROM APPLICATION DATASENTINEL_APP;
REVOKE USAGE ON DATABASE <DATABASE_NAME>
  FROM APPLICATION DATASENTINEL_APP;
```

### Remove the installed application

After external objects and dependencies have been reviewed:

```sql
DROP APPLICATION IF EXISTS DATASENTINEL_APP;
```

### Optional backend removal

Only if governance policies have been safely removed or transferred and retention requirements permit deletion:

```sql
DROP DATABASE IF EXISTS GOVERNANCE_DB;
```

Do not run this command merely to uninstall the UI. Review all objects in `GOVERNANCE_DB`, ownership, dependencies, retention, and recovery requirements first.

### Post-uninstallation validation

- Confirm the application no longer appears.
- Confirm application grants are removed.
- Confirm required masking controls remain or were intentionally removed.
- Confirm no orphaned tag-policy associations remain.
- Confirm governance metadata was retained or deleted according to policy.
- Close the change record with validation evidence.

---

## Appendix A: Release-readiness checklist

- [ ] All six package files are present.
- [ ] `manifest.yml` requests only necessary privileges.
- [ ] All 11 procedures have been reviewed and tested.
- [ ] Dynamic SQL inputs are validated and safely quoted.
- [ ] Setup uses a dedicated least-privilege governance role where possible.
- [ ] Trial or paid-access behavior is implemented as advertised.
- [ ] Logging contains no sensitive table values.
- [ ] Install, upgrade, rollback, and uninstall tests pass.
- [ ] Authorized and unauthorized role tests pass for each masking type.
- [ ] Metadata reconciliation procedures are documented.
- [ ] Support and security contact placeholders are replaced.
- [ ] Public documentation URL is accessible without authentication.
- [ ] Compliance claims have documentary evidence or are removed.

## Appendix B: Change history

| Version | Date | Description |
|---|---|---|
| 1.0 | 17 September 2026 | Initial marketplace documentation draft based on supplied application files. |
