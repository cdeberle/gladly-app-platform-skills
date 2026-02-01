# UI Card Patterns

This reference covers Flexible Card XML patterns for Gladly Hero UI templates.

## Data Context and Scoping

### Understanding Data Context

Each component renders within a data context (scope). The context determines what data is accessible:

```xml
<StringValue dataSource="..." />      <!-- Scoped to top-level data object -->

<List dataSource="orders">            <!-- Scoped to top-level data -->
  <DateTimeValue dataSource="..." />  <!-- Scoped to orders[x] -->
  <List dataSource="items">           <!-- Iterates over orders[x].items -->
    <StringValue dataSource="..." />  <!-- Scoped to orders[x].items[y] -->
  </List>
</List>
```

### Special Variables

Each component has access to these special variables:

| Variable | Description | Availability |
|----------|-------------|--------------|
| `__data` | Current data context/scope | Always |
| `__root` | Top-level data object from GraphQL | Always |
| `__parent` | Parent scope of the list element | Inside List only |
| `__index` | Current iterator index (0-based) | Inside List only |

**Example using special variables:**

```xml
<List dataSource="orders">
  <!-- Access current item's field -->
  <StringValue dataSource="orderNumber" label="Order #" />

  <!-- Access parent scope -->
  <Formula label="Customer">{{__parent.customerName}}</Formula>

  <!-- Access root data -->
  <Formula label="Total Orders">{{__root.orders | length}}</Formula>

  <!-- Access index -->
  <Formula label="Order">{{__index + 1}} of {{__root.orders | length}}</Formula>
</List>
```

---

## Directory Structure

```
ui/templates/<card-name>/
├── config.json          # Required: Schema type, GraphQL type
├── flexible.card        # Required: XML template
└── _test_/
    └── default.json     # Test data
```

---

## config.json

```json
{
  "description": "Display customer orders",
  "graphqlSchema": "data",
  "graphqlType": "Customer"
}
```

- `graphqlSchema`: `"data"` or `"actions"`
- `graphqlType`: Root type from schema (e.g., `Customer`)

---

## Basic Card Structure

```xml
<?Template languageVersion="1.1" ?>
<Card title="Card Title">
  <!-- Content goes here -->
</Card>
```

### Dynamic Title

```xml
<Card title="{{ orders | length }} Orders">
```

---

## List with Drawer Pattern

The most common pattern for displaying collections:

```xml
<?Template languageVersion="1.1" ?>
<Card title="{{ orders | length }} Orders">
  <List dataSource="orders" separateItems="true">
    <Drawer initialState="expandFirst">
      <Fixed>
        <!-- Always visible: 3-4 key fields -->
        <StringValue dataSource="orderNumber" label="Order #" valueAlignment="right" valueFontWeight="medium" />
        <Formula label="Status" valueAlignment="right">{{status | title}}</Formula>
        <CurrencyValue dataSource="totalPrice" label="Total" valueAlignment="right" defaultCurrencyCode="USD" />
      </Fixed>
      <Expandable>
        <!-- Shown when expanded: details -->
        <DateTimeValue dataSource="orderDate" label="Date" valueAlignment="right" format="MM/DD/YYYY" />
        <StringValue dataSource="shippingAddress" label="Ship To" displayStyle="stacked" maxLines="3" />
      </Expandable>
    </Drawer>
  </List>
</Card>
```

### Drawer Initial States

- `expandFirst` - First item expanded, others collapsed
- `expanded` - All items expanded
- `collapsed` - All items collapsed

---

## Component Reference

### StringValue

Display text fields:

```xml
<!-- Right-aligned (default for short values) -->
<StringValue dataSource="orderNumber" label="Order #" valueAlignment="right" />

<!-- Stacked (for long content like addresses) -->
<StringValue dataSource="address" label="Address" displayStyle="stacked" maxLines="3" />

<!-- With font weight -->
<StringValue dataSource="name" label="Name" valueFontWeight="medium" />
```

### Formula

For computed values, combined fields, or transformations:

