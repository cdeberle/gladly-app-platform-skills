# Action Templates

This reference covers patterns for action (mutation) configurations and templates.

## Directory Structure

```
actions/<action_name>/
├── config.json                  # Required: HTTP method, content type
├── request_url.gtpl             # Required: URL template
├── request_body.gtpl            # Required for POST/PUT: Request payload
├── response_transformation.gtpl # Required: Transform response to result
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

---

## Context Data Structures

### request_url.gtpl and request_body.gtpl Context

```jsonc
{
    "integration": {
        // Dynamic configuration associated with the app
        "configuration": {},
        // Secrets for making REST calls
        "secrets": {}
    },
    // Input values from the GraphQL mutation arguments
    "inputs": {}
}
```

The structure of `inputs` corresponds to the GraphQL field arguments. For example:

```graphql
type Mutation {
  cancelOrder(orderId: String!, reason: String!): ActionResult @action(name: "cancel_order")
}
```

Would provide:
```json
{
  "inputs": {
    "orderId": "order_123",
    "reason": "Customer requested"
  }
}
```

### response_transformation.gtpl Context

```jsonc
{
    "integration": {
        "configuration": {},
        "secrets": {}
    },
    // The inputs for the GraphQL field
    "inputs": {},
    "request": {
        // The request URL from request_url.gtpl
        "url": "",
        // The HTTP method from config.json
        "method": "",
        // Headers from authentication/headers/
        "headers": {
            "header_name": ["header value"]
        },
        // Raw bytes of the request body
        "body": ""
    },
    "response": {
        // HTTP response status text
        "status": "",
        // HTTP response status code
        "statusCode": 200,
        // Response headers
        "headers": {
            "response_header_name": ["header value"]
        },
        // Raw bytes of the response body
        "body": ""
    },
    // The raw response data
    // - For JSON: object or array
    // - For XML: Gladly-specific XML DOM
    // - For rawResponse=true: byte array
    "rawData": {}
}
```

**Important:** Action response transformations MUST handle non-success status codes. The platform does not filter errors for actions like it does for data pulls. Authentication errors (401/403) are handled automatically.

### config.json with rawResponse

Set `rawResponse: true` when the response Content-Type is not JSON or XML:

```json
{
    "httpMethod": "POST",
    "contentType": "application/json",
    "rawResponse": true
}
```

---

## config.json

### POST Action

```json
{
  "httpMethod": "POST",
  "contentType": "application/json"
}
```

### PUT Action

```json
{
  "httpMethod": "PUT",
  "contentType": "application/json"
}
```

### DELETE Action (no body)

```json
{
  "httpMethod": "DELETE"
}
```

---

## request_url.gtpl Patterns

### Simple URL with Input

```go
{{.integration.configuration.apiBaseUrl}}/api/orders/{{urlquery .inputs.orderId}}/cancel
```

### URL with Multiple Parameters

```go
{{.integration.configuration.apiBaseUrl}}/api/customers/{{urlquery .inputs.customerId}}/subscriptions/{{urlquery .inputs.subscriptionId}}
```

### URL with Query Parameters

```go
{{.integration.configuration.apiBaseUrl}}/api/orders/{{urlquery .inputs.orderId}}/refund?notify={{.inputs.sendNotification}}
```

---

## request_body.gtpl Patterns

### Simple JSON Body

```json
{
  "reason": "{{.inputs.reason}}"
}
```

### Multiple Fields

```json
{
  "orderId": "{{.inputs.orderId}}",
  "reason": "{{.inputs.reason}}",
  "amount": {{.inputs.amount}},
  "sendNotification": {{.inputs.sendNotification | default false}}
}
```

### Array Input

```json
{
  "orderId": "{{.inputs.orderId}}",
  "lineItemIds": {{.inputs.lineItemIds | toJson}}
}
```

### Conditional Fields

```go
{
  "orderId": "{{.inputs.orderId}}"
{{- if .inputs.reason}},
  "reason": {{.inputs.reason | toJson}}
{{- end}}
{{- if .inputs.amount}},
  "refundAmount": {{.inputs.amount}}
{{- end}}
}
```

### Nested Object

```json
{
  "orderId": "{{.inputs.orderId}}",
  "shippingAddress": {
    "name": "{{.inputs.recipientName}}",
    "address1": "{{.inputs.address1}}",
    "city": "{{.inputs.city}}",
    "state": "{{.inputs.state}}",
    "zip": "{{.inputs.zip}}"
  }
}
```

---

## response_transformation.gtpl Patterns

### Basic Success/Error Handling

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
  "message": {{if .rawData.message}}{{.rawData.message | toJson}}{{else}}"Operation completed successfully."{{end}}
}
{{- end}}
```

### With Return Data

