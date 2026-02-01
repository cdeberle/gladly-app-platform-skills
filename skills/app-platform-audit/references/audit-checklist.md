# App Platform Audit Checklist

Complete checklist for auditing Gladly App Platform apps against best practices.

## CLI Validation

- [ ] `appcfg validate --root .` passes
- [ ] `appcfg test data-pull --root .` all tests pass
- [ ] `appcfg test action --root .` all tests pass (if actions exist)

---

## Project Structure

### Required Files

- [ ] `manifest.json` exists and is valid JSON
- [ ] `data/data_schema.graphql` exists
- [ ] `authentication/` directory exists
- [ ] At least one authentication method configured

### Data Pulls

- [ ] `data/pull/` contains at least one data pull
- [ ] Each data pull has:
  - [ ] `config.json`
  - [ ] `request_url.gtpl`
  - [ ] `external_id.gtpl`
  - [ ] `_test_/default/` directory with test fixtures

### Actions (if applicable)

- [ ] `actions/actions_schema.graphql` exists
- [ ] Each action has:
  - [ ] `config.json`
  - [ ] `request_url.gtpl`
  - [ ] `request_body.gtpl` (for POST/PUT)
  - [ ] `response_transformation.gtpl`
  - [ ] `_test_/success/` test fixtures
  - [ ] `_test_/error/` test fixtures

### UI Templates

- [ ] `ui/templates/` contains at least one template
- [ ] Each template has:
  - [ ] `config.json`
  - [ ] `flexible.card`
  - [ ] `_test_/default.json`

---

## GraphQL Schemas

### Data Schema

- [ ] All pullable types have `@dataType(name: "prefix_name", version: "X.Y")`
- [ ] Data type names use snake_case with app prefix
- [ ] Types defined before they are referenced (no forward declarations)
- [ ] Root types have `id: ID!` field
- [ ] Child types have parent reference field (e.g., `customerId: String`)
- [ ] Relationship directives used correctly:
  - [ ] `@parentId(template: "{{.id}}")` when child has parentId field
  - [ ] `@childIds(template: "{{.ids | join \",\"}}")` when parent has ID array
- [ ] DateTime fields:
  - [ ] Fields ending in `At`, `Date`, `Time` use `DateTime` type
  - [ ] `createdAt`, `updatedAt`, `orderDate`, etc.
- [ ] Currency fields:
  - [ ] Price/amount fields use `Currency` type
  - [ ] `totalPrice`, `amount`, `cost`, etc.
- [ ] Query type defines root queries

### Actions Schema (if exists)

- [ ] All mutations have `@action(name: "action_name")` directive
- [ ] Action names use snake_case
- [ ] Required inputs marked with `!`
- [ ] Return type includes `status: String!` and `message: String`

---

## Data Pull Templates

### request_url.gtpl - Root Pulls

- [ ] Checks for customer email/phone before using:
  ```go
  {{- if eq (len .customer.emailAddresses) 0}}
    {{- stop "Customer profile has no email addresses"}}
  {{- end}}
  ```
- [ ] Uses `urlquery` for all path and query parameters
- [ ] Accesses configuration via `.integration.configuration.fieldName`

### request_url.gtpl - Dependent Pulls

- [ ] Checks externalData exists:
  ```go
  {{- if not $customers}}
    {{- stop "No customer data available"}}
  {{- end}}
  ```
- [ ] Checks array not empty:
  ```go
  {{- if eq (len $customers) 0}}
    {{- stop "Customer not found"}}
  {{- end}}
  ```
- [ ] Checks ID field exists:
  ```go
  {{- if not $customerId}}
    {{- stop "Customer ID is empty"}}
  {{- end}}
  ```

### response_transformation.gtpl

- [ ] Handles null/missing rawData:
  ```go
  {{- if not .rawData}}
  []
  {{- else}}
  ```
- [ ] Extracts array correctly:
  ```go
  {{- if kindIs "slice" .rawData}}
    {{- $items = .rawData}}
  {{- else if hasKey .rawData "data"}}
    {{- $items = .rawData.data}}
  {{- end}}
  ```
- [ ] Handles empty arrays:
  ```go
  {{- if eq (len $items) 0}}
  []
  {{- else}}
  ```
- [ ] Uses `toJson` for string values with potential special characters
- [ ] Outputs `null` (not `""`) for missing optional fields
- [ ] Converts date-only values to DateTime format when needed

### external_id.gtpl

- [ ] Extracts correct ID field: `{{.id}}`

### external_parent_id.gtpl (child types)

