---
name: app-platform-deploy
description: Deploy and manage Gladly App Platform apps. Use this skill when testing, validating, building, installing, configuring, or upgrading App Platform applications. Handles the full deployment lifecycle including version management and configuration updates.
---

# App Platform Deploy

This skill guides you through deploying and managing Gladly App Platform applications.

## Prerequisites

Ensure the `appcfg` CLI is installed. If not:
```bash
cd /path/to/.app-platform-toolkit/appcfg-cli && ./install.sh
```

## Mode Detection

Before proceeding, determine the current state:

| Check | Command | Result |
|-------|---------|--------|
| Is this an App Platform project? | `ls manifest.json` | Must exist |
| Is there a built ZIP? | `ls *.zip` | Check for `{author}-{appName}-{version}.zip` |
| Is .env configured? | `cat .env` | Must have HOST, USER, TOKEN |
| Is app installed? | `appcfg apps list --root .` | Shows installed versions |

### Mode Selection

| State | Mode | Actions |
|-------|------|---------|
| No ZIP file | **DEVELOP** | Test → Validate → Build |
| ZIP exists, not installed | **DEPLOY** | Install → Create config |
| App installed, config needs changes | **UPDATE** | Update or delete configs |
| New version, old version installed | **UPGRADE** | Install new → Upgrade configs → Cleanup |

---

## DEVELOP Mode

**When:** You have code but no built ZIP file.

### Step 1: Test Data Pulls

List available data pulls:
```bash
ls data/pull/
```

Test each data pull:
```bash
appcfg test data-pull <name> --root .
```

Test all data pulls at once:
```bash
appcfg test data-pull --root .
```

**On failure:** Check `_test_/<name>/_output_/` for detailed errors.

### Step 2: Validate

```bash
appcfg validate --root .
```

Must return `valid` before building.

### Step 3: Build

```bash
appcfg build --root .
```

**Output:** `{author}-{appName}-{version}.zip`

---

## DEPLOY Mode

**When:** ZIP exists but app is not installed in target Gladly org.

### Prerequisites: Configure .env

The `.env` file must contain:
```bash
GLADLY_APP_CFG_ROOT="/path/to/app"
GLADLY_APP_CFG_HOST=[org-name].us-uat.gladly.qa    # UAT
# or                [org-name].us-1.gladly.com     # Production
GLADLY_APP_CFG_USER=email@domain.com               # Email tied to API token
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

### Step 3: Create Configuration

```bash
appcfg apps config create "{author}/{appName}/v{version}" \
  --name "Config Name" \
  --config '{"apiBaseUrl": "https://...", "brandSlug": "..."}' \
  --secrets '{}' \
  --activate \
  --root .
```

**Note:** Find required config fields by searching templates:
```bash
grep -r "integration.configuration" data/ authentication/
grep -r "integration.secrets" data/ authentication/
```

---

## UPDATE Mode

**When:** App is installed and you need to modify existing configurations.

### List Current Configurations

```bash
appcfg apps config list --root .
```

### Update Configuration

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

### Delete Configuration (Careful!)

**WARNING:** Deletion is irreversible.

1. First deactivate:
   ```bash
   appcfg apps config update "<config-name>" --deactivate --root .
   ```

2. Monitor for issues (recommended: several days)

3. Then delete:
   ```bash
   appcfg apps config delete "<config-name>" --root .
   ```

---

## UPGRADE Mode

**When:** New app version is ready, older version is installed with existing configs.

### Step 1: Install New Version

```bash
appcfg apps install -f "./{author}-{appName}-{newVersion}.zip" --root .
```

Both versions now coexist.

### Step 2: Upgrade Configurations

Move all configs from old version to new:
```bash
appcfg apps upgrade "{author}/{appName}/v{oldVersion}" -n v{newVersion} --root .
```

**Requirements:**
- New version must be installed first
- Only works within same major version (1.x → 1.y, NOT 1.x → 2.x)
- For breaking changes (major version), create new configs instead

### Step 3: Verify

```bash
appcfg apps config list --root .
```

### Step 4: Cleanup Old Version (Optional)

After confirming everything works:
```bash
appcfg apps uninstall "{author}/{appName}/v{oldVersion}" --root .
```

**Best Practice:** Keep N and N-1 versions for rollback capability.

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
| Create config | `appcfg apps config create <app-id> --name <name> --config '<json>' --activate --root .` |
| Update config | `appcfg apps config update <name> [--config/--secrets/--name/--activate/--deactivate] --root .` |
| Upgrade configs | `appcfg apps upgrade <old-app-id> -n v<new> --root .` |
| Delete config | `appcfg apps config delete <name> --root .` |
| Uninstall app | `appcfg apps uninstall <app-id> --root .` |
| View logs | `appcfg apps logs list --root .` |
| Check outdated | `appcfg apps config outdated --root .` |

---

## Breaking vs Non-Breaking Changes

| Change Type | Breaking? | Version Bump | Path |
|-------------|-----------|--------------|------|
| Add new field | No | Minor (x.Y.z) | `appcfg apps upgrade` |
| Add enum value | No | Minor | `appcfg apps upgrade` |
| Remove field | **Yes** | Major (X.0.0) | New config required |
| String → enum | **Yes** | Major | New config required |
| String → DateTime | **Yes** | Major | New config required |
| Change nullability | **Yes** | Major | New config required |

---

## Common Issues

### "Something's wrong with this card"
1. Run `appcfg test data-pull <name>` for each data pull
2. Test with `appcfg run data-graphql` against real API
3. Check dependent data pulls succeed first
4. Verify JSON output is valid (no trailing commas)

### Credentials Error
- Verify .env has correct HOST, USER, TOKEN
- Check API token has appropriate permissions
- Ensure HOST matches environment (UAT vs Production)

### Upgrade Fails
- Cannot upgrade across major versions
- New version must be installed before upgrading
- For breaking changes, create new configurations

---

## References

- [command-reference.md](references/command-reference.md) - Complete command documentation
- [troubleshooting.md](references/troubleshooting.md) - Debugging workflow and error resolution
- [schema-versioning.md](references/schema-versioning.md) - Version management and deployment patterns
