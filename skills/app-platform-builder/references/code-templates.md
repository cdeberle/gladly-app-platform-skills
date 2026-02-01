# Code Templates Reference

Ready-to-use code templates for common App Platform patterns. Copy and adapt these for your implementation.

## Authentication Templates

### Bearer Token Header
`authentication/headers/Authorization.gtpl`:
```go
Bearer {{.integration.secrets.apiToken}}
```

### API Key Header
`authentication/headers/X-API-Key.gtpl`:
```go
{{.integration.secrets.apiKey}}
```

---

## Data Schema Templates

### Basic Data Schema
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

### Actions Schema
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

---

## Data Pull Templates

### Root Data Pull config.json
```json
{
  "dataType": {
    "name": "acme_customer",
    "version": "1.0"
  },
  "httpMethod": "GET"
}
```

### Root Data Pull request_url.gtpl (Customer Lookup)
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

### Root Data Pull external_id.gtpl
```go
{{.id}}
```

### Dependent Data Pull config.json
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

### Dependent Data Pull request_url.gtpl (with Safety Checks)
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

### Dependent Data Pull external_parent_id.gtpl
```go
{{.customerId}}
```

### Response Transformation Template
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

---

## Action Templates

### Action config.json
```json
{
  "httpMethod": "POST",
  "contentType": "application/json"
}
```

### Action request_url.gtpl
```go
{{.integration.configuration.apiBaseUrl}}/api/orders/{{urlquery .inputs.orderId}}/cancel
```

### Action request_body.gtpl
```json
{
  "reason": "{{.inputs.reason}}"
}
```

### Action response_transformation.gtpl
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

---

## UI Template

### Basic Card with Drawer
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

### UI config.json
```json
{
  "description": "Display customer orders",
  "graphqlSchema": "data",
  "graphqlType": "Customer"
}
```
