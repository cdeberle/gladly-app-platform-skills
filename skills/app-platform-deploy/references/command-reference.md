# App Platform CLI Command Reference

Quick reference for `appcfg` commands used in deployment workflows.

## Development Commands

### Test Data Pulls
```bash
# Test specific data pull
appcfg test data-pull <name> --root .

# Test all data pulls
appcfg test data-pull --root .

# Test with specific dataset
appcfg test data-pull <name> --data-set <dataset-name> --root .
```

### Validate
```bash
appcfg validate --root .
```

### Build
```bash
appcfg build --root .
# Output: {author}-{appName}-{version}.zip
```

## App Management Commands

### Install App
```bash
appcfg apps install -f <zip-file> --root .

# With explicit credentials
appcfg apps install -f <zip-file> \
  -g <gladly-host> \
  -u <user-email> \
  -t <api-token> \
  --root .
```

### List Installed Apps
```bash
appcfg apps list --root .
```

### App Info
```bash
appcfg apps info <app-id> --root .
# Example: appcfg apps info "CEberle/demoCRM/v1.0.5" --root .
```

### Uninstall App
```bash
appcfg apps uninstall <app-id> --root .
```

### Upgrade App Configurations
```bash
appcfg apps upgrade <old-app-id> -n v<new-version> --root .

# Example: Upgrade all configs from v1.0.0 to v1.2.0
appcfg apps upgrade "gladly.com/shopify/v1.0.0" -n v1.2.0 --root .
```

## Configuration Commands

### List Configurations
```bash
appcfg apps config list --root .
```

### Create Configuration
```bash
appcfg apps config create <app-id> \
  --name "Config Name" \
  --config '{"apiBaseUrl": "https://...", "brandSlug": "..."}' \
  --secrets '{"apiKey": "..."}' \
  --activate \
  --root .
```

### Update Configuration
```bash
# Update name
appcfg apps config update "<config-name>" --name "New Name" --root .

# Update configuration data
appcfg apps config update "<config-name>" --config '{"key": "value"}' --root .

# Update secrets
appcfg apps config update "<config-name>" --secrets '{"key": "value"}' --root .

# Activate
appcfg apps config update "<config-name>" --activate --root .

# Deactivate
appcfg apps config update "<config-name>" --deactivate --root .
```

### Delete Configuration
```bash
# Must be deactivated first
appcfg apps config delete "<config-name>" --root .
```

### Check Outdated Configurations
```bash
appcfg apps config outdated --root .
```

## Debugging Commands

### View Logs
```bash
# List recent logs
appcfg apps logs list --root .

# View log details
appcfg apps logs detail <log-id> --root .
```

### Test Against Real API
```bash
# Interactive GraphQL console
appcfg run data-graphql --root .

# Example query:
# query { customer(email: "test@example.com") { id name orders { id } } }
```

## Environment Variables

All commands can use environment variables instead of flags:

| Variable | Flag Equivalent |
|----------|-----------------|
| `GLADLY_APP_CFG_ROOT` | `--root` |
| `GLADLY_APP_CFG_HOST` | `-g, --gladly-host` |
| `GLADLY_APP_CFG_USER` | `-u, --user` |
| `GLADLY_APP_CFG_TOKEN` | `-t, --token` |

## Common Patterns

### Full Deploy Workflow
```bash
# 1. Validate
appcfg validate --root .

# 2. Test
appcfg test data-pull --root .

# 3. Build
appcfg build --root .

# 4. Install
appcfg apps install -f "./{author}-{appName}-{version}.zip" --root .

# 5. Create config
appcfg apps config create "{author}/{appName}/v{version}" \
  --name "Production Config" \
  --config '{"apiBaseUrl": "...", "brandSlug": "..."}' \
  --activate \
  --root .
```

### Version Upgrade Workflow
```bash
# 1. Build new version
appcfg build --root .

# 2. Install new version
appcfg apps install -f "./{author}-{appName}-{newVersion}.zip" --root .

# 3. Upgrade all configs
appcfg apps upgrade "{author}/{appName}/v{oldVersion}" -n v{newVersion} --root .

# 4. Cleanup old versions (keep N and N-1)
appcfg apps uninstall "{author}/{appName}/v{veryOldVersion}" --root .
```

### Rollback Workflow
```bash
# Reactivate old config
appcfg apps config update "Old Config" --activate --root .

# Deactivate new config
appcfg apps config update "New Config" --deactivate --root .
```
