---
name: app-platform-builder
description: Build Gladly App Platform apps from scratch. Use this skill when creating a new integration with an external system API. Guides you from raw API payload analysis through manifest creation, GraphQL schema generation, data pulls, actions, UI templates, and validation. Works with any REST API.
---

# App Platform Builder

Build a complete Gladly App Platform app from scratch using a 7-phase checkpoint workflow.

## Prerequisites

Ensure `appcfg` CLI is installed: https://github.com/gladly/app-platform-appcfg-cli/releases

## Workflow Overview

```
PHASE 1: DISCOVER    → Gather API info, analyze payloads
         ↓ checkpoint
PHASE 2: SCAFFOLD    → Initialize project, set up auth
         ↓ checkpoint
PHASE 3: MODEL       → Generate GraphQL schemas
         ↓ checkpoint
PHASE 4: CONNECT     → Create data pulls
         ↓ checkpoint
PHASE 5: ACTIONS     → Create action handlers (if needed)
         ↓ checkpoint
PHASE 6: PRESENT     → Create UI templates
         ↓ checkpoint
PHASE 7: VERIFY      → Test and validate
         ↓
HANDOFF TO /app-platform-deploy
```

**IMPORTANT**: Pause after each phase for user review. Do not proceed without explicit confirmation.

---

## Phase 1: DISCOVER - Gather API Information

**Required Reading**: None (information gathering phase)

**Goal**: Understand the external API and plan the data model.

### Questions to Ask

1. **What external system?** (name, API base URL)
2. **Authentication method?** (Bearer Token, OAuth 2.0, Basic Auth, API Key)
3. **Sample JSON payload** from the main API endpoint
4. **Actions needed?** (e.g., cancel order, issue refund)

### Payload Analysis

When you receive JSON, identify:

| Element | Look For |
|---------|----------|
| Root entity | Top-level object (usually Customer) |
| Child entities | Nested arrays (orders, subscriptions) |
| ID fields | `id`, `*Id`, `*ID` |
| DateTime fields | `*At`, `*Date`, `*Time`, ISO8601 strings |
| Currency fields | `*Price`, `total`, `amount`, `cost` |

### PHASE 1 CHECKPOINT

Present analysis:
```
Based on your API payload, I identified:

**Data Types:**
- [Root type] - looked up by [email/phone]
- [Child type] - linked by [field]

**Actions:** (if any)
- [action name] → [HTTP method] [endpoint]

**Ask user**: "Phase 1 complete. Ready to scaffold the project (Phase 2)?"
```

**If unclear**: Ask clarifying questions before proceeding.

---

## Phase 2: SCAFFOLD - Initialize Project

**Required Reading**: [authentication-patterns.md](references/authentication-patterns.md)

**Goal**: Create app directory structure and authentication.

### Initialize

```bash
appcfg init \
  --author "<company.com>" \
  --app-name "<AppName>" \
  --description "<Description>" \
  --version "1.0.0" \
  --root .
```

### Configure Authentication

Based on auth method, create the appropriate templates. See [authentication-patterns.md](references/authentication-patterns.md) for complete patterns.

For **Bearer Token**: Create `authentication/headers/Authorization.gtpl`

For **OAuth**: Run `appcfg add oauth client_credentials --root .` or `appcfg add oauth authorization_code --root .`

### PHASE 2 CHECKPOINT

Verify deliverables:
- [ ] `manifest.json` exists
- [ ] `authentication/` directory configured
- [ ] Run `appcfg validate --root .` - should pass

**Ask user**: "Phase 2 complete. Ready to generate schemas (Phase 3)?"

**If validation fails**: Check manifest.json format and authentication template syntax.

---

## Phase 3: MODEL - Generate GraphQL Schemas

**Required Reading**: [graphql-schema-patterns.md](references/graphql-schema-patterns.md)

**Goal**: Create `data/data_schema.graphql` and `actions/actions_schema.graphql`.

### Type Mapping

| JSON Pattern | GraphQL Type |
|--------------|--------------|
| `"field": "text"` | `String` |
| `"field": 123` | `Int` |
| `"field": 12.34` | `Float` |
| `"field": true` | `Boolean` |
| `*At`, `*Date` | `DateTime` |
| `*Price`, `total` | `Currency` |
| Nested objects | Embedded type |
| Arrays | `[Type]` |

### Relationship Directives

**@parentId** - Child has field referencing parent:
```graphql
type Customer {
  orders: [Order] @parentId(template: "{{.id}}")
}
```

**@childIds** - Parent has array of child IDs:
```graphql
type Customer {
  orders: [Order] @childIds(template: "{{.orderIds | join \",\"}}")
}
```

See [code-templates.md](references/code-templates.md) for complete schema examples.

### PHASE 3 CHECKPOINT

Present schemas for review:
- Data schema types and fields
- Relationship directives (@parentId/@childIds)
- Action signatures (if any)

**Ask user**: "Phase 3 complete. Review the schemas. Ready to create data pulls (Phase 4)?"

**If types seem wrong**: Ask user to clarify field purposes.

---

## Phase 4: CONNECT - Create Data Pulls

**Required Reading**: [data-pull-templates.md](references/data-pull-templates.md), [go-template-reference.md](references/go-template-reference.md)

