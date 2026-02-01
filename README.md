# Gladly App Platform Skills for Claude Code

Claude Code skills for building and deploying Gladly App Platform applications. These skills guide you from raw API payload analysis through fully deployed integrations on the Gladly platform.

## Skills Included

### `/app-platform-builder`

Build App Platform apps from scratch with a guided 7-phase checkpoint workflow:

| Phase | Name | Description |
|-------|------|-------------|
| 1 | **DISCOVER** | Analyze API payloads, identify data types and relationships |
| 2 | **SCAFFOLD** | Initialize project structure and authentication |
| 3 | **MODEL** | Generate GraphQL schemas (data + actions) |
| 4 | **CONNECT** | Create data pulls with safety patterns |
| 5 | **ACTIONS** | Create mutation handlers with error handling |
| 6 | **PRESENT** | Build Flexible Card UI templates |
| 7 | **VERIFY** | Test and validate before deployment |

**Key Features:**
- Checkpoint-based workflow with user review between phases
- Explicit reference loading instructions per phase
- Automatic type inference from JSON payloads
- Safety patterns for dependent data pulls
- Error recovery guidance at each phase
- Seamless handoff to deploy skill

### `/app-platform-deploy`

Deploy and manage App Platform apps with 4 operating modes:

| Mode | When to Use |
|------|-------------|
| **DEVELOP** | No ZIP file - test, validate, build |
| **DEPLOY** | ZIP exists - install and create config |
| **UPDATE** | App installed - modify existing configs |
| **UPGRADE** | New version - migrate configs to new version |

**Key Features:**
- Automatic mode detection based on project state
- Prerequisites verification from builder skill
- Version management (keep N and N-1 for rollback)
- Breaking vs non-breaking change guidance
- Troubleshooting decision trees

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/gladly/gladly-app-platform-skills.git
   ```

2. Symlink to your Claude Code skills directory:
   ```bash
   ln -s /path/to/gladly-app-platform-skills/skills/app-platform-builder ~/.claude/skills/app-platform-builder
   ln -s /path/to/gladly-app-platform-skills/skills/app-platform-deploy ~/.claude/skills/app-platform-deploy
   ```

## Prerequisites

- [appcfg CLI](https://github.com/gladly/app-platform-appcfg-cli/releases) - download for your OS from releases page
- [Claude Code](https://claude.ai/code) CLI

## Usage

```bash
# Start Claude Code in your project directory
claude

# Build a new app from scratch
/app-platform-builder

# Deploy an existing app
/app-platform-deploy
```

### Example: Building a New Integration

1. Run `/app-platform-builder`
2. Provide your API details (base URL, auth method)
3. Paste a sample JSON response from your API
4. Review generated GraphQL schemas at checkpoint
5. Approve data pulls and actions at each checkpoint
6. Customize UI templates
7. Validate and handoff to `/app-platform-deploy`

## Skill Structure

```
skills/
├── app-platform-builder/
│   ├── SKILL.md                          # Main skill (7-phase workflow)
│   └── references/
│       ├── code-templates.md             # Ready-to-use code templates
│       ├── go-template-reference.md      # Template fundamentals & pitfalls
│       ├── graphql-schema-patterns.md    # Type mapping, directives
│       ├── data-pull-templates.md        # Request/response patterns
│       ├── action-templates.md           # Mutation patterns
│       ├── ui-card-patterns.md           # Flexible Card XML
│       └── authentication-patterns.md    # Auth setup
│
└── app-platform-deploy/
    ├── SKILL.md                          # Main skill (4 modes)
    └── references/
        ├── command-reference.md          # appcfg CLI commands
        ├── troubleshooting.md            # Debugging guide
        └── schema-versioning.md          # Version management
```

## Design Principles

These skills are designed for high AI adherence rates:

1. **Concise main skills** - SKILL.md files are kept under 400 lines to reduce context pressure
2. **Explicit reference loading** - Each phase specifies which reference docs to read
3. **Strong checkpoints** - Verification checklists and explicit user prompts prevent runaway execution
4. **Inline error recovery** - "If X fails" guidance at each phase
5. **Consistent command syntax** - All appcfg commands include `--root .` for predictability
6. **Self-contained references** - All documentation is within the skill package (no external dependencies)

## Reference Documentation

| Document | Purpose |
|----------|---------|
| [code-templates.md](skills/app-platform-builder/references/code-templates.md) | Copy-paste code templates |
| [go-template-reference.md](skills/app-platform-builder/references/go-template-reference.md) | Go template syntax and gotchas |
| [graphql-schema-patterns.md](skills/app-platform-builder/references/graphql-schema-patterns.md) | Schema generation rules |
| [data-pull-templates.md](skills/app-platform-builder/references/data-pull-templates.md) | Data pull configuration |
| [action-templates.md](skills/app-platform-builder/references/action-templates.md) | Action/mutation patterns |
| [ui-card-patterns.md](skills/app-platform-builder/references/ui-card-patterns.md) | Flexible Card XML |
| [authentication-patterns.md](skills/app-platform-builder/references/authentication-patterns.md) | Auth setup patterns |
| [command-reference.md](skills/app-platform-deploy/references/command-reference.md) | CLI command reference |
| [troubleshooting.md](skills/app-platform-deploy/references/troubleshooting.md) | Debugging workflows |
| [schema-versioning.md](skills/app-platform-deploy/references/schema-versioning.md) | Version management |

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for release history.

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

When contributing, please follow the design principles above to maintain high AI adherence rates.

## License

MIT License - see [LICENSE](LICENSE) for details.
