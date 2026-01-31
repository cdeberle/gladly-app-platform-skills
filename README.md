# Gladly App Platform Skills for Claude Code

Claude Code skills for building and deploying Gladly App Platform applications.

## Skills Included

### `/app-platform-builder`

Build App Platform apps from scratch with a guided 7-phase workflow:

1. **DISCOVER** - Analyze API payloads, identify data types and relationships
2. **SCAFFOLD** - Initialize project structure and authentication
3. **MODEL** - Generate GraphQL schemas (data + actions)
4. **CONNECT** - Create data pulls with safety patterns
5. **ACTIONS** - Create mutation handlers with error handling
6. **PRESENT** - Build Flexible Card UI templates
7. **VERIFY** - Test and validate before deployment

Features:
- Checkpoint-based workflow with user review between phases
- Automatic type inference from JSON payloads
- Safety patterns for dependent data pulls
- Error handling templates for actions
- Flexible Card XML patterns

### `/app-platform-deploy`

Deploy and manage App Platform apps with 4 operating modes:

- **DEVELOP** - Test data pulls, validate schemas, build ZIP
- **DEPLOY** - Install app and create configurations
- **UPDATE** - Modify existing configurations
- **UPGRADE** - Deploy new versions and migrate configs

Features:
- Mode detection based on current state
- Version management (keep N and N-1)
- Breaking vs non-breaking change guidance
- Troubleshooting decision trees

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/gladly-app-platform-skills.git
   ```

2. Symlink to your Claude skills directory:
   ```bash
   ln -s /path/to/gladly-app-platform-skills/skills/app-platform-builder ~/.claude/skills/app-platform-builder
   ln -s /path/to/gladly-app-platform-skills/skills/app-platform-deploy ~/.claude/skills/app-platform-deploy
   ```

## Prerequisites

- [appcfg CLI](https://github.com/gladly/app-platform-appcfg-cli) installed
- [Claude Code](https://claude.ai/claude-code) CLI
- Access to Gladly App Platform toolkit (`.app-platform-toolkit/`)

## Usage

```bash
# Start Claude Code
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
4. Review generated GraphQL schemas
5. Approve data pulls and actions
6. Customize UI templates
7. Validate and deploy with `/app-platform-deploy`

## Skill Structure

```
skills/
├── app-platform-builder/
│   ├── SKILL.md                          # Main skill (7-phase workflow)
│   └── references/
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

## Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

MIT License - see [LICENSE](LICENSE) for details.