- [ ] Extracts correct parent ID field: `{{.customerId}}`

---

## Advanced Template Patterns

### Type Checking (Critical)

- [ ] Uses `kindIs "slice"` before iterating over arrays:
  ```go
  {{- if kindIs "slice" .rawData}}
    {{- $items = .rawData}}
  {{- end}}
  ```
- [ ] Uses `kindIs "string"` or `kindIs "map"` before accessing nested fields
- [ ] Uses `hasKey` before accessing map keys:
  ```go
  {{- if hasKey .rawData "data"}}
    {{- $items = .rawData.data}}
  {{- end}}
  ```

### Root Context in Range (Critical)

- [ ] Uses `$` prefix when accessing root data inside `{{range}}`:
  ```go
  {{range $i, $item := .items}}
    Item {{$i}} of {{len $.items}}  {{/* Note: $.items, not .items */}}
  {{end}}
  ```

### DateTime Handling

- [ ] Date-only strings converted to ISO8601:
  ```go
  {{if contains "T" $date}}"{{$date}}"{{else}}"{{$date}}T00:00:00Z"{{end}}
  ```
- [ ] Fields ending in `At`, `Date`, `Time` output proper DateTime

### Metadata Templates

- [ ] `external_id.gtpl` exists and extracts unique ID
- [ ] `external_updated_at.gtpl` exists (if API provides timestamps)
- [ ] `dependsOnDataTypes` in config.json matches externalData usage in templates

---

## Action Templates

### config.json

- [ ] `httpMethod` specified (POST, PUT, DELETE)
- [ ] `contentType` specified for POST/PUT (usually `application/json`)

### request_url.gtpl

- [ ] Uses `urlquery` for path parameters
- [ ] Accesses inputs via `.inputs.fieldName`

### request_body.gtpl

- [ ] Uses `toJson` for user-provided string values:
  ```go
  "reason": {{.inputs.reason | toJson}}
  ```
- [ ] Handles optional inputs with conditionals

### response_transformation.gtpl

- [ ] Handles 400 Bad Request
- [ ] Handles 404 Not Found
- [ ] Handles 500+ Server Errors
- [ ] Has success case (else clause)
- [ ] Returns consistent structure:
  ```json
  {"status": "success|error", "message": "..."}
  ```

---

## UI Templates

### config.json

- [ ] `graphqlSchema` is "data" or "actions"
- [ ] `graphqlType` matches a type in the schema

### flexible.card

- [ ] Starts with `<?Template languageVersion="1.1" ?>`
- [ ] Card has title (can use interpolation)
- [ ] Labels are under 15 characters:
  - [ ] Use "Order #" not "Order Number"
  - [ ] Use "Ship To" not "Shipping Address"
  - [ ] Use "Exp. Date" not "Expiration Date"

### Conditional Content

- [ ] `<Text>` elements do NOT have `when` attribute directly
- [ ] `<Spacer>` elements do NOT have `when` attribute directly
- [ ] Conditionals use `<Block when="...">` wrapper:
  ```xml
  <Block when="field is not null">
    <Text>...</Text>
  </Block>
  ```
- [ ] Boolean conditions use `is not null` not `== true`

### Data Display

- [ ] Addresses use `displayStyle="stacked"` with `maxLines="3"`
- [ ] Currency values have `defaultCurrencyCode="USD"` (or appropriate)
- [ ] DateTime values have `format` specified

### List/Drawer Pattern

- [ ] `<List>` has `dataSource` attribute
- [ ] `<Drawer>` has `initialState` (expandFirst, expanded, collapsed)
- [ ] Fixed section has 3-4 key fields
- [ ] Expandable section has detail fields

### Conditional Expression Operators

- [ ] Logical operators are **lowercase**: `and`, `or`, `not`
  ```xml
  <!-- CORRECT -->
  <Block when="(status == 'active') and (amount > 0)">

  <!-- WRONG - will fail validation -->
  <Block when="(status == 'active') AND (amount > 0)">
  ```

### Image Component

- [ ] Uses `dataSource` NOT `imageUrl`:
  ```xml
  <!-- CORRECT -->
  <Image dataSource="imageUrl" size="48px" />

  <!-- WRONG -->
  <Image imageUrl="{{url}}" width="48px" height="48px" />
  ```

### Column Width Values

Valid values: `auto`, `stretch`, `{N}px`, `{N}%`

- [ ] No use of `width="fill"` (use `width="stretch"` instead)

### Formula Component Attributes

