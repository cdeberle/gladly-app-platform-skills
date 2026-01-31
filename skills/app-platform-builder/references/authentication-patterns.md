# Authentication Patterns

This reference covers authentication setup patterns for App Platform apps.

## Authentication Directory Structure

```
authentication/
├── headers/                    # Static auth headers
│   ├── Authorization.gtpl      # Bearer token
│   └── X-API-Key.gtpl          # API key
├── oauth/                      # OAuth flows
│   ├── config.json             # Grant type config
│   ├── access_token/           # Token request
│   │   ├── config.json
│   │   ├── request_url.gtpl
│   │   └── request_body.gtpl
│   └── refresh_token/          # Token refresh (if needed)
│       ├── config.json
│       ├── request_url.gtpl
│       └── request_body.gtpl
└── request_signing/            # Request signature headers
    └── X-Signature.gtpl
```

---

## Configuration vs Secrets

| Type | Use For | Stored In |
|------|---------|-----------|
| Configuration | Non-sensitive settings | `integration.configuration` |
| Secrets | Sensitive credentials | `integration.secrets` |

### Configuration Examples
- API base URL
- Client ID (public)
- Tenant ID
- Region/environment

### Secrets Examples
- API tokens
- Client secrets
- Passwords
- Private keys

---

## Bearer Token Authentication

### Header Template

`authentication/headers/Authorization.gtpl`:

```go
Bearer {{.integration.secrets.apiToken}}
```

### Required Config Fields

Document in your app README:
- **Secrets:** `apiToken` - API token from external system

---

## API Key Authentication

### Header Template

`authentication/headers/X-API-Key.gtpl`:

```go
{{.integration.secrets.apiKey}}
```

### Custom Header Name

Name the file after the header:
- `authentication/headers/Api-Key.gtpl` for `Api-Key: xxx`
- `authentication/headers/X-Auth-Token.gtpl` for `X-Auth-Token: xxx`

---

## Basic Authentication

### Header Template

`authentication/headers/Authorization.gtpl`:

```go
Basic {{printf "%s:%s" .integration.configuration.username .integration.secrets.password | b64enc}}
```

### Required Config Fields
- **Configuration:** `username`
- **Secrets:** `password`

---

## OAuth 2.0 Client Credentials

For server-to-server authentication without user interaction.

### Directory Structure

```
authentication/
├── headers/
│   └── Authorization.gtpl
└── oauth/
    ├── config.json
    └── access_token/
        ├── config.json
        ├── request_url.gtpl
        └── request_body.gtpl
```

### oauth/config.json

```json
{
  "grantType": "client_credentials"
}
```

### oauth/access_token/config.json

```json
{
  "httpMethod": "POST",
  "contentType": "application/x-www-form-urlencoded"
}
```

### oauth/access_token/request_url.gtpl

```go
{{.integration.configuration.tokenUrl}}
```

### oauth/access_token/request_body.gtpl

```go
grant_type=client_credentials&client_id={{urlquery .integration.configuration.clientId}}&client_secret={{urlquery .integration.secrets.clientSecret}}
```

### headers/Authorization.gtpl

```go
Bearer {{.integration.secrets.access_token}}
```

### Required Config Fields
- **Configuration:** `tokenUrl`, `clientId`
- **Secrets:** `clientSecret`

---

## OAuth 2.0 Authorization Code

For user-authorized access (requires user login flow).

### Directory Structure

```
authentication/
├── headers/
│   └── Authorization.gtpl
└── oauth/
    ├── config.json
    ├── auth_code/
    │   ├── config.json
    │   └── request_url.gtpl
    ├── access_token/
    │   ├── config.json
    │   ├── request_url.gtpl
    │   └── request_body.gtpl
    └── refresh_token/
        ├── config.json
        ├── request_url.gtpl
        └── request_body.gtpl
```

### oauth/config.json

```json
{
  "grantType": "authorization_code",
  "authDomain": "company.com"
}
```

### oauth/auth_code/config.json

```json
{
  "httpMethod": "GET"
}
```

### oauth/auth_code/request_url.gtpl

