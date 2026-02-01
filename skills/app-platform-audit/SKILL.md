---
name: app-platform-audit
description: Audit existing Gladly App Platform apps for best practices compliance. Use this skill to validate an existing app against coding standards, safety patterns, test coverage, and documentation requirements. Generates a compliance report with findings and recommendations.
---

# App Platform Audit

Analyze an existing Gladly App Platform app for best practices compliance and generate a detailed audit report.

## Prerequisites

- Existing App Platform app directory with `manifest.json`
- `appcfg` CLI installed: https://github.com/gladly/app-platform-appcfg-cli/releases

## Audit Workflow

```
STEP 1: VALIDATE     → Run appcfg validate
STEP 2: STRUCTURE    → Check directory structure
STEP 3: AUTH         → Audit authentication setup
STEP 4: SCHEMAS      → Audit GraphQL schemas
STEP 5: DATA PULLS   → Audit templates and tests
STEP 6: ACTIONS      → Audit error handling
STEP 7: UI           → Audit Flexible Cards
STEP 8: REPORT       → Generate findings report
```

---

## Step 1: VALIDATE - Run CLI Validation

**Goal**: Ensure app passes basic CLI validation.

```bash
appcfg validate --root .
```

**Record result**: PASS or FAIL with error messages.

If validation fails, note errors but continue audit to identify all issues.

---

## Step 2: STRUCTURE - Check Directory Structure

**Goal**: Verify required files and directories exist.

```bash
ls manifest.json
ls data/data_schema.graphql
ls -d authentication/
ls data/pull/
ls ui/templates/
```

### Required Structure

| Path | Required | Purpose |
|------|----------|---------|
| `manifest.json` | Yes | App metadata |
| `data/data_schema.graphql` | Yes | Data type definitions |
| `authentication/` | Yes | Auth configuration |
| `data/pull/` | Yes | At least one data pull |
| `ui/templates/` | Yes | At least one UI template |
| `actions/actions_schema.graphql` | If actions | Mutation definitions |

**Record findings**: List any missing required files.

---

## Step 3: AUTH - Audit Authentication Setup

**Goal**: Verify authentication is properly configured.

### Authentication Structure

```bash
ls -la authentication/
ls authentication/headers/
ls -d authentication/oauth/ 2>/dev/null
ls -d authentication/request_signing/ 2>/dev/null
```

### Auth Method Checks

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Auth headers exist | `authentication/headers/*.gtpl` | Critical |
| Secrets usage | Headers use `.integration.secrets.*` for tokens | Critical |
| Config vs secrets | URLs in `.configuration`, tokens in `.secrets` | High |

### OAuth Checks (if oauth/ exists)

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Grant type | `oauth/config.json` has `grantType` field | Critical |
| Token URL | `access_token/request_url.gtpl` exists | Critical |
| Token body | `access_token/request_body.gtpl` exists | Critical |
| Auth code flow | If `grantType: "authorization_code"`, check `auth_code/` exists | Critical |
| Refresh token | If auth_code, check `refresh_token/` exists | High |
| State param | auth_code URL uses `{{.correlationId}}` for state | High |
| URL encoding | All body params use `{{urlquery ...}}` | High |

### Request Signing Checks (if request_signing/ exists)

| Check | What to Look For | Severity |
|-------|------------------|----------|
| Signing key | Uses `.integration.secrets.*` for key | Critical |
| Timestamp | Includes timestamp in signature payload | High |

**Record findings**: List authentication issues found.

---

## Step 4: SCHEMAS - Audit GraphQL Schemas

**Goal**: Verify schemas follow required patterns.

### Data Schema Checks

Read `data/data_schema.graphql` and verify:

| Check | What to Look For | Severity |
|-------|------------------|----------|
| @dataType directive | All pullable types have `@dataType(name: "prefix_name", version: "X.Y")` | Critical |
| Type naming | Data type names use snake_case with prefix (e.g., `acme_customer`) | High |
| Type ordering | Types defined before they are referenced (no forward declarations) | Critical |
| ID field | Root types have id: ID! | High |
| DateTime fields | Fields ending in `At`, `Date`, `Time` use `DateTime` type | Medium |
| Currency fields | Price/amount fields use `Currency` type | Medium |
| Relationships | `@parentId` or `@childIds` directive on child collections | High |

### Actions Schema Checks (if exists)

