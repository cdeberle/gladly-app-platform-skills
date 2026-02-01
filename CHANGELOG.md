# Changelog

All notable changes to the Gladly App Platform Skills will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-02-01

### Added

#### app-platform-builder skill
- 7-phase checkpoint workflow: DISCOVER → SCAFFOLD → MODEL → CONNECT → ACTIONS → PRESENT → VERIFY
- Explicit reference loading instructions at each phase
- Phase completion checklists with verification steps
- Inline error recovery guidance ("If X fails...")
- Seamless handoff protocol to app-platform-deploy skill
- Reference documentation:
  - `code-templates.md` - Ready-to-use code templates for all components
  - `go-template-reference.md` - Go template syntax, Sprig functions, and common pitfalls
  - `graphql-schema-patterns.md` - Type mapping, @dataType, @parentId/@childIds directives
  - `data-pull-templates.md` - Request URL, response transformation, and test fixtures
  - `action-templates.md` - Mutation handlers with error code handling
  - `ui-card-patterns.md` - Flexible Card XML components and patterns
  - `authentication-patterns.md` - Bearer token, API key, OAuth 2.0 patterns

#### app-platform-deploy skill
- 4 operating modes: DEVELOP, DEPLOY, UPDATE, UPGRADE
- Automatic mode detection based on project state
- Prerequisites verification from builder skill
- Mode completion checklists
- Rollback procedure documentation
- Reference documentation:
  - `command-reference.md` - Complete appcfg CLI command reference
  - `schema-versioning.md` - Breaking vs non-breaking changes, upgrade paths
  - `troubleshooting.md` - Debugging workflows, common error messages, decision trees

#### app-platform-audit skill
- 8-step audit workflow: VALIDATE → STRUCTURE → AUTH → SCHEMAS → DATA PULLS → ACTIONS → UI → REPORT
- Comprehensive best practices verification against all reference documentation
- Severity-rated findings system (Critical, High, Medium)
- Authentication audit: OAuth flows, request signing, secrets separation
- Safety pattern verification for data pulls and actions
- Advanced template pattern checks: `kindIs`, `hasKey`, root context in range, DateTime conversion
- Advanced UI pattern checks: lowercase operators, special variables, Image/Column/Formula attributes
- Test coverage assessment
- Structured audit report generation with prioritized remediation plan
- Self-contained output with actionable fix recommendations
- Reference documentation:
  - `audit-checklist.md` - Complete checklist covering all patterns from builder skill

### Design Decisions

The following design principles were applied to maximize AI adherence rates:

1. **Concise SKILL.md files** - Main skill files kept under 400 lines to reduce context window pressure
2. **Explicit reference loading** - Each phase specifies exactly which reference documents to read
3. **Strong checkpoints** - Verification checklists and explicit "Ask user" prompts prevent runaway execution
4. **Inline error recovery** - Every phase includes guidance for common failure modes
5. **Consistent command syntax** - All appcfg commands include `--root .` for predictability
6. **Self-contained documentation** - All references are within the skill package; no external local dependencies

### Technical Notes

- All external URLs verified as publicly accessible (GitHub, pkg.go.dev, Sprig docs)
- Command syntax standardized across all files (single quotes for JSON, `-f` flag for files)
- Test fixture patterns documented for edge cases (empty arrays, null fields, missing data)

[1.0.0]: https://github.com/gladly/gladly-app-platform-skills/releases/tag/v1.0.0