```xml
<!-- Simple transformation -->
<Formula label="Status">{{status | title}}</Formula>

<!-- Combined fields -->
<Formula label="Name">{{firstName}} {{lastName}}</Formula>

<!-- With conditional -->
<Formula label="Price">{{if salePrice}}{{salePrice}}{{else}}{{regularPrice}}{{end}}</Formula>
```

### CurrencyValue

For monetary values:

```xml
<CurrencyValue
  dataSource="totalPrice"
  label="Total"
  valueAlignment="right"
  currencyCodeSource="currency"
  defaultCurrencyCode="USD"
/>
```

### DateTimeValue

For date/time fields:

```xml
<DateTimeValue
  dataSource="orderDate"
  label="Date"
  valueAlignment="right"
  format="MM/DD/YYYY"
/>
```

Common formats:
- `MM/DD/YYYY` - US standard (01/15/2024)
- `DD/MM/YYYY` - International (15/01/2024)
- `YYYY-MM-DD` - ISO (2024-01-15)
- `MMM DD, YYYY` - Readable (Jan 15, 2024)

### BooleanValue

For true/false fields:

```xml
<BooleanValue
  dataSource="autoRenew"
  label="Auto-Renew"
  valueAlignment="right"
  displayTrueAs="Yes"
  displayFalseAs="No"
/>
```

---

## Conditional Content

### Block with when

Use `<Block when="...">` to conditionally show content:

```xml
<Block when="lineItems is not null">
  <Text fontWeight="medium">Line Items</Text>
  <List dataSource="lineItems">
    <Formula>{{productName}} (x{{quantity}})</Formula>
  </List>
</Block>
```

### Condition Syntax

| Condition | Meaning |
|-----------|---------|
| `field is not null` | Field exists and has value |
| `field is null` | Field is null or doesn't exist |
| `field == "value"` | String equals |
| `field != "value"` | String not equals |
| `field > 0` | Numeric comparison |

**Important:** Never use `field == true`. Use `field is not null` instead.

---

## Visual Hierarchy

### Spacers

Group related content with spacers:

```xml
<Expandable>
  <!-- Order details -->
  <DateTimeValue dataSource="orderDate" label="Date" valueAlignment="right" />
  <StringValue dataSource="fulfillmentStatus" label="Fulfillment" valueAlignment="right" />

  <Spacer size="small" />

  <!-- Payment details -->
  <StringValue dataSource="paymentMethod" label="Payment" valueAlignment="right" />

  <Spacer size="small" />

  <!-- Address -->
  <StringValue dataSource="shippingAddress" label="Ship To" displayStyle="stacked" maxLines="3" />
</Expandable>
```

### Section Headers

```xml
<Text fontWeight="medium">Line Items</Text>
<List dataSource="lineItems">
  <!-- items -->
</List>
```

---

## Label Best Practices

Keep labels under 15 characters to avoid truncation:

| Avoid (Too Long) | Use Instead |
|------------------|-------------|
| Payment Expiration | Exp. Date |
| Subscription Number | Sub # |
| Order Number | Order # |
| Confirmation Number | Conf # |
| Billing Address | Billing Addr |
| Shipping Address | Ship To |
| Phone Number | Phone |
| Email Address | Email |

---

## Display Styles

### valueAlignment="right" (Default)

Best for: Short values, numbers, dates, statuses

```
Order #                    ORD-12345
Total                        $149.99
Date                      01/15/2024
```

### displayStyle="stacked"

Best for: Long text, addresses, descriptions

```
Billing Address
John Smith, 123 Main St
San Francisco, CA 94102
```

---

## Nested Lists

For line items or nested data:

```xml
<Block when="lineItems is not null">
  <Spacer size="small" />
  <Text fontWeight="medium">Items</Text>
  <List dataSource="lineItems">
    <Formula>{{productName}} (x{{quantity}}) - ${{price}}</Formula>
  </List>
</Block>
```

---

## Links

