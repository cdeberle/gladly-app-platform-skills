# Data Pull Templates

This reference covers patterns for data pull configurations and templates.

## Directory Structure

```
data/pull/<type_name>/
├── config.json                  # Required: Data type, HTTP method, dependencies
├── request_url.gtpl             # Required: URL template
├── request_body.gtpl            # Optional: For POST/PUT requests
├── response_transformation.gtpl # Optional: Transform API response
├── external_id.gtpl             # Required: Extract entity ID
├── external_parent_id.gtpl      # Required for child types: Extract parent ID
└── _test_/
    └── default/
        ├── integration.json     # Test: Integration config
        ├── customer.json        # Test: Customer context (root pulls)
        ├── externalData.json    # Test: Parent data (dependent pulls)
        ├── rawData.json         # Test: API response
        ├── expected_request_url.txt
        └── expected_response_transformation.json
```

---

## config.json

### Root Data Pull

```json
{
  "dataType": {
    "name": "acme_customer",
    "version": "1.0"
  },
  "httpMethod": "GET"
}
```

### Dependent Data Pull

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

### POST Request

```json
{
  "dataType": {
    "name": "acme_search_results",
    "version": "1.0"
  },
  "httpMethod": "POST",
  "contentType": "application/json"
}
```

### With Timeout

```json
{
  "dataType": {
    "name": "acme_order",
    "version": "1.0"
  },
  "httpMethod": "GET",
  "requestTimeout": 30000
}
```

---

## request_url.gtpl Patterns

### Root Data Pull (Customer Lookup by Email)

```go
{{- /* Look up customer by email address */}}
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

### Root Data Pull (Customer Lookup by Phone)

```go
{{- /* Look up customer by phone number */}}
{{- if eq (len .customer.phoneNumbers) 0}}
  {{- stop "Customer profile has no phone numbers"}}
{{- end}}

{{- $phone := ""}}
{{- if .customer.primaryPhoneNumber}}
  {{- $phone = .customer.primaryPhoneNumber.number}}
{{- else}}
  {{- $phone = (index .customer.phoneNumbers 0).number}}
{{- end}}

{{- $baseUrl := .integration.configuration.apiBaseUrl}}
{{$baseUrl}}/api/customers?phone={{urlquery $phone}}
```

### Dependent Data Pull (with Safety Checks)

```go
{{- /* Fetch orders for customer - ALWAYS include safety checks */}}
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

### Multiple Requests (Batch Lookup)

```go
{{- /* Generate multiple URLs - one per order to fetch products */}}
{{- range .externalData.acme_order}}
{{- range .lineItems}}
{{$.integration.configuration.apiBaseUrl}}/api/products/{{.productId}}
{{end}}
{{- end}}
```

---

## response_transformation.gtpl Patterns

### Handle Paginated vs Direct Array

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
{{- else if hasKey .rawData "results"}}
  {{- $items = .rawData.results}}
{{- end}}

{{- /* Handle empty array */}}
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

### Format Address Object to String

```go
{{- $formattedAddr := "" -}}
{{- if $item.address -}}
  {{- if kindIs "string" $item.address -}}
    {{- $formattedAddr = $item.address -}}
  {{- else -}}
    {{- $addrParts := list -}}
    {{- if $item.address.name }}{{ $addrParts = append $addrParts $item.address.name }}{{ end -}}
    {{- if $item.address.address1 }}{{ $addrParts = append $addrParts $item.address.address1 }}{{ end -}}
    {{- if $item.address.address2 }}{{ $addrParts = append $addrParts $item.address.address2 }}{{ end -}}
    {{- $cityLine := "" -}}
    {{- if $item.address.city }}{{ $cityLine = $item.address.city }}{{ end -}}
    {{- if $item.address.state -}}
      {{- if $cityLine }}{{ $cityLine = printf "%s, %s" $cityLine $item.address.state }}{{ else }}{{ $cityLine = $item.address.state }}{{ end -}}
    {{- end -}}
    {{- if $item.address.zip }}{{ $cityLine = printf "%s %s" $cityLine $item.address.zip }}{{ end -}}
    {{- if $cityLine }}{{ $addrParts = append $addrParts $cityLine }}{{ end -}}
    {{- if $item.address.country }}{{ $addrParts = append $addrParts $item.address.country }}{{ end -}}
    {{- $formattedAddr = $addrParts | join ", " -}}
  {{- end -}}
{{- end -}}
```

### Convert Date to DateTime

```go
{{- /* Convert date-only to DateTime with timezone */}}
"startDate": {{if $item.startDate}}{{if contains "T" $item.startDate}}"{{$item.startDate}}"{{else}}"{{$item.startDate}}T00:00:00Z"{{end}}{{else}}null{{end}}
```

### Handle Nullable Fields

```go
"note": {{if $item.note}}{{$item.note | toJson}}{{else}}null{{end}},
"tags": {{if $item.tags}}{{$item.tags | toJson}}{{else}}null{{end}},
"metadata": {{if $item.metadata}}{{$item.metadata | toJson}}{{else}}null{{end}}
```

---

## external_id.gtpl

Extract the unique identifier for the entity:

```go
{{.id}}
```

For numeric IDs (preserve formatting):

```go
{{int64 .id}}
```

---

## external_parent_id.gtpl

Extract the parent's ID from a child entity:

```go
{{.customerId}}
```

---

## Test Fixtures

### integration.json

```json
{
  "configuration": {
    "apiBaseUrl": "https://api.acme.com/v1"
  },
  "secrets": {
    "apiToken": "test-token-123"
  }
}
```

### customer.json (for root pulls)

```json
{
  "id": "gladly-profile-123",
  "primaryEmailAddress": "test@example.com",
  "emailAddresses": ["test@example.com", "alt@example.com"],
  "primaryPhoneNumber": {
    "number": "+15551234567",
    "type": "MOBILE"
  },
  "phoneNumbers": [
    {"number": "+15551234567", "type": "MOBILE"}
  ]
}
```

### externalData.json (for dependent pulls)

```json
{
  "acme_customer": [
    {
      "id": "cust_123",
      "email": "test@example.com",
      "fullName": "Test User"
    }
  ]
}
```

### rawData.json

```json
{
  "data": [
    {
      "id": "order_456",
      "orderNumber": "ORD-001",
      "status": "delivered",
      "totalPrice": 99.99,
      "orderDate": "2024-01-15T10:30:00Z",
      "customerId": "cust_123"
    }
  ]
}
```

---

## Common Safety Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `stop "message"` | Halt with error | `{{stop "No customer data"}}` |
| `urlquery` | URL encode value | `{{urlquery $email}}` |
| `default` | Provide fallback | `{{.value \| default ""}}` |
| `kindIs` | Check type | `{{if kindIs "slice" .data}}` |
| `hasKey` | Check map key | `{{if hasKey .rawData "data"}}` |
| `len` | Get length | `{{if eq (len $items) 0}}` |
| `toJson` | Safe JSON output | `{{$value \| toJson}}` |

---

## Common Gotchas

### Empty Arrays Are Truthy

```go
// BAD - empty array passes
{{if $items}}

// GOOD - check length
{{if gt (len $items) 0}}
```

### Missing Key Access

```go
// BAD - errors if key doesn't exist
{{.rawData.data}}

// GOOD - check key first
{{if hasKey .rawData "data"}}{{.rawData.data}}{{end}}
```

### Type Mismatch

```go
// BAD - fails if field is object not string
{{$item.address}}

// GOOD - check type first
{{if kindIs "string" $item.address}}{{$item.address}}{{end}}
```
