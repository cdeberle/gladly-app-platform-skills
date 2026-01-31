---
name: app-platform-builder
description: Build Gladly App Platform apps from scratch. Use this skill when creating a new integration with an external system API. Guides you from raw API payload analysis through manifest creation, GraphQL schema generation, data pulls, actions, UI templates, and validation. Works with any REST API.
---

# App Platform Builder

This skill guides you through creating a complete Gladly App Platform app from scratch, including data pulls, actions (mutations), and UI templates.

## Prerequisites

Ensure the `appcfg` CLI is installed:
```bash
cd /path/to/.app-platform-toolkit/appcfg-cli && ./install.sh
```

## Required Context Files

Before starting, read these toolkit references for guidance:
- `.app-platform-toolkit/ai-context/app_platform_context.md` - Complete platform specification
- `.app-platform-toolkit/ai-context/go-template-gotchas.md` - Template pitfalls
- `.app-platform-toolkit/ai-context/card-ui-best-practices.md` - UI guidelines
- `.app-platform-toolkit/templates/patterns/data-pulls/data-pull-patterns.md` - Data pull patterns

## Workflow Overview

This skill uses a **checkpoint workflow** - pause after each phase for user review before proceeding.

```
PHASE 1: DISCOVER    → Gather API info, analyze payloads
         ↓ CHECKPOINT
PHASE 2: SCAFFOLD    → Initialize project, set up auth
         ↓ CHECKPOINT
PHASE 3: MODEL       → Generate GraphQL schemas
         ↓ CHECKPOINT
PHASE 4: CONNECT     → Create data pulls
         ↓ CHECKPOINT
PHASE 5: ACTIONS     → Create action handlers
         ↓ CHECKPOINT
PHASE 6: PRESENT     → Create UI templates
         ↓ CHECKPOINT
PHASE 7: VERIFY      → Test and validate
         ↓
HANDOFF TO /app-platform-deploy
```

---

## Phase 1: DISCOVER - Gather API Information

**Goal:** Understand the external API and plan the data model.

### Questions to Ask the User

1. **What external system are you integrating?**
   - System name (e.g., "Acme OMS", "Shopify", "Custom CRM")
   - API base URL (e.g., `https://api.acme.com/v1`)

2. **What authentication method does the API use?**
   - Bearer Token / API Key
   - OAuth 2.0 Client Credentials
   - OAuth 2.0 Authorization Code
   - Basic Auth

3. **Request a sample JSON payload from their API:**
   ```
   Please paste a sample JSON response from your API for the main data type
   you want to display (e.g., customer data with orders, subscriptions, etc.)
   ```

4. **What actions should agents be able to perform?** (Optional)
   - e.g., Cancel order, Issue refund, Update subscription
   - For each action: endpoint URL pattern, HTTP method, required inputs

### Payload Analysis

When you receive the JSON payload, analyze it to identify:

| Element | Look For |
|---------|----------|
| Root entity | Top-level object (usually Customer) |
| Child entities | Nested arrays (orders, subscriptions, etc.) |
| ID fields | `id`, `*Id`, `*ID` - for external_id |
| Parent references | `customerId`, `parentId` - for external_parent_id |
| DateTime fields | `*At`, `*Date`, `*Time`, ISO8601 strings |
| Currency fields | `*Price`, `total`, `amount`, `cost` |
| Relationships | @parentId (child has parentId) vs @childIds (parent has array of IDs) |

### CHECKPOINT

Present your analysis:
```
Based on your API payload, I identified:

**Data Types:**
- Customer (root) - looked up by email
- Order (child of Customer) - linked by customerId
- LineItem (nested in Order) - no separate pull needed

**Actions:**
- cancelOrder(orderId, reason) → POST /orders/{id}/cancel
- refundOrder(orderId, amount) → POST /orders/{id}/refund

**Proceed with scaffolding? [Yes/No]**
```

---

## Phase 2: SCAFFOLD - Initialize Project

**Goal:** Create app directory structure and authentication.

### Initialize App

```bash
appcfg init \
  --author "<company.com>" \
  --app-name "<AppName>" \
  --description "<Description>" \
  --version "1.0.0" \
  --root .
```

### Create Authentication

Based on user's auth method:

**Bearer Token:**
```
authentication/headers/Authorization.gtpl:
```
```go
Bearer {{.integration.secrets.apiToken}}
```

