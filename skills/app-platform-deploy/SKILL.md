---
name: app-platform-deploy
description: Deploy and manage Gladly App Platform apps. Use this skill when testing, validating, building, installing, configuring, or upgrading App Platform applications. Handles the full deployment lifecycle including version management and configuration updates.
---

# App Platform Deploy

Deploy and manage Gladly App Platform applications across 4 operating modes.

## Prerequisites

Ensure `appcfg` CLI is installed: https://github.com/gladly/app-platform-appcfg-cli/releases

### Verify Builder Completed (if coming from /app-platform-builder)

```bash
ls manifest.json          # Must exist
appcfg validate --root .  # Must return "valid"
```

**If validation fails**: Return to `/app-platform-builder` Phase 7 to resolve issues.

---

## Mode Detection

Determine current state before proceeding:

```bash
ls manifest.json        # Check: Is this an App Platform project?
ls *.zip               # Check: Is there a built ZIP?
cat .env               # Check: Is .env configured?
appcfg apps list --root .  # Check: Is app installed?
```

### Mode Selection

| State | Mode | What to Do |
|-------|------|------------|
| No ZIP file | **DEVELOP** | Test → Validate → Build |
| ZIP exists, not installed | **DEPLOY** | Install → Create config |
| App installed, need config changes | **UPDATE** | Modify existing configs |
| New version ready, old installed | **UPGRADE** | Install new → Migrate configs |

**Ask user**: "Based on the current state, I'll proceed with [MODE] mode. Continue?"

---

## DEVELOP Mode

**When**: You have code but no built ZIP file.

**Required Reading**: [command-reference.md](references/command-reference.md)

### Step 1: Test Data Pulls

```bash
# Test all data pulls
appcfg test data-pull --root .

# Or test specific pull
appcfg test data-pull <name> --root .
```

**If tests fail**: Check `_test_/<name>/_output_/` for errors. See [troubleshooting.md](references/troubleshooting.md).

### Step 2: Validate

```bash
appcfg validate --root .
```

Must return `valid` before building.

### Step 3: Build

```bash
appcfg build --root .
```

**Output**: `{author}-{appName}-{version}.zip`

### DEVELOP MODE COMPLETE

Verify:
- [ ] All data pull tests pass
- [ ] `appcfg validate --root .` returns "valid"
- [ ] ZIP file created: `ls *.zip`

**Ask user**: "Build complete. Ready to deploy (DEPLOY mode)?"

---

## DEPLOY Mode

**When**: ZIP exists but app is not installed in target Gladly org.

**Required Reading**: [command-reference.md](references/command-reference.md)

### Prerequisites: Configure .env

The `.env` file must contain:

```bash
GLADLY_APP_CFG_ROOT="/path/to/app"
GLADLY_APP_CFG_HOST=[org-name].us-uat.gladly.qa    # UAT
# or                [org-name].us-1.gladly.com     # Production
GLADLY_APP_CFG_USER=email@domain.com
GLADLY_APP_CFG_TOKEN=<api-token>
```

### Step 1: Install the App

```bash
appcfg apps install -f "./{author}-{appName}-{version}.zip" --root .
```

### Step 2: Verify Installation

```bash
appcfg apps list --root .
```

### Step 3: Find Required Config Fields

```bash
grep -r "integration.configuration" data/ authentication/
grep -r "integration.secrets" data/ authentication/
```

### Step 4: Create Configuration

```bash
appcfg apps config create "{author}/{appName}/v{version}" \
  --name "Config Name" \
  --config '{"apiBaseUrl": "https://...", "brandSlug": "..."}' \
  --secrets '{}' \
  --activate \
  --root .
```

### DEPLOY MODE COMPLETE

Verify:
- [ ] App shows in `appcfg apps list --root .`
- [ ] Config shows in `appcfg apps config list --root .`
- [ ] Config is active

