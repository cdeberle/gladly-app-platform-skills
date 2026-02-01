# Go Template Reference for Gladly App Platform

This document covers Go text template fundamentals, common pitfalls, and best practices for App Platform development.

## Template Language Basics

App Platform uses [Go text templates](https://pkg.go.dev/text/template) with [Sprig functions](https://masterminds.github.io/sprig) for generating request URLs, bodies, and transforming responses.

### Range Actions

Template context data fields referenced within a `range` action that are not relative to the current iteration must use `$` to reference the root context:

```go
{{ range $i, $customer := .customers}}
Customer {{ $i }} of {{ len $.customers }}
Name: {{ $customer.firstName }} {{ $customer.lastName }}
{{ end }}
```

### Sprig Function Clarifications

#### contains

Returns `true` when the first string is contained in the second:

```go
{{ contains "hello" "hello world" }}  {{/* returns true */}}
```

#### Date Functions

`date`, `dateInZone`, `toDate`, `mustToDate` use Go time layout format. See [Go time constants](https://pkg.go.dev/time#pkg-constants) for examples.

---

## Critical Gotchas

### 1. Empty Arrays Are Truthy

**The Problem:** In Go templates, an empty array `[]` is **not** falsy.

```go
// BROKEN - empty array passes this check
{{- $items := .rawData.data | default list}}
{{- if $items}}
  // This executes even when $items is []
{{- end}}
```

When an API returns `{"data": [], "pagination": {"total": 0}}`, the template tries to iterate over an empty array.

**The Fix:** Always use explicit length checks:

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
[
{{- range $i, $item := $items}}
  // Process items
{{- end}}
]
{{- end}}
```

---

### 2. Type Checking Before Field Access

**The Problem:** APIs may return fields as either strings or objects:

```json
// Sometimes this:
{"paymentMethod": "Visa 1234"}

// Sometimes this:
{"paymentMethod": {"brand": "Visa", "last4": "1234"}}
```

Accessing `.paymentMethod.brand` when it's a string causes errors.

**The Fix:** Always check type before accessing nested fields:

```go
{{- $paymentMethod := "" -}}
{{- if $item.paymentMethod -}}
  {{- if kindIs "string" $item.paymentMethod -}}
    {{- $paymentMethod = $item.paymentMethod -}}
  {{- else -}}
    {{- if $item.paymentMethod.brand -}}
      {{- $paymentMethod = $item.paymentMethod.brand -}}
      {{- if $item.paymentMethod.last4 -}}
        {{- $paymentMethod = printf "%s %s" $paymentMethod $item.paymentMethod.last4 -}}
      {{- end -}}
    {{- end -}}
  {{- end -}}
{{- end -}}
```

---

### 3. Null vs Empty String Output

**The Problem:** GraphQL distinguishes between `null` and `""`. Wrong output causes schema validation errors.

**The Fix:** Use conditional JSON output:

```go
// For optional strings - output null if empty
"fieldName": {{if $value}}{{$value | toJson}}{{else}}null{{end}}

// For required strings - always output string
"fieldName": "{{$value}}"

// For optional objects - output null or valid JSON
"nestedField": {{if $nested}}{{$nested | toJson}}{{else}}null{{end}}
```

---

### 4. Object-to-String Formatting

**The Problem:** Nested objects display as `map[key:value]` when output directly.

**Address Formatting Pattern:**

```go
{{- $formattedAddr := "" -}}
{{- if $item.address -}}
  {{- if kindIs "string" $item.address -}}
    {{- $formattedAddr = $item.address -}}
  {{- else -}}
    {{- $addrParts := list -}}
    {{- if $item.address.name }}{{ $addrParts = append $addrParts $item.address.name }}{{ end -}}
    {{- if $item.address.address1 }}{{ $addrParts = append $addrParts $item.address.address1 }}{{ end -}}
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

**Payment Method Formatting Pattern:**

```go
{{- $paymentMethod := "" -}}
{{- $paymentExpiration := "" -}}
{{- if $item.paymentMethod -}}
  {{- if kindIs "string" $item.paymentMethod -}}
    {{- $paymentMethod = $item.paymentMethod -}}
  {{- else -}}
    {{- if $item.paymentMethod.brand -}}
      {{- $paymentMethod = $item.paymentMethod.brand -}}
      {{- if $item.paymentMethod.last4 -}}
        {{- $paymentMethod = printf "%s %s" $paymentMethod $item.paymentMethod.last4 -}}
      {{- end -}}
    {{- else if $item.paymentMethod.type -}}
      {{- $paymentMethod = $item.paymentMethod.type -}}
    {{- end -}}
    {{- if and $item.paymentMethod.expiryMonth $item.paymentMethod.expiryYear -}}
      {{- $paymentExpiration = printf "%v/%v" $item.paymentMethod.expiryMonth $item.paymentMethod.expiryYear -}}
    {{- end -}}
  {{- end -}}
{{- end -}}
```

---

### 5. Using `stop` for Error Handling

**The Problem:** Silent failures make debugging difficult. When dependent data is missing, you want clear error messages.

**The Fix:** Use `stop` in `request_url.gtpl`:

```go
{{- $customers := .externalData.demo_crm_customer}}
{{- if not $customers}}
  {{- stop "No customer data available - demo_crm_customer not found"}}
{{- end}}
{{- if eq (len $customers) 0}}
  {{- stop "No customer found in demo_crm_customer"}}
{{- end}}
{{- $customerId := (index $customers 0).id}}
{{- if not $customerId}}
  {{- stop "Customer ID is empty"}}
{{- end}}
```

---

### 6. JSON Array Output Pattern

**Complete Pattern for Response Transformation:**

```go
{{- /*
Transform API response into array of objects.
Handles: direct array, paginated {"data": [...]} wrapper, empty results
*/ -}}

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

{{- /* Handle empty array */}}
{{- if eq (len $items) 0}}
[]
{{- else}}

{{- /* Process items */}}
[
{{- range $i, $item := $items}}
{{- if $i}},{{end}}
{
  "id": "{{$item.id}}",
  "name": {{$item.name | toJson}},
  "status": "{{$item.status}}"
}
{{- end}}
]
{{- end}}
{{- end}}
```

---

## Type Checking Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `kindIs "slice"` | Check if value is an array | `{{if kindIs "slice" .data}}` |
| `kindIs "map"` | Check if value is an object | `{{if kindIs "map" .field}}` |
| `kindIs "string"` | Check if value is a string | `{{if kindIs "string" .field}}` |
| `kindIs "bool"` | Check if value is a boolean | `{{if kindIs "bool" .field}}` |
| `kindIs "float64"` | Check if value is a number | `{{if kindIs "float64" .field}}` |
| `kindIs "int"` | Check if value is an integer | `{{if kindIs "int" .field}}` |
| `hasKey` | Check if map has key | `{{if hasKey .rawData "data"}}` |
| `len` | Get array/map length | `{{if eq (len $items) 0}}` |
| `list` | Create empty list | `{{$items := list}}` |

---

## Common Utility Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `urlquery` | URL-encode a string | `{{urlquery $customerId}}` |
| `toJson` | Convert to JSON string | `{{$value \| toJson}}` |
| `default` | Provide default value | `{{.field \| default "fallback"}}` |
| `join` | Join array with separator | `{{$parts \| join ", "}}` |
| `printf` | Format string | `{{printf "%s %s" $first $last}}` |
| `index` | Access array element | `{{(index $customers 0).id}}` |
| `stop` | Stop with error message | `{{stop "Error message"}}` |
| `fail` | Fail with error | `{{fail "Fatal error"}}` |

---

## Gladly-Specific Template Functions

These functions are provided by the Gladly App Platform runtime in addition to standard Go templates and Sprig functions.

### XML Parsing

| Function | Description | Returns |
|----------|-------------|---------|
| `fromXml` | Parses XML string into Gladly XML DOM | XML Document object (nil if parsing fails) |
| `mustFromXml` | Parses XML string, errors on failure | XML Document object |

```go
{{- $doc := fromXml .response.body }}
{{- $root := $doc.Root }}
```

See [XML DOM Handling](data-pull-templates.md#xml-dom-handling) for working with parsed XML.

### JWT Generation

| Function | Description | Returns |
|----------|-------------|---------|
| `genJwt` | Generates JWT with claims, algorithm, and key | JWT string (empty on error) |
| `mustGenJwt` | Generates JWT, errors on failure | JWT string |

**Parameters:** claims (map), algorithm (string), base64-encoded key (string)

**Supported algorithms:** RS* (RSA), ES* (ECDSA), HS* (HMAC - e.g., "HS256")

```go
{{- $claims := dict "iss" "my-app" "exp" (now | unixEpoch | add 3600) }}
{{- $jwt := genJwt $claims "HS256" .integration.secrets.jwtKey }}
```

### HMAC Signing

| Function | Description | Returns |
|----------|-------------|---------|
| `hmacSha256` | Computes SHA256 HMAC | HMAC hash (empty on error) |
| `mustHmacSha256` | Computes SHA256 HMAC, errors on failure | HMAC hash |

**Parameters:** signing key, salt, string to sign

```go
{{- $signature := hmacSha256 .integration.secrets.signingKey "" $stringToSign }}
```

### URL Parsing

| Function | Description | Returns |
|----------|-------------|---------|
| `url` | Parses URL into url.URL structure | url.URL object (empty on error) |
| `mustUrl` | Parses URL, errors on failure | url.URL object |

```go
{{- $parsed := url .oauth.request.url }}
{{- $code := index $parsed.Query "code" 0 }}
```

See [Go url.URL documentation](https://pkg.go.dev/net/url#URL) for available fields.

### Encoding/Decoding

| Function | Description |
|----------|-------------|
| `hexEnc` | Encodes bytes to hexadecimal string |
| `hexDec` | Decodes hexadecimal string to bytes |
| `urlb64enc` | Base64 encodes using RFC 4648 (URL-safe) |
| `urlb64dec` | Base64 decodes using RFC 4648 (URL-safe) |

```go
{{- $encoded := hexEnc $bytes }}
{{- $decoded := hexDec "48656c6c6f" }}
{{- $urlSafe := urlb64enc $data }}
```

### Flow Control

| Function | Description | Use Case |
|----------|-------------|----------|
| `stop` | Halts template with message | Missing required data (not errors) |
| `fail` | Fails template with error | Unexpected error conditions |

```go
{{- if not .customer.primaryEmailAddress }}
  {{- stop "Customer has no email address" }}
{{- end }}
```

### Debug Functions (Development Only)

| Function | Description |
|----------|-------------|
| `consolePrintln` | Prints line to console |
| `consolePrintf` | Prints formatted output to console |

**Important:** Remove all debug functions before validating and building apps for deployment.

```go
{{- consolePrintln "Debug: processing customer" }}
{{- consolePrintf "Order count: %d" (len $orders) }}
```

---

## External References

- [Go text/template](https://pkg.go.dev/text/template) - Base template language
- [Sprig functions](https://masterminds.github.io/sprig/) - Extended template functions
- [appcfg CLI](https://github.com/gladly/app-platform-appcfg-cli) - CLI tool and documentation

---

## Quick Reference

| Issue | Bad Pattern | Good Pattern |
|-------|-------------|--------------|
| Empty array | `{{if .data}}` | `{{if gt (len .data) 0}}` |
| Key exists | `{{if .rawData.data}}` | `{{if hasKey .rawData "data"}}` |
| Type check | `{{.field.nested}}` | `{{if kindIs "map" .field}}{{.field.nested}}{{end}}` |
| Null output | `"field": ""` | `"field": {{if $val}}{{$val\|toJson}}{{else}}null{{end}}` |
| Error stop | (silent fail) | `{{stop "Error message"}}` |
| List init | `{{$x := .data \| default list}}` | `{{$x := list}}{{if .data}}{{$x = .data}}{{end}}` |

---

## DateTime Handling

The `DateTime` scalar requires ISO8601 format. Date-only API values must be converted:

```go
{{- /* Convert date-only string to DateTime */}}
{{if $value}}{{if contains "T" $value}}"{{$value}}"{{else}}"{{$value}}T00:00:00Z"{{end}}{{else}}null{{end}}
```

**Common fields requiring conversion:**
- `startDate`, `endDate` (subscriptions)
- `memberSince`, `tierExpirationDate` (loyalty programs)
- `checkInDate`, `checkOutDate` (hotel stays)

---

## Multiple Request Patterns

### Multiple URLs

A URL template can create multiple requests by outputting each URL on a new line:

```go
{{- range .customer.emailAddresses -}}
https://api.example.com/customers?email={{urlquery .}}
{{ end -}}
```

### Multiple Bodies

Use `#body` marker for multiple request bodies:

```go
{{- range .customer.emailAddresses -}}
#body
{
    "emailAddress": "{{.}}"
}
{{ end -}}
```

**Important:** Template actions after URL or body text must not strip the newline. The `{{` that follows should not include `-` for stripping preceding whitespace.
