# Troubleshooting Guide for Gladly App Platform

This guide documents common issues encountered when developing Gladly apps and how to resolve them.

## Debugging Workflow

When a card fails to load with "Something's wrong with this card", follow this sequence:

### Step 1: Check Server Logs
```bash
appcfg apps logs list --root .
```
- Often empty for template-related issues
- Useful for authentication or network errors

### Step 2: Test Data Pull Locally
```bash
appcfg test data-pull <pull-name> --root .
```
- Uses test fixtures in `_test_/` directory
- Validates template syntax and transformation logic
- **Does not** test against real API

### Step 3: Test Against Real API
```bash
appcfg run data-graphql --root .
```
Then run a query like:
```graphql
query {
  customer(email: "test@example.com") {
    id
    orders { id orderNumber }
  }
}
```
- Tests actual API responses
- Reveals runtime issues not caught by local tests
- Essential for catching empty array handling bugs

### Step 4: Create Edge Case Test Fixtures
Add fixtures for:
- Empty arrays: `{"data": [], "pagination": {...}}`
- Missing fields
- Null values
- Malformed responses

---

## Common Error Messages

### "can't evaluate field [fieldname] in type interface {}"

**Cause:** Template is trying to access a field on an empty or nil value, typically when an array is empty.

**Common scenario:** API returns `{"data": []}` and template uses `.rawData.data` with `| default` which doesn't work for empty arrays.

**Fix:** Use explicit empty array handling:
```go
{{- $items := list}}
{{- if kindIs "slice" .rawData}}
  {{- $items = .rawData}}
{{- else if hasKey .rawData "data"}}
  {{- $items = .rawData.data}}
{{- end}}
{{- if eq (len $items) 0}}
[]
{{- else}}
...
{{- end}}
```

---

### "Something's wrong with this card" (generic)

**Possible causes:**
1. Response transformation produces invalid JSON
2. GraphQL schema mismatch with transformed data
3. Empty array handling issue (see above)
4. Dependent data pull failed silently

**Debug steps:**
1. Run `appcfg test data-pull <name> --root .` for each data pull
2. Test with `appcfg run data-graphql --root .` against real API
3. Check that all `dependsOnDataTypes` pulls succeed first
4. Verify JSON output is valid (no trailing commas, proper escaping)

---

### "No customer data available" or "No customer found"

**Cause:** A dependent data pull couldn't find the parent data.

**This is expected when:** The customer lookup didn't find a match for the email/phone.

**This is a bug when:** The `externalData` reference is incorrect in `request_url.gtpl`.

**Check:**
```go
{{- $customers := .externalData.demo_crm_customer}}  // Verify this data type name
```

---

### Card displays `map[key:value key2:value2]` instead of formatted data

**Cause:** Object field is being output with `toJson` without string formatting.

**Fix:** Format objects into strings before output:
```go
{{- $formatted := "" -}}
{{- if $item.nestedObject -}}
  {{- if $item.nestedObject.field1 -}}
    {{- $formatted = $item.nestedObject.field1 -}}
  {{- end -}}
{{- end -}}
"fieldName": {{if $formatted}}{{$formatted | toJson}}{{else}}null{{end}}
```

---

### Credentials Error

**Symptoms:** Commands fail with authentication or permission errors.

**Check:**
1. Verify `.env` has correct values:
   - `GLADLY_APP_CFG_HOST` matches target environment (UAT vs Production)
   - `GLADLY_APP_CFG_USER` is the email tied to the API token
   - `GLADLY_APP_CFG_TOKEN` is valid and not expired
2. Verify API token has appropriate permissions

---

### Upgrade Fails

**Symptoms:** `appcfg apps upgrade` command fails.

**Common causes:**
1. **Cannot upgrade across major versions** - Breaking changes require new configs
2. **New version not installed** - Install new version before upgrading
3. **Version doesn't exist** - Verify version string format

**Fix:** For breaking changes (major version bump), create new configurations instead of upgrading.

---

## Decision Tree: Card Loading Failures

```
Card fails to load
│
├─ Check appcfg apps logs list --root .
│  ├─ Auth error → Check credentials in integration config
│  ├─ Network error → Check API URL, firewall
│  └─ Empty logs → Continue...
│
├─ Run appcfg test data-pull <name> --root .
│  ├─ Template error → Fix template syntax
│  ├─ Test passes → Continue...
│  └─ No test fixtures → Create _test_/default/
│
├─ Run appcfg run data-graphql --root . with real email
│  ├─ Error returned → Check response_transformation.gtpl
│  ├─ Empty data → Check if customer exists in external system
│  └─ Data returned → Check UI template binding
│
└─ Check dependent data pulls
   ├─ Parent data missing → Fix parent pull first
   └─ All pulls succeed → Check GraphQL schema types
```

---

## Testing Checklist

Before deploying, test these scenarios:

- [ ] Customer with all data types populated
- [ ] Customer with empty orders
- [ ] Customer with empty subscriptions
- [ ] Customer with empty reservations
- [ ] Customer with no loyalty profile
- [ ] Customer not found in external system
- [ ] API returns paginated response
- [ ] API returns direct array response
- [ ] Fields with null values
- [ ] Fields with nested objects
- [ ] Long text that might overflow in UI