- [ ] Uses `valueFontSize` NOT `fontSize`
- [ ] Uses `valueFontWeight` NOT `fontWeight`

### Special Variables (Inside List only)

| Variable | Purpose | Valid |
|----------|---------|-------|
| `__data` | Current scope | Yes |
| `__root` | Top-level data | Yes |
| `__parent` | Parent scope | Yes (List only) |
| `__index` | Current index | Yes (List only) |

### Formula Filters

- [ ] Valid filter syntax used:
  - `{{ field | length }}` for array count
  - `{{ field | title }}` for title case
  - `{{ field | upper }}` / `{{ field | lower }}`
  - `{{ field | date('format') }}` for dates

### UI Test Data

- [ ] `_test_/default.json` exists for each template
- [ ] Test data structure matches `graphqlType` from config.json
- [ ] Test data includes edge cases:
  - [ ] Empty arrays where lists are used
  - [ ] Null values for optional fields
  - [ ] Long strings to test label/value overflow
  - [ ] Maximum expected items in lists

---

## Test Coverage

### Data Pull Tests

For each data pull in `_test_/default/`:

- [ ] `integration.json` with test config/secrets
- [ ] `customer.json` for root pulls (with email/phone)
- [ ] `externalData.json` for dependent pulls
- [ ] `rawData.json` with sample API response
- [ ] `expected_request_url.txt` with expected URL
- [ ] `expected_response_transformation.json` (if transformation exists)

### Edge Case Tests

- [ ] Empty array response: `{"data": []}`
- [ ] Null fields in response
- [ ] Customer not found scenario
- [ ] Missing optional fields

### Action Tests

For each action:

- [ ] `_test_/success/` with:
  - [ ] `inputs.json`
  - [ ] `rawData.json`
  - [ ] `response.json` (statusCode: 200)
  - [ ] `expected_response_transformation.json`
- [ ] `_test_/error/` with:
  - [ ] `inputs.json`
  - [ ] `rawData.json`
  - [ ] `response.json` (statusCode: 400/404/500)
  - [ ] `expected_response_transformation.json`

---

## Authentication

### Directory Structure

- [ ] `authentication/` directory exists
- [ ] At least one auth method configured:
  - [ ] `authentication/headers/*.gtpl` for static headers, OR
  - [ ] `authentication/oauth/` for OAuth flows

### Header Authentication

- [ ] Header templates use `.integration.secrets.*` for tokens
- [ ] API URLs in `.integration.configuration.*`
- [ ] Tokens/keys in `.integration.secrets.*`
- [ ] No hardcoded credentials

### OAuth Client Credentials

- [ ] `oauth/config.json` has `"grantType": "client_credentials"`
- [ ] `oauth/access_token/config.json` exists
- [ ] `oauth/access_token/request_url.gtpl` exists
- [ ] `oauth/access_token/request_body.gtpl` exists
- [ ] Body uses `{{urlquery ...}}` for all parameters
- [ ] `headers/Authorization.gtpl` uses `.integration.secrets.access_token`

### OAuth Authorization Code

- [ ] `oauth/config.json` has `"grantType": "authorization_code"`
- [ ] `oauth/auth_code/` directory exists with:
  - [ ] `config.json`
  - [ ] `request_url.gtpl` using `{{.correlationId}}` for state
- [ ] `oauth/access_token/` directory exists
- [ ] `oauth/refresh_token/` directory exists with:
  - [ ] `config.json`
  - [ ] `request_url.gtpl`
  - [ ] `request_body.gtpl`

### Request Signing (if present)

- [ ] `authentication/request_signing/*.gtpl` uses `.integration.secrets.*` for key
- [ ] Signing payload includes timestamp
- [ ] Uses `hmacSha256` or similar for signature

### Configuration vs Secrets Separation

| Item | Should Be In |
|------|--------------|
| API base URLs | `configuration` |
| Client ID | `configuration` |
| Client secrets | `secrets` |
| API tokens | `secrets` |
| Passwords | `secrets` |
| Private keys | `secrets` |

---

## Common Issues Quick Check

| Issue | How to Find | Fix |
|-------|-------------|-----|
| Empty array not handled | Search for `| default` on arrays | Use `len()` check |
| Missing stop on null | Search `externalData` without `if not` | Add null check + stop |
| Text with when | Search `<Text when=` | Wrap in `<Block when=>` |
| Long labels | Visual review of labels | Shorten to <15 chars |
| Missing error handling | Check action response_transformation | Add 400/404/500 handling |
| Unescaped strings | Search `.inputs.` without `toJson` | Add `| toJson` |