| Check | What to Look For | Severity |
|-------|------------------|----------|
| @action directive | All mutations have `@action(name: "action_name")` | Critical |
| Return type | Mutations return type with status: String! | High |
| Required inputs | Required inputs marked with ! | Medium |

**Record findings**: List schema issues found.

---

## Step 5: DATA PULLS - Audit Templates and Tests

**Goal**: Verify data pulls use safety patterns and have test coverage.

For each directory in `data/pull/`:

### Safety Pattern Checks

Read `request_url.gtpl` and search for these patterns:

| Pattern | What to Find | Severity |
|---------|--------------|----------|
| Null check | `{{- if not $variable}}{{- stop` | Critical |
| Empty array check | `{{- if eq (len $variable) 0}}{{- stop` | Critical |
| URL encoding | `{{urlquery $variable}}` for path/query params | High |
| Stop on error | `{{- stop "descriptive message"}}` | Critical |

**Critical patterns for dependent pulls** (those with `dependsOnDataTypes` in config.json):

```go
// MUST have all three checks:
{{- if not $parentData}}{{- stop "No parent data"}}{{- end}}
{{- if eq (len $parentData) 0}}{{- stop "Parent not found"}}{{- end}}
{{- if not $parentId}}{{- stop "Parent ID empty"}}{{- end}}
```

Read `response_transformation.gtpl` (if exists) and check:

| Pattern | What to Find | Severity |
|---------|--------------|----------|
| Null rawData | `{{- if not .rawData}}` at start | Critical |
| Array extraction | `{{- if kindIs "slice" .rawData}}` | High |
| Empty array handling | Returns `[]` for empty results | Critical |
| Proper null output | `null` not `""` for missing fields | Medium |
| String escaping | `{{$value \| toJson}}` for user content | Medium |

### Test Coverage Checks

```bash
ls data/pull/<name>/_test_/
```

| Check | Requirement | Severity |
|-------|-------------|----------|
| Default test | `_test_/default/` exists | High |
| integration.json | Test config present | High |
| rawData.json | Sample response present | High |
| Expected outputs | expected_*.txt or expected_*.json | High |

### Advanced Template Pattern Checks

Read `response_transformation.gtpl` and verify proper type handling:

| Pattern | What to Find | Severity |
|---------|--------------|----------|
| Type check arrays | `{{- if kindIs "slice" .rawData}}` before iteration | Critical |
| Type check objects | `{{- if kindIs "map" .field}}` before nested access | High |
| Key existence | `{{- if hasKey .rawData "data"}}` before `.rawData.data` | Critical |
| Root context in range | `{{$.rootField}}` when accessing root in `{{range}}` | High |
| DateTime conversion | Date-only values get `T00:00:00Z` suffix | High |

### Metadata Files

| File | What to Check | Severity |
|------|---------------|----------|
| `external_id.gtpl` | Extracts unique ID: `{{.id}}` or similar | Critical |
| `external_updated_at.gtpl` | Extracts timestamp if API provides it | Medium |
| `config.json` → `dependsOnDataTypes` | Matches actual externalData usage | Critical |

**Record findings**: For each data pull, list missing patterns and tests.

---

## Step 6: ACTIONS - Audit Error Handling

**Goal**: Verify actions handle all error cases.

For each directory in `actions/`:

### Error Handling Checks

Read `response_transformation.gtpl` and verify ALL are present:

| HTTP Code | Required Pattern | Severity |
|-----------|------------------|----------|
| 400 | `{{- if eq .response.statusCode 400 }}` | Critical |
| 404 | `{{- if eq .response.statusCode 404 }}` | Critical |
| 500+ | `{{- if ge .response.statusCode 500 }}` | Critical |
| Success | Else clause returning success response | Critical |

### Input Safety Checks

Read `request_body.gtpl` and check:

| Pattern | What to Find | Severity |
|---------|--------------|----------|
| String escaping | `{{.inputs.field \| toJson}}` for text fields | High |

### Test Coverage

```bash
ls actions/<name>/_test_/
```

| Check | Requirement | Severity |
|-------|-------------|----------|
| Success test | `_test_/success/` with all fixtures | High |
| Error test | `_test_/error/` with 4xx response | High |

**Record findings**: For each action, list missing error handling and tests.

---

## Step 7: UI - Audit Flexible Cards

**Goal**: Verify UI templates follow best practices.

For each directory in `ui/templates/`:

### Template Checks

Read `flexible.card` and check:

| Check | What to Find | Severity |
|-------|--------------|----------|
| Language version | `<?Template languageVersion="1.1" ?>` | High |
| Label length | All labels under 15 characters | Medium |
| Text conditional | `<Text>` NOT inside `when=""` (wrap in `<Block>`) | High |
| Spacer conditional | `<Spacer>` NOT inside `when=""` (wrap in `<Block>`) | High |
| Boolean conditions | Uses `is not null` NOT `== true` | High |
| Address display | `displayStyle="stacked"` with `maxLines` | Medium |
| Currency default | `defaultCurrencyCode` specified | Medium |

### Common Label Violations

| Too Long | Use Instead |
|----------|-------------|
| Order Number | Order # |
| Shipping Address | Ship To |
| Payment Expiration | Exp. Date |
| Confirmation Number | Conf # |

### Advanced UI Pattern Checks

| Check | What to Find | Severity |
|-------|--------------|----------|
| Operators lowercase | `and`, `or`, `not` (NOT `AND`, `OR`, `NOT`) | Critical |
| Special vars valid | `__data`, `__root`, `__parent`, `__index` only | High |
| Image attributes | Uses `dataSource` NOT `imageUrl` | High |
| Image size | Uses `size` NOT `width/height` | High |
| Column width values | `auto`, `stretch`, `{N}px`, `{N}%` only (NOT `fill`) | High |
| Formula attributes | Uses `valueFontSize` NOT `fontSize` | High |

### Formula Filter Validation

Valid filters (search for these patterns):

| Filter | Valid Syntax |
|--------|--------------|
| Length | `{{ items \| length }}` |
| Title case | `{{ status \| title }}` |
| Upper/lower | `{{ code \| upper }}`, `{{ name \| lower }}` |
| Date format | `{{ date \| date('MM/DD/YYYY') }}` |
| Currency | `{{ total \| format_currency }}` |

### Test Data Checks

```bash
ls ui/templates/<name>/_test_/
```

| Check | Requirement | Severity |
|-------|-------------|----------|
| Test file exists | `_test_/default.json` present | High |
| Data structure | Matches `graphqlType` from config.json | High |
| Edge cases | Includes empty arrays, null fields | Medium |
| Long values | Tests label/value overflow | Medium |

**Record findings**: List UI issues found.

---

## Step 8: REPORT - Generate Findings Report

**Goal**: Summarize all findings in a structured report.

### Report Template

```markdown
# App Platform Audit Report

**App**: {from manifest.json}
**Version**: {version}
**Date**: {today}

## Summary

| Category | Status | Critical | High | Medium |
|----------|--------|----------|------|--------|
| CLI Validation | PASS/FAIL | - | - | - |
| Structure | PASS/FAIL | {n} | {n} | {n} |
| Authentication | PASS/FAIL | {n} | {n} | {n} |
| Schemas | PASS/FAIL | {n} | {n} | {n} |
| Data Pulls | PASS/FAIL | {n} | {n} | {n} |
| Actions | PASS/FAIL | {n} | {n} | {n} |
| UI Templates | PASS/FAIL | {n} | {n} | {n} |

**Overall**: {PASS if 0 critical, otherwise NEEDS WORK}

## Critical Issues (Must Fix)

{For each critical issue, include:}
- **File**: `path/to/file`
- **Issue**: Description of the problem
- **Fix**: Specific code or pattern to add/change

## High Priority Issues (Should Fix)

{Same format as critical}

## Medium Priority Issues (Consider Fixing)

{Same format}

## Suggested Remediation Plan

Based on findings, here is the recommended fix order:

1. **Fix critical issues first** (will cause runtime failures)
   - {List specific files and what to fix}

2. **Address high priority issues** (edge case failures)
   - {List specific files and what to fix}

3. **Consider medium issues** (best practice improvements)
   - {List specific files and what to fix}

4. **Re-run validation**
   ```bash
   appcfg validate --root .
   appcfg test data-pull --root .
   ```

## Files Audited

{List all files checked}
```

**Audit complete.** Review the findings above and consult Gladly's App Platform documentation for best practices guidance on implementing fixes.

---

## Severity Reference

| Severity | Definition | Action |
|----------|------------|--------|
| **Critical** | Will cause runtime failures | Must fix before deploy |
| **High** | May cause edge case failures | Should fix before deploy |
| **Medium** | Best practice violation | Fix when convenient |

---

## Reference

See [audit-checklist.md](references/audit-checklist.md) for the complete checklist in copyable format.