```go
{{.integration.configuration.authUrl}}?client_id={{urlquery .integration.configuration.clientId}}&redirect_uri={{urlquery .oauth.redirect_uri}}&response_type=code&scope={{urlquery .integration.configuration.scope}}&state={{.correlationId}}
```

### oauth/access_token/config.json

```json
{
  "httpMethod": "POST",
  "contentType": "application/x-www-form-urlencoded"
}
```

### oauth/access_token/request_url.gtpl

```go
{{.integration.configuration.tokenUrl}}
```

### oauth/access_token/request_body.gtpl

```go
grant_type=authorization_code&code={{urlquery .oauth.code}}&redirect_uri={{urlquery .oauth.redirect_uri}}&client_id={{urlquery .integration.configuration.clientId}}&client_secret={{urlquery .integration.secrets.clientSecret}}
```

### oauth/refresh_token/config.json

```json
{
  "httpMethod": "POST",
  "contentType": "application/x-www-form-urlencoded"
}
```

### oauth/refresh_token/request_url.gtpl

```go
{{.integration.configuration.tokenUrl}}
```

### oauth/refresh_token/request_body.gtpl

```go
grant_type=refresh_token&refresh_token={{urlquery .integration.secrets.refresh_token}}&client_id={{urlquery .integration.configuration.clientId}}&client_secret={{urlquery .integration.secrets.clientSecret}}
```

### headers/Authorization.gtpl

```go
Bearer {{.integration.secrets.access_token}}
```

---

## OAuth 2.0 Password Grant

For username/password authentication (legacy, avoid if possible).

### oauth/config.json

```json
{
  "grantType": "password"
}
```

### oauth/access_token/request_body.gtpl

```go
grant_type=password&username={{urlquery .integration.configuration.username}}&password={{urlquery .integration.secrets.password}}&client_id={{urlquery .integration.configuration.clientId}}&client_secret={{urlquery .integration.secrets.clientSecret}}
```

---

## Request Signing

For APIs that require request signatures (e.g., AWS-style signing).

### authentication/request_signing/X-Signature.gtpl

```go
{{- $timestamp := now | date "2006-01-02T15:04:05Z07:00" -}}
{{- $payload := printf "%s\n%s\n%s" .request.method .request.url $timestamp -}}
{{- $signature := hmac "sha256" .integration.secrets.signingKey $payload | b64enc -}}
{{$signature}}
```

---

## Multiple Headers

You can have multiple header files:

```
authentication/headers/
├── Authorization.gtpl      # Bearer token
├── X-API-Version.gtpl      # API version
└── X-Tenant-ID.gtpl        # Tenant identifier
```

### X-API-Version.gtpl

```go
2024-01-01
```

### X-Tenant-ID.gtpl

```go
{{.integration.configuration.tenantId}}
```

---

## Testing Authentication

### _test_/default/integration.json

```json
{
  "configuration": {
    "apiBaseUrl": "https://api.example.com/v1",
    "clientId": "test-client-id",
    "tenantId": "test-tenant"
  },
  "secrets": {
    "apiToken": "test-api-token-123",
    "clientSecret": "test-client-secret"
  }
}
```

### Test Header Output

Create `authentication/headers/_test_/default/`:

```
_test_/default/
├── integration.json
└── expected_Authorization.txt
```

`expected_Authorization.txt`:
```
Bearer test-api-token-123
```

---

## Quick Reference

| Auth Type | Files Needed |
|-----------|--------------|
| Bearer Token | `headers/Authorization.gtpl` |
| API Key | `headers/X-API-Key.gtpl` |
| Basic Auth | `headers/Authorization.gtpl` |
| OAuth Client Creds | `oauth/config.json`, `oauth/access_token/*`, `headers/Authorization.gtpl` |
| OAuth Auth Code | Above + `oauth/auth_code/*`, `oauth/refresh_token/*` |

---

## CLI Commands

```bash
# Add bearer token header
appcfg add auth-header Authorization --root .

# Add OAuth client credentials
appcfg add oauth client_credentials --root .

# Add OAuth authorization code
appcfg add oauth authorization_code --root .
```