```xml
<Link
  linkText="View in System"
  linkUrl="https://app.example.com/orders/{{orderNumber}}"
/>
```

---

## Complete Example

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
        <StringValue dataSource="paymentMethod" label="Payment" valueAlignment="right" />

        <Spacer size="small" />

        <StringValue dataSource="shippingAddress" label="Ship To" displayStyle="stacked" maxLines="3" />

        <Block when="lineItems is not null">
          <Spacer size="small" />
          <Text fontWeight="medium">Items ({{lineItems | length}})</Text>
          <List dataSource="lineItems">
            <Formula>{{productName}} x{{quantity}} - ${{price}}</Formula>
          </List>
        </Block>

        <Spacer size="small" />

        <Link linkText="View Order" linkUrl="https://admin.example.com/orders/{{orderNumber}}" />
      </Expandable>
    </Drawer>
  </List>
</Card>
```

---

## Test Data (default.json)

```json
{
  "orders": [
    {
      "orderNumber": "ORD-12345",
      "status": "delivered",
      "totalPrice": 149.99,
      "orderDate": "2024-01-15T10:30:00Z",
      "paymentMethod": "Visa 1234",
      "shippingAddress": "John Smith, 123 Main St, San Francisco, CA 94102",
      "lineItems": [
        {
          "productName": "Widget Pro",
          "quantity": 2,
          "price": 49.99
        },
        {
          "productName": "Gadget Basic",
          "quantity": 1,
          "price": 49.99
        }
      ]
    }
  ]
}
```

---

## Common Gotchas

1. **Text/Spacer don't support `when`** - Wrap in `<Block when="...">`
2. **Boolean conditions** - Never use `= true`, use `is not null`
3. **Long labels truncate** - Keep under 15 characters
4. **Addresses need stacked** - Use `displayStyle="stacked"` with `maxLines`
5. **Test with real data** - Fixtures may not show overflow issues

---

## Conditional Rendering (`when` Support)

### Components That Support `when`

| Component | Supports `when`? |
|-----------|------------------|
| Block | Yes |
| ColumnGroup | Yes |
| Section | Yes |
| Drawer | Yes |
| Divider | Yes |
| Image | Yes |
| List | Yes |
| StringValue | Yes |
| NumericValue | Yes |
| CurrencyValue | Yes |
| DateTimeValue | Yes |
| BooleanValue | Yes |
| Link | Yes |
| Formula | Yes |
| **Text** | **No** |
| **Spacer** | **No** |

### Workaround for Text and Spacer

Wrap in a `Block` element:

```xml
<!-- WRONG: Text doesn't support when -->
<Text when="cancelledAt is not null" fontWeight="medium">Cancelled</Text>

<!-- CORRECT: Wrap in Block -->
<Block when="cancelledAt is not null">
  <Text fontWeight="medium">Cancelled</Text>