**API Key Header:**
```
authentication/headers/X-API-Key.gtpl:
```
```go
{{.integration.secrets.apiKey}}
```

**OAuth Client Credentials:**
```bash
appcfg add oauth client_credentials --root .
```

### CHECKPOINT

Show created structure:
```
Created app structure:
├── manifest.json
├── authentication/
│   └── headers/
│       └── Authorization.gtpl
├── data/
│   └── data_schema.graphql (empty)
├── actions/
│   └── actions_schema.graphql (empty)
└── ui/
    └── templates/

Proceed to schema generation? [Yes/No]
```

---

## Phase 3: MODEL - Generate GraphQL Schemas

**Goal:** Create `data/data_schema.graphql` and `actions/actions_schema.graphql`.

### Data Schema Type Mapping

| JSON Pattern | GraphQL Type | Example |
|--------------|--------------|---------|
| `"field": "text"` | `String` | `name: String` |
| `"field": 123` | `Int` | `quantity: Int` |
| `"field": 12.34` | `Float` | `rating: Float` |
| `"field": true` | `Boolean` | `isActive: Boolean` |
| `*At`, `*Date`, `*Time` | `DateTime` | `createdAt: DateTime` |
| `*Price`, `total`, `amount` | `Currency` | `totalPrice: Currency` |
| Nested objects | Embedded type | `address: Address` |
| Arrays of objects | Child type | `orders: [Order]` |
| Arrays of primitives | `[String]` | `tags: [String]` |

### Data Schema Example

```graphql
type LineItem {
  id: String
  productName: String
  quantity: Int
  price: Currency
}

type Order @dataType(name: "acme_order", version: "1.0") {
  id: ID!
  orderNumber: String!
  status: String
  totalPrice: Currency
  orderDate: DateTime
  customerId: String
  lineItems: [LineItem]
}

type Customer @dataType(name: "acme_customer", version: "1.0") {
  id: ID!
  email: String!
  fullName: String
  createdAt: DateTime
  orders: [Order] @parentId(template: "{{.id}}")
}

type Query {
  customer: Customer
  orders: [Order]
}
```

### Actions Schema Example

```graphql
type Mutation {
  cancelOrder(orderId: String!, reason: String!): ActionResult @action(name: "cancel_order")
  refundOrder(orderId: String!, amount: Float!, reason: String): ActionResult @action(name: "refund_order")
}

type ActionResult {
  status: String!
  message: String
}
```

### Relationship Directives

**@parentId** - When child has a field referencing parent:
```graphql
# Child Order has customerId field
type Customer {
  orders: [Order] @parentId(template: "{{.id}}")
}
```

**@childIds** - When parent has array of child IDs:
```graphql
# Customer has orderIds array
type Customer {
  orders: [Order] @childIds(template: "{{.orderIds | join \",\"}}")
}
```

### CHECKPOINT

Present schemas for review:
```
Generated GraphQL schemas:

**data/data_schema.graphql:**
[show schema]

**actions/actions_schema.graphql:**
[show schema]

Please review:
1. Are the field types correct?
2. Are DateTime/Currency fields identified correctly?
3. Are the relationships (@parentId/@childIds) correct?
4. Are the action signatures correct?

Proceed to create data pulls? [Yes/No]
```

---

## Phase 4: CONNECT - Create Data Pulls

**Goal:** Create data pull configurations for each data type.

### For Each Data Type, Create:

```
data/pull/<type_name>/
├── config.json
├── request_url.gtpl
├── response_transformation.gtpl (if needed)
├── external_id.gtpl
├── external_parent_id.gtpl (if child type)
└── _test_/
    └── default/
        ├── integration.json
        ├── customer.json (for root pulls)
        ├── externalData.json (for dependent pulls)
        ├── rawData.json
        ├── expected_request_url.txt
        └── expected_response_transformation.json
```

### Root Data Pull (Customer Lookup)

**config.json:**
```json
{
  "dataType": {
    "name": "acme_customer",
    "version": "1.0"
  },
  "httpMethod": "GET"
}
```

**request_url.gtpl:**
```go
{{- /* Look up customer by email */}}
{{- if eq (len .customer.emailAddresses) 0}}
  {{- stop "Customer profile has no email addresses"}}
{{- end}}
{{- $email := ""}}
{{- if .customer.primaryEmailAddress}}
  {{- $email = .customer.primaryEmailAddress}}
{{- else}}
  {{- $email = index .customer.emailAddresses 0}}
{{- end}}
{{- $baseUrl := .integration.configuration.apiBaseUrl}}
{{$baseUrl}}/api/customers?email={{urlquery $email}}
```