**If credentials error**: Verify .env has correct HOST, USER, TOKEN. See [troubleshooting.md](references/troubleshooting.md#credentials-error).

---

## UPDATE Mode

**When**: App is installed and you need to modify existing configurations.

**Required Reading**: [command-reference.md](references/command-reference.md)

### List Current Configurations

```bash
appcfg apps config list --root .
```

### Update Operations

```bash
# Update name
appcfg apps config update "<config-name>" --name "New Name" --root .

# Update configuration data
appcfg apps config update "<config-name>" --config '{"key": "newValue"}' --root .

# Update secrets
appcfg apps config update "<config-name>" --secrets '{"apiKey": "newSecret"}' --root .

# Activate/Deactivate
appcfg apps config update "<config-name>" --activate --root .
appcfg apps config update "<config-name>" --deactivate --root .
```

### Delete Configuration (Use Caution)

**WARNING**: Deletion is irreversible.

1. Deactivate first:
   ```bash
   appcfg apps config update "<config-name>" --deactivate --root .
   ```
2. Recommend: Monitor for issues before deleting
3. Then delete:
   ```bash
   appcfg apps config delete "<config-name>" --root .
   ```

### UPDATE MODE COMPLETE

Verify:
- [ ] Config changes applied
- [ ] Run `appcfg apps config list --root .` to confirm

---

## UPGRADE Mode

**When**: New app version is ready, older version is installed with existing configs.

**Required Reading**: [schema-versioning.md](references/schema-versioning.md)

### Determine Change Type

| Change Type | Breaking? | Path |
|-------------|-----------|------|
| Add new field | No | `appcfg apps upgrade` |
| Add enum value | No | `appcfg apps upgrade` |
| Remove field | **Yes** | New config required |
| String → enum | **Yes** | New config required |
| String → DateTime | **Yes** | New config required |

### Non-Breaking Upgrade (Minor Version)

```bash
# 1. Install new version
appcfg apps install -f "./{author}-{appName}-{newVersion}.zip" --root .

# 2. Upgrade all configs
appcfg apps upgrade "{author}/{appName}/v{oldVersion}" -n v{newVersion} --root .

# 3. Verify
appcfg apps config list --root .

# 4. Cleanup old version (keep N and N-1 for rollback)
appcfg apps uninstall "{author}/{appName}/v{veryOldVersion}" --root .
```

### Breaking Upgrade (Major Version)

```bash
# 1. Install new version
appcfg apps install -f "./{author}-{appName}-{newVersion}.zip" --root .

# 2. Create NEW configuration (can't upgrade across major versions)
appcfg apps config create "{author}/{appName}/v{newVersion}" \
  --name "Config Name" \
  --config '{"apiBaseUrl": "...", "brandSlug": "..."}' \
  --activate \
  --root .

# 3. Deactivate old config (keep for rollback)
appcfg apps config update "Old Config" --deactivate --root .
```

### UPGRADE MODE COMPLETE

Verify:
- [ ] New version shows in `appcfg apps list --root .`
- [ ] Configs point to new version
- [ ] Old version kept for rollback (N-1)

**If upgrade fails**: See [troubleshooting.md](references/troubleshooting.md#upgrade-fails).

---

## Rollback Procedure

If issues arise after deployment:

```bash
# Reactivate previous config
appcfg apps config update "Old Config" --activate --root .

# Deactivate problematic config
appcfg apps config update "New Config" --deactivate --root .
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Test data pull | `appcfg test data-pull <name> --root .` |
| Validate | `appcfg validate --root .` |
| Build | `appcfg build --root .` |
| Install | `appcfg apps install -f <zip> --root .` |
| List apps | `appcfg apps list --root .` |
| List configs | `appcfg apps config list --root .` |
| Create config | `appcfg apps config create <app-id> --name <n> --config '<json>' --activate --root .` |
| Update config | `appcfg apps config update <name> [options] --root .` |
| Upgrade configs | `appcfg apps upgrade <old-app-id> -n v<new> --root .` |
| View logs | `appcfg apps logs list --root .` |

---

## Reference Documentation

| When | Read |
|------|------|
| CLI commands | [command-reference.md](references/command-reference.md) |
| Version management | [schema-versioning.md](references/schema-versioning.md) |
| Debugging | [troubleshooting.md](references/troubleshooting.md) |
