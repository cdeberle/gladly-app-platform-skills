# Schema Versioning & Deployment Guide

## Breaking vs Non-Breaking Changes

The App Platform enforces strict schema compatibility. Understanding which changes require major version bumps is critical for deployment planning.

| Change Type | Breaking? | Version Bump | Deployment Path |
|-------------|-----------|--------------|-----------------|
| Add new field | No | Minor (x.Y.z) | `appcfg apps upgrade` |
| Add new enum value | No | Minor (x.Y.z) | `appcfg apps upgrade` |
| Remove field | **Yes** | Major (X.0.0) | New config required |
| `String` → `enum` | **Yes** | Major (X.0.0) | New config required |
| `String` → `DateTime` | **Yes** | Major (X.0.0) | New config required |
| `String` → `Int` | **Yes** | Major (X.0.0) | New config required |
| Change field nullability | **Yes** | Major (X.0.0) | New config required |

---

## Standard Deployment Pattern (Keep Last 2 Versions)

Always maintain the current version plus one previous version for rollback. Clean up anything older.

### Minor Version Upgrade (Non-Breaking)

```bash
# 1. Build new version
appcfg build --root .  # Creates app-v1.1.0.zip

# 2. Install new version alongside current
appcfg apps install -f "app-v1.1.0.zip" --root .

# 3. Upgrade existing configuration to new version
appcfg apps upgrade "author/appName/v1.0.5" -n v1.1.0 --root .

# 4. Clean up versions older than N-1 (keep v1.1.0 and v1.0.5)
appcfg apps uninstall "author/appName/v1.0.4" --root .
appcfg apps uninstall "author/appName/v1.0.3" --root .
```

### Major Version Upgrade (Breaking Changes)

```bash
# 1. Build new major version
appcfg build --root .  # Creates app-v3.0.0.zip

# 2. Install new version alongside current
appcfg apps install -f "app-v3.0.0.zip" --root .

# 3. Create new configuration (can't use upgrade path for breaking changes)
appcfg apps config create "author/appName/v3.0.0" \
  --name "Config Name" \
  --activate \
  --config '{"apiBaseUrl":"...","brandSlug":"..."}' \
  --secrets '{}' \
  --root .

# 4. Deactivate old configuration (keep for rollback)
appcfg apps config update "Old Config Name" --deactivate --root .

# 5. Clean up versions older than N-1
# Keep: v3.0.0 (current) and v2.0.2 (rollback)
# Delete: v2.0.1, v2.0.0, v1.x.x, etc.
```

---

## Rollback Procedure

```bash
# Reactivate previous version's config:
appcfg apps config update "Old Config Name" --activate --root .
appcfg apps config update "Config Name" --deactivate --root .
```

---

## Version Retention Policy

- **Always keep:** Current version (N) + Previous version (N-1)
- **Delete:** All versions older than N-1
- This ensures rollback capability while preventing version bloat

---

## Configuration Lifecycle

### States

| State | Description | Can Delete? |
|-------|-------------|-------------|
| Active | Currently in use | No |
| Inactive | Deactivated, available for rollback | Yes |

### Commands

```bash
# Create new configuration
appcfg apps config create "author/app/v1.0.0" \
  --name "Config Name" \
  --activate \
  --config '{"key":"value"}' \
  --secrets '{"secret":"value"}' \
  --root .

# Deactivate (prepare for deletion or rollback reserve)
appcfg apps config update "Config Name" --deactivate --root .

# Reactivate (rollback)
appcfg apps config update "Config Name" --activate --root .

# Delete (only inactive configs)
appcfg apps config delete "Config Name" --root .
```

---

## Finding Required Configuration Fields

Search templates for required integration fields:

```bash
grep -r "integration.configuration" data/ authentication/
grep -r "integration.secrets" data/ authentication/
```

---

## Pre-Deployment Checklist

1. [ ] Run `appcfg validate --root .` - schema and templates valid
2. [ ] Run `appcfg test data-pull --root .` for all data pulls
3. [ ] Determine if changes are breaking (see table above)
4. [ ] Bump version appropriately in `manifest.json`
5. [ ] Build with `appcfg build --root .`
6. [ ] Install new version alongside existing
7. [ ] Create/upgrade configuration
8. [ ] Verify functionality
9. [ ] Clean up old versions (keep N and N-1)