</Block>
```

---

## Conditional Expression Reference

**Important:** Logical operators must be **lowercase**: `and`, `or`, `not`

| Expression | Meaning |
|------------|---------|
| `x < y`, `x > y` | Less than / greater than |
| `x == y` | Equals |
| `(x < y) and (z > q)` | Both conditions true |
| `(x < y) or (z > q)` | Either condition true |
| `not (x == y)` | Not equals |
| `x is defined` | Variable exists |
| `x is not defined` | Variable doesn't exist |
| `x is null` | Variable is null |
| `x is not null` | Variable is not null |
| `x is empty` | Empty object `{}` |
| `x is not empty` | Has attributes |

---

## Component Attribute Quick Reference

### Data Display Components

All data components share these common attributes:

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `dataSource` | String | Data binding path |
| `label` | String | Display label |
| `valueAlignment` | `left`, `right`, `center` | Value alignment |
| `displayStyle` | `default`, `stacked` | Layout style |
| `valueFontWeight` | `light`, `regular`, `medium`, `semiBold`, `bold` | Value emphasis |
| `valueFontSize` | `small`, `default`, `medium`, `large`, `extraLarge` | Value size |
| `maxLines` | 1-10 | Text truncation |
| `when` | Expression | Conditional rendering |

### CurrencyValue Specific

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `currencyCodeSource` | String | Data path for currency code |
| `defaultCurrencyCode` | String | Fallback currency (e.g., "USD") |
| `negativeValueStyle` | `standard`, `accounting` | -$12.95 vs ($12.95) |

### DateTimeValue Specific

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `format` | String | Date format mask (e.g., "MM/DD/YYYY") |

### BooleanValue Specific

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `displayTrueAs` | String | Text for true (e.g., "Yes") |
| `displayFalseAs` | String | Text for false (e.g., "No") |

### Link Specific

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `linkUrl` | String | URL (supports interpolation) |
| `linkText` | String | Display text |

### Image Specific

| Attribute | Values | Purpose |
|-----------|--------|---------|
| `dataSource` | String | URL data path |
| `defaultUrl` | String | Fallback URL |
| `size` | `auto`, `extraSmall`, `small`, `medium`, `large`, `extraLarge`, `{N}px` | Image size |
| `mode` | `fill`, `fit` | Scaling mode |
| `shape` | `circle`, `rectangle` | Image shape |

### Container Components

**Block:**
| Attribute | Values |
|-----------|--------|
| `padding` | `none`, `extraSmall`, `small`, `medium`, `large`, `extraLarge` |
| `border` | `true`, `false` |
| `alignment` | `left`, `right`, `center` |

**Section:**
| Attribute | Values |
|-----------|--------|
| `title` | String (supports interpolation) |
| `expandable` | `true`, `false` |
| `padding` | Same as Block |
| `border` | `true`, `false` |

**Drawer:**
| Attribute | Values |
|-----------|--------|
| `initialState` | `collapsed`, `expanded`, `expandFirst` |
| `border` | `true`, `false` |

**List:**
| Attribute | Values |
|-----------|--------|
| `dataSource` | String (required) |
| `separateItems` | `true`, `false` |
| `itemSpacing` | `none`, `extraSmall`, `small`, `medium`, `large`, `extraLarge` |

---

## Common Validation Errors

### Image Component

```xml
<!-- WRONG -->
<Image imageUrl="{{url}}" width="48px" height="48px" />

<!-- CORRECT -->
<Image dataSource="imageUrl" size="48px" />
```

### Column Width

Valid values: `"auto"`, `"stretch"`, `"{N}px"`, `"{N}%"`

```xml
<!-- WRONG -->
<Column width="fill">

<!-- CORRECT -->
<Column width="stretch">
```

### Formula Font Attributes

```xml
<!-- WRONG -->
<Formula fontSize="small" fontWeight="medium">{{value}}</Formula>

<!-- CORRECT -->
<Formula valueFontSize="small" valueFontWeight="medium">{{value}}</Formula>
```

---

## Text Interpolation

Use `{{ }}` delimiters to insert data into text:

```xml
<Card title="{{ orders | length }} Orders">
<Text>Order placed on {{ orderDate | date('F j, Y') }}</Text>
<Formula label="Total">{{ subtotal + tax + shipping }}</Formula>
```

### Available Filters

| Filter | Purpose | Example |
|--------|---------|---------|
| `length` | Array count | `{{ orders \| length }}` |
| `title` | Title case | `{{ status \| title }}` |
| `upper` | Uppercase | `{{ code \| upper }}` |
| `lower` | Lowercase | `{{ name \| lower }}` |
| `date` | Format date | `{{ orderDate \| date('M/D/Y') }}` |
| `format_currency` | Format money | `{{ total \| format_currency }}` |

---

## Testing Checklist

Before deploying card templates:

- [ ] Test with maximum length data (long names, addresses)
- [ ] Test with minimum data (null/empty fields)
- [ ] Verify labels don't truncate
- [ ] Verify values don't overflow
- [ ] Check drawer expand/collapse behavior
- [ ] Verify currency formatting
- [ ] Verify date formatting
- [ ] Test conditional blocks with missing data