```go
{{- if ge .response.statusCode 400 }}
{
  "status": "error",
  "message": {{if .rawData.error}}{{.rawData.error | toJson}}{{else}}"Request failed"{{end}}
}
{{- else }}
{
  "status": "success",
  "message": "Order cancelled successfully.",
  "data": {
    "orderId": "{{.rawData.orderId}}",
    "newStatus": "{{.rawData.status}}",
    "cancelledAt": "{{.rawData.cancelledAt}}"
  }
}
{{- end}}
```

### Multiple Error Codes

```go
{{- if eq .response.statusCode 404 }}
{
  "status": "error",
  "message": "Order not found."
}
{{- else if eq .response.statusCode 409 }}
{
  "status": "error",
  "message": "Order has already been {{.rawData.currentStatus}}."
}
{{- else if eq .response.statusCode 422 }}
{
  "status": "error",
  "message": {{if .rawData.validationErrors}}{{index .rawData.validationErrors 0 | toJson}}{{else}}"Validation failed"{{end}}
}
{{- else if eq .response.statusCode 403 }}
{
  "status": "error",
  "message": "Not authorized to perform this action."
}
{{- else if ge .response.statusCode 400 }}
{
  "status": "error",
  "message": {{if .rawData.message}}{{.rawData.message | toJson}}{{else}}"Request failed"{{end}}
}
{{- else }}
{
  "status": "success",
  "message": "Operation completed."
}
{{- end}}
```

### Business Logic Errors from 200 Response

Some APIs return 200 with error in body:

```go
{{- if .rawData.success }}
{
  "status": "success",
  "message": {{if .rawData.message}}{{.rawData.message | toJson}}{{else}}"Completed"{{end}}
}
{{- else }}
{
  "status": "error",
  "message": {{if .rawData.errorMessage}}{{.rawData.errorMessage | toJson}}{{else}}"Operation failed"{{end}}
}
{{- end}}
```

---

## Test Fixtures

### success/inputs.json

```json
{
  "orderId": "order_123",
  "reason": "Customer requested cancellation"
}
```

### success/rawData.json

```json
{
  "orderId": "order_123",
  "status": "cancelled",
  "cancelledAt": "2024-01-15T10:30:00Z",
  "message": "Order has been cancelled"
}
```

### success/response.json

```json
{
  "statusCode": 200,
  "status": "OK",
  "headers": {}
}
```

### success/expected_response_transformation.json

```json
{
  "status": "success",
  "message": "Order has been cancelled"
}
```

### error/inputs.json

```json
{
  "orderId": "nonexistent_order",
  "reason": "Test error case"
}
```

### error/rawData.json

```json
{
  "error": "Order not found",
  "code": "ORDER_NOT_FOUND"
}
```

### error/response.json

```json
{
  "statusCode": 404,
  "status": "Not Found",
  "headers": {}
}
```

### error/expected_response_transformation.json

```json
{
  "status": "error",
  "message": "Order not found."
}
```

---

## GraphQL Actions Schema

### Basic Action

```graphql
type Mutation {
  cancelOrder(orderId: String!, reason: String!): ActionResult @action(name: "cancel_order")
}

type ActionResult {
  status: String!
  message: String
}
```

### Action with Return Data

```graphql
type Mutation {
  refundOrder(orderId: String!, amount: Float!, reason: String): RefundResult @action(name: "refund_order")
}

type RefundResult {
  status: String!
  message: String
  refundId: String
  refundedAmount: Float
}
```

### Action with Complex Input

```graphql
input LineItemInput {
  lineItemId: String!
  quantity: Int!
  reason: String
}

type Mutation {
  createReturn(orderId: String!, lineItems: [LineItemInput!]!): ActionResult @action(name: "create_return")
}
```

---

## Common Patterns

### Input Validation in request_body.gtpl

```go
{{- if not .inputs.orderId}}
  {{- fail "orderId is required"}}
{{- end}}
{{- if and (not .inputs.reason) (eq .inputs.actionType "cancel")}}
  {{- fail "reason is required for cancellation"}}
{{- end}}
{
  "orderId": "{{.inputs.orderId}}",
  "reason": {{.inputs.reason | toJson}}
}
```

### Safe String Escaping

Always use `toJson` for string values that might contain special characters:

```go
{
  "reason": {{.inputs.reason | toJson}},
  "note": {{.inputs.note | toJson}}
}
```

### Handling Optional Inputs

```go
{
  "orderId": "{{.inputs.orderId}}"
{{- if .inputs.amount}},
  "amount": {{.inputs.amount}}
{{- end}}
{{- if .inputs.reason}},
  "reason": {{.inputs.reason | toJson}}
{{- end}}
}
```

---

## HTTP Status Code Reference

| Code | Meaning | Typical Response |
|------|---------|------------------|
| 200 | Success | Return success result |
| 201 | Created | Return success with new ID |
| 400 | Bad Request | Validation error |
| 401 | Unauthorized | Auth error (handled by platform) |
| 403 | Forbidden | Permission denied |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | State conflict (already cancelled) |
| 422 | Unprocessable | Business logic error |
| 500+ | Server Error | Generic server error message |