**Goal**: Create data pull configurations for each data type.

### Directory Structure

```
data/pull/<type_name>/
├── config.json
├── request_url.gtpl
├── response_transformation.gtpl (if needed)
├── external_id.gtpl
├── external_parent_id.gtpl (if child type)
└── _test_/default/
```

### Key Patterns

**Root pulls**: Look up by email/phone from `.customer`

**Dependent pulls**: Access parent data from `.externalData.<parent_type>`

**Safety checks**: Always validate data exists before using:
```go
{{- if not $customers}}{{- stop "No customer data"}}{{- end}}
{{- if eq (len $customers) 0}}{{- stop "Customer not found"}}{{- end}}
```

See [code-templates.md](references/code-templates.md) for complete data pull templates.

### Test Each Pull

```bash
appcfg test data-pull <name> --root .
```

### PHASE 4 CHECKPOINT

Verify:
- [ ] All data pull tests pass
- [ ] Root pull correctly uses customer email/phone
- [ ] Dependent pulls have proper safety checks

**Ask user**: "Phase 4 complete. All data pulls tested. Ready for actions (Phase 5)?"

**If tests fail**: Check `_test_/<name>/_output_/` for errors. Common issues: empty array handling, missing safety checks. See [go-template-reference.md](references/go-template-reference.md#critical-gotchas).

---

## Phase 5: ACTIONS - Create Action Handlers

**Required Reading**: [action-templates.md](references/action-templates.md)

**Goal**: Create action configurations for mutations (if any actions defined).

**Skip this phase** if no actions were identified in Phase 1.

### Directory Structure

```
actions/<action_name>/
├── config.json
├── request_url.gtpl
├── request_body.gtpl
├── response_transformation.gtpl
└── _test_/success/ and _test_/error/
```

### Key Points

- Actions **must** handle HTTP error codes (400, 404, 500)
- Always escape user input with `toJson`
- Create test fixtures for both success and error cases

See [code-templates.md](references/code-templates.md) for complete action templates.

### Test Each Action

```bash
appcfg test action <name> --root .
```

### PHASE 5 CHECKPOINT

Verify:
- [ ] All action tests pass (success and error cases)
- [ ] Error responses are user-friendly

**Ask user**: "Phase 5 complete. Actions tested. Ready to create UI (Phase 6)?"

**If tests fail**: Check response_transformation.gtpl handles all status codes.

---

## Phase 6: PRESENT - Create UI Templates

**Required Reading**: [ui-card-patterns.md](references/ui-card-patterns.md)

**Goal**: Create Flexible Card templates for Gladly Hero.

### Directory Structure

```
ui/templates/<card-name>/
├── config.json
├── flexible.card
└── _test_/default.json
```

### Key Patterns

- **Fixed section**: 3-4 most important fields (ID, status, total)
- **Expandable**: Secondary details (dates, addresses, line items)
- **Labels**: Keep under 15 characters
- **Conditionals**: Use `<Block when="...">` not `<Text when="...">`

See [code-templates.md](references/code-templates.md) for UI template example.

### PHASE 6 CHECKPOINT

Verify:
- [ ] Card displays correctly with test data
- [ ] Labels don't truncate
- [ ] Drawer expand/collapse works

**Ask user**: "Phase 6 complete. UI template ready. Ready for final validation (Phase 7)?"

---

## Phase 7: VERIFY - Test and Validate

**Required Reading**: None (execution phase)

**Goal**: Ensure app passes all tests and validates.

### Run Full Validation

```bash
# Test all data pulls
appcfg test data-pull --root .

# Test all actions (if any)
appcfg test action --root .

# Validate entire app
appcfg validate --root .
```

### Validation Checklist

- [ ] All data pull tests pass
- [ ] All action tests pass
- [ ] `appcfg validate --root .` returns "valid"
- [ ] GraphQL types match data pull configs

### PHASE 7 CHECKPOINT

```
Validation Results:
✓ Data pulls: PASSED
✓ Actions: PASSED (or N/A)
✓ Validation: VALID

**Ask user**: "Phase 7 complete. App is ready for deployment. Invoke /app-platform-deploy?"
```

**If validation fails**: Run individual tests to isolate the issue. See [troubleshooting.md](../app-platform-deploy/references/troubleshooting.md).

---

## Handoff to Deploy

Before invoking `/app-platform-deploy`, verify:

- [ ] `appcfg validate --root .` returns "valid"
- [ ] All `appcfg test data-pull --root .` tests pass
- [ ] No outstanding issues from checkpoints

**Invoke**: `/app-platform-deploy`

---

## Reference Documentation

| When | Read |
|------|------|
| Phase 2 (Auth) | [authentication-patterns.md](references/authentication-patterns.md) |
| Phase 3 (Schema) | [graphql-schema-patterns.md](references/graphql-schema-patterns.md) |
| Phase 4 (Data) | [data-pull-templates.md](references/data-pull-templates.md), [go-template-reference.md](references/go-template-reference.md) |
| Phase 5 (Actions) | [action-templates.md](references/action-templates.md) |
| Phase 6 (UI) | [ui-card-patterns.md](references/ui-card-patterns.md) |
| Code examples | [code-templates.md](references/code-templates.md) |
