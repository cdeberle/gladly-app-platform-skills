# GraphQL Schema Patterns

This reference covers patterns for generating GraphQL schemas from JSON API payloads.

## JSON to GraphQL Type Mapping

### Primitive Types

| JSON Pattern | GraphQL Type | Example Field |
|--------------|--------------|---------------|
| `"field": "text"` | `String` | `name: String` |
| `"field": ""` (empty) | `String` | `note: String` |
| `"field": 123` | `Int` | `quantity: Int` |
| `"field": 12.34` | `Float` | `rating: Float` |
| `"field": true/false` | `Boolean` | `isActive: Boolean` |
| `"field": null` | Optional type | `note: String` (nullable) |

### Special Types

| JSON Pattern | GraphQL Type | Detection Rule |
|--------------|--------------|----------------|
| `"createdAt": "2024-01-15T..."` | `DateTime` | Field ends with `At`, `Date`, `Time`, or contains ISO8601 |
| `"totalPrice": 99.99` | `Currency` | Field is `price`, `total`, `amount`, `cost`, or ends with `Price` |
| `"id": "abc123"` | `ID!` | Field is `id` and is required |

### Collection Types

| JSON Pattern | GraphQL Type | Notes |
|--------------|--------------|-------|
| `"tags": ["a", "b"]` | `[String]` | Array of primitives |
| `"orders": [{...}]` | `[Order]` | Array of objects - needs separate type |
| `"address": {...}` | `Address` | Nested object - embedded type |

---

## @dataType Directive

The `@dataType` directive registers a type for external data retrieval.

### Syntax

```graphql
type TypeName @dataType(name: "prefix_typename", version: "1.0") {
  id: ID!
  # ... fields
}
```

### Naming Convention

- Use snake_case for type names
- Prefix with app identifier to avoid conflicts
- Example: `acme_customer`, `acme_order`, `acme_subscription`

### Version

- Start with `"1.0"` for new types
- Increment minor version for non-breaking changes
- Major version changes require new config creation

### Example

```graphql
type Customer @dataType(name: "acme_customer", version: "1.0") {
  id: ID!
  email: String!
  fullName: String
  createdAt: DateTime
  lifetimeValue: Currency
}
```

---

## Relationship Directives

### @parentId - Child References Parent

Use when the **child entity has a field** containing the parent's ID.

```json
// Order has customerId field
{
  "id": "order_123",
  "customerId": "cust_456",
  "total": 99.99
}
```

```graphql
type Order @dataType(name: "acme_order", version: "1.0") {
  id: ID!
  customerId: String
  total: Currency
}

type Customer @dataType(name: "acme_customer", version: "1.0") {
  id: ID!
  orders: [Order] @parentId(template: "{{.id}}")
}
```

**How it works:**
1. Customer data pull returns customer with `id: "cust_456"`
2. Orders data pull fetches orders for customer
3. Platform matches orders where `external_parent_id` (from `customerId`) equals customer's `id`

**Requires:** `external_parent_id.gtpl` in the child's data pull directory.

### @childIds - Parent References Children

Use when the **parent entity has an array** of child IDs.

```json
// Customer has orderIds array
{
  "id": "cust_456",
  "orderIds": ["order_1", "order_2", "order_3"]
}
```

```graphql
type Order @dataType(name: "acme_order", version: "1.0") {
  id: ID!
  total: Currency
}

type Customer @dataType(name: "acme_customer", version: "1.0") {
  id: ID!
  orders: [Order] @childIds(template: "{{.orderIds | join \",\"}}")
}
```

**How it works:**
1. Customer data pull returns customer with `orderIds: ["order_1", "order_2"]`
2. Template outputs: `order_1,order_2`
3. Platform matches orders where `external_id` is in this list

### Choosing Between @parentId and @childIds

| Scenario | Use |
|----------|-----|
| Child has `parentId` field | `@parentId` |
| Parent has array of child IDs | `@childIds` |
| Separate API calls for child data | `@parentId` (with dependent pull) |
| All data in single response | Either works |

---

## Actions Schema

### @action Directive

Maps GraphQL mutations to action handlers.

```graphql
type Mutation {
  actionName(input: Type!, ...): ResultType @action(name: "action_folder_name")
}
```

### Complete Actions Schema Example

```graphql
type Mutation {
  cancelOrder(orderId: String!, reason: String!): ActionResult @action(name: "cancel_order")
  refundOrder(orderId: String!, amount: Float!, reason: String): ActionResult @action(name: "refund_order")
  updateSubscription(subscriptionId: String!, status: String!): ActionResult @action(name: "update_subscription")
}

type ActionResult {
  status: String!
  message: String
  data: ActionData
}

type ActionData {
  orderId: String
  refundAmount: Float
  newStatus: String
}
```

### Input Types

For complex inputs, define input types:

```graphql
input LineItemInput {
  productId: String!
  quantity: Int!
}

type Mutation {
  createReturn(orderId: String!, lineItems: [LineItemInput!]!): ActionResult @action(name: "create_return")
}
```

---

## Query Type

Define the root query for data access:

```graphql
type Query {
  customer: Customer      # Root lookup
  orders: [Order]         # Direct access to orders
  subscriptions: [Subscription]
}
```

---

## Complete Schema Example

```graphql
# Child types first (referenced by parent types)
type LineItem {
  id: String
  productId: String
  productName: String
  quantity: Int
  unitPrice: Currency
  totalPrice: Currency
}

type Order @dataType(name: "acme_order", version: "1.0") {
  id: ID!
  orderNumber: String!
  status: String
  totalPrice: Currency
  shippingPrice: Currency
  orderDate: DateTime
  customerId: String
  shippingAddress: String
  billingAddress: String
  lineItems: [LineItem]
}

type Subscription @dataType(name: "acme_subscription", version: "1.0") {
  id: ID!
  subscriptionNumber: String!
  status: String
  plan: String
  monthlyPrice: Currency
  startDate: DateTime
  nextBillingDate: DateTime
  customerId: String
  autoRenew: Boolean
}

# Root type last (references child types)
type Customer @dataType(name: "acme_customer", version: "1.0") {
  id: ID!
  email: String!
  fullName: String
  phone: String
  createdAt: DateTime
  lifetimeValue: Currency
  orders: [Order] @parentId(template: "{{.id}}")
  subscriptions: [Subscription] @parentId(template: "{{.id}}")
}

type Query {
  customer: Customer
  orders: [Order]
  subscriptions: [Subscription]
}
```

---

## DateTime Handling

### Date-Only Values

If API returns date without time, append timezone:

```go
// In response_transformation.gtpl
"startDate": {{if $item.startDate}}"{{$item.startDate}}T00:00:00Z"{{else}}null{{end}}
```

### Breaking Change Warning

Changing a field from `String` to `DateTime` is a **breaking change** requiring:
- Major version bump (1.x.x → 2.0.0)
- New configuration creation

---

## Enum Considerations

### When to Use Enums

| Use Enum | Use String |
|----------|------------|
| Values are stable and documented | API may add new values |
| Type safety is important | Values are user-generated |
| Limited, known set of values | Dynamic/flexible values |

### Enum Casing

Match API response values to avoid transformation:

```graphql
# RECOMMENDED: Match API values
enum OrderStatus {
  pending
  confirmed
  shipped
  delivered
  cancelled
}

# AVOID: Requires transformation
enum OrderStatus {
  PENDING
  CONFIRMED
}
```

### Breaking Change Warning

Changing a field from `String` to enum is a **breaking change**.