**external_id.gtpl:**
```go
{{.id}}
```

### Dependent Data Pull (Orders)

**config.json:**
```json
{
  "dataType": {
    "name": "acme_order",
    "version": "1.0"
  },
  "httpMethod": "GET",
  "dependsOnDataTypes": ["acme_customer"]
}
```

**request_url.gtpl with Safety Checks:**
```go
{{- /* Fetch orders for customer */}}
{{- $customers := .externalData.acme_customer}}
{{- if not $customers}}
  {{- stop "No customer data available"}}
{{- end}}
{{- if eq (len $customers) 0}}
  {{- stop "Customer not found"}}
{{- end}}
{{- $customerId := (index $customers 0).id}}
{{- if not $customerId}}
  {{- stop "Customer ID is empty"}}
{{- end}}
{{- $baseUrl := .integration.configuration.apiBaseUrl}}
{{$baseUrl}}/api/customers/{{urlquery $customerId}}/orders
```

**external_id.gtpl:**
```go
{{.id}}
```

**external_parent_id.gtpl:**
```go
{{.customerId}}
```

### Response Transformation (if needed)

```go
{{- if not .rawData}}
[]
{{- else}}

{{- /* Extract array from response */}}
{{- $items := list}}
{{- if kindIs "slice" .rawData}}
  {{- $items = .rawData}}
{{- else if hasKey .rawData "data"}}
  {{- $items = .rawData.data}}
{{- end}}

{{- if eq (len $items) 0}}
[]
{{- else}}
[
{{- range $i, $item := $items}}
{{- if $i}},{{end}}
{
  "id": "{{$item.id}}",
  "orderNumber": {{$item.orderNumber | toJson}},
  "status": "{{$item.status}}",
  "totalPrice": {{if $item.totalPrice}}{{$item.totalPrice}}{{else}}null{{end}},
  "orderDate": {{if $item.orderDate}}"{{$item.orderDate}}"{{else}}null{{end}},
  "customerId": "{{$item.customerId}}"
}
{{- end}}
]
{{- end}}
{{- end}}
```

### Test Data Pull

```bash
appcfg test data-pull <name> --root .
```

### CHECKPOINT

```
Created data pulls:
- customer_lookup: GET /api/customers?email={email}
- orders: GET /api/customers/{customerId}/orders

Test results:
✓ customer_lookup: PASSED
✓ orders: PASSED

Proceed to create actions? [Yes/No]
```

---

## Phase 5: ACTIONS - Create Action Handlers

**Goal:** Create action configurations for mutations.

### For Each Action, Create:

```
actions/<action_name>/
├── config.json
├── request_url.gtpl
├── request_body.gtpl
├── response_transformation.gtpl
└── _test_/
    ├── success/
    │   ├── inputs.json
    │   ├── rawData.json
    │   ├── response.json
    │   └── expected_response_transformation.json
    └── error/
        ├── inputs.json
        ├── rawData.json
        ├── response.json
        └── expected_response_transformation.json
```

### Action Example: cancel_order

**config.json:**
```json
{
  "httpMethod": "POST",
  "contentType": "application/json"
}
```

**request_url.gtpl:**
```go
{{.integration.configuration.apiBaseUrl}}/api/orders/{{urlquery .inputs.orderId}}/cancel
```

**request_body.gtpl:**
```json
{
  "reason": "{{.inputs.reason}}"
}
```

**response_transformation.gtpl:**
```go
{{- if eq .response.statusCode 404 }}
{
  "status": "error",
  "message": "Order not found."
}
{{- else if eq .response.statusCode 400 }}
{
  "status": "error",
  "message": {{if .rawData.error}}{{.rawData.error | toJson}}{{else}}"Invalid request"{{end}}
}
{{- else if ge .response.statusCode 500 }}
{
  "status": "error",
  "message": "Server error. Please try again later."
}
{{- else }}
{
  "status": "success",
  "message": {{if .rawData.message}}{{.rawData.message | toJson}}{{else}}"Order cancelled successfully."{{end}}
}
{{- end}}
```

### CHECKPOINT

```
Created actions:
- cancel_order: POST /api/orders/{orderId}/cancel
- refund_order: POST /api/orders/{orderId}/refund

Test results:
✓ cancel_order/success: PASSED
✓ cancel_order/error: PASSED

Proceed to create UI templates? [Yes/No]
```

---

## Phase 6: PRESENT - Create UI Templates

**Goal:** Create Flexible Card templates for Gladly Hero.

### For Each Card, Create:

```
ui/templates/<card-name>/
├── config.json
├── flexible.card
└── _test_/
    └── default.json
```

### config.json

```json
{
  "description": "Display customer orders",
  "graphqlSchema": "data",
  "graphqlType": "Customer"
}
```

### Flexible Card Template

```xml
<?Template languageVersion="1.1" ?>
<Card title="{{ orders | length }} Orders">
  <List dataSource="orders" separateItems="true">
    <Drawer initialState="expandFirst">
      <Fixed>
        <StringValue dataSource="orderNumber" label="Order #" valueAlignment="right" valueFontWeight="medium" />
        <Formula label="Status" valueAlignment="right">{{status | title}}</Formula>
        <CurrencyValue dataSource="totalPrice" label="Total" valueAlignment="right" defaultCurrencyCode="USD" />
      </Fixed>
      <Expandable>
        <DateTimeValue dataSource="orderDate" label="Date" valueAlignment="right" format="MM/DD/YYYY" />

        <Spacer size="small" />

        <Block when="lineItems is not null">
          <Text fontWeight="medium">Line Items</Text>
          <List dataSource="lineItems">
            <Formula>{{productName}} (x{{quantity}}) - {{price}}</Formula>
          </List>
        </Block>
      </Expandable>
    </Drawer>
  </List>
</Card>
```

### UI Best Practices

1. **Labels:** Keep under 15 characters (use "Order #" not "Order Number")
2. **Fixed section:** 3-4 most important fields (ID, status, total)
3. **Expandable:** Secondary details (dates, addresses, line items)
4. **Addresses:** Use `displayStyle="stacked"` with `maxLines="3"`
5. **Conditional content:** Use `<Block when="...">` not `<Text when="...">`

### CHECKPOINT

```
Created UI template: customer-orders

Preview with sample data shows:
- 3 Orders (title)
- Drawer with order number, status, total
- Expands to show date and line items

Proceed to validation? [Yes/No]
```

---

## Phase 7: VERIFY - Test and Validate

**Goal:** Ensure app passes all tests and validates.

### Run Tests

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
- [ ] `appcfg validate` returns "valid"
- [ ] GraphQL schema types match data pull configs
- [ ] All @dataType names and versions align
- [ ] Test fixtures cover success, error, and edge cases

### CHECKPOINT

```
Validation Results:

✓ data/pull/customer_lookup: PASSED
✓ data/pull/orders: PASSED
✓ actions/cancel_order: PASSED
✓ actions/refund_order: PASSED
✓ ui/templates/customer-orders: PASSED
✓ appcfg validate: VALID

Your app is ready for deployment!

To deploy, run: /app-platform-deploy
```

---

## Quick Reference

### CLI Commands

| Task | Command |
|------|---------|
| Initialize app | `appcfg init --author <domain> --app-name <name> --root .` |
| Add data pull | `appcfg add data-pull <name> --data-type <type> --root .` |
| Add action | `appcfg add action <name> --root .` |
| Add UI template | `appcfg add ui-template <name> --root .` |
| Add auth header | `appcfg add auth-header <name> --root .` |
| Add OAuth | `appcfg add oauth <grant-type> --root .` |
| Test data pull | `appcfg test data-pull <name> --root .` |
| Test action | `appcfg test action <name> --root .` |
| Validate | `appcfg validate --root .` |
| Build | `appcfg build --root .` |

### Common Gotchas

| Issue | Fix |
|-------|-----|
| Empty arrays are truthy | Always check `len()`, not just existence |
| DateTime needs ISO8601 | Append `T00:00:00Z` to date-only values |
| Text/Spacer no `when` | Wrap in `<Block when="...">` |
| Objects show as map[] | Format nested objects to strings |
| Boolean conditions | Never use `= true`, use `is not null` |

---

## References

- [graphql-schema-patterns.md](references/graphql-schema-patterns.md) - Schema generation rules
- [data-pull-templates.md](references/data-pull-templates.md) - Data pull patterns
- [action-templates.md](references/action-templates.md) - Action patterns
- [ui-card-patterns.md](references/ui-card-patterns.md) - Flexible Card examples
- [authentication-patterns.md](references/authentication-patterns.md) - Auth setup
