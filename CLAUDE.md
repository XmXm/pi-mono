# CLAUDE.md

This file provides guidance for AI assistants working with the pi-mono codebase.

## Project Overview

pi-mono is a TypeScript monorepo containing tools for building AI agents and managing LLM deployments. It provides a unified multi-provider LLM API, an agent runtime, an interactive CLI coding agent, a Slack bot, a terminal UI library, and web UI components.

## Repository Structure

```
pi-mono/
├── packages/
│   ├── ai/             # Unified LLM API (OpenAI, Anthropic, Google, Bedrock, etc.)
│   ├── agent/          # Agent runtime with tool calling and state management
│   ├── coding-agent/   # Interactive CLI coding agent ("pi")
│   ├── mom/            # Slack bot delegating to the coding agent
│   ├── tui/            # Terminal UI library with differential rendering
│   ├── web-ui/         # Web components for AI chat interfaces
│   └── pods/           # CLI for managing vLLM deployments on GPU pods
├── scripts/            # Release, versioning, and utility scripts
├── .github/            # CI workflows and contributor approval
├── .husky/             # Git hooks (pre-commit)
├── .pi/                # Pi configuration (extensions, prompts)
├── biome.json          # Linting/formatting configuration
├── tsconfig.base.json  # Shared TypeScript compiler options
└── tsconfig.json       # Root TypeScript config with path aliases
```

## Package Dependencies (Build Order)

Packages must be built in dependency order:
1. `tui` (no internal deps)
2. `ai` (no internal deps)
3. `agent` (depends on ai)
4. `coding-agent` (depends on agent, ai, tui)
5. `mom` (depends on coding-agent)
6. `web-ui` (depends on ai)
7. `pods` (no internal deps)

## Tech Stack

- **Runtime**: Node.js >= 20.0.0
- **Language**: TypeScript ^5.9.2 (strict mode, ES2022 target, Node16 modules)
- **Package Manager**: npm with workspaces
- **Build**: TypeScript compiler (`tsc`), `tsgo` for type checking
- **Linting/Formatting**: Biome 2.3.5
- **Testing**: Vitest (most packages), Node.js native test runner (tui)
- **Binaries**: Bun for standalone binary compilation
- **Git Hooks**: Husky (pre-commit)

## Essential Commands

### After code changes (not documentation)
```bash
npm run check    # Lint, format, and type-check (MUST run after build)
```
This runs `biome check --write --error-on-warnings` and `tsgo --noEmit`. Fix ALL errors, warnings, and infos before committing. Does NOT run tests.

### Running specific tests
```bash
# From the package root directory (e.g., packages/ai/):
npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

### Building
```bash
npm run build    # Build all packages in dependency order
```

### Commands to NEVER run directly
- `npm run dev` - starts dev servers, not useful for AI workflows
- `npm test` - runs all tests across all packages, too broad

## Code Style and Conventions

### Formatting (Biome)
- **Indentation**: Tabs (width 3)
- **Line width**: 120 characters
- **Module type**: ESM (`"type": "module"`)

### TypeScript Rules
- No `any` types unless absolutely necessary
- **No inline imports** - never use `await import("./foo.js")` or `import("pkg").Type`. Always use standard top-level imports
- Never remove/downgrade code to fix type errors from outdated deps; upgrade the dependency
- Check `node_modules` for external API type definitions instead of guessing
- Use `const` over `let` where possible (enforced by linter)

### Keybindings (coding-agent, tui)
- Never hardcode key checks with `matchesKey(keyData, "ctrl+x")`
- All keybindings must be configurable via `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS`

### Style
- No emojis in commits, issues, PR comments, or code
- Keep communication short, concise, technical
- No fluff or cheerful filler text

## Git Workflow

### Committing
- **NEVER** use `git add -A` or `git add .` - always add specific files
- Include `fixes #<number>` or `closes #<number>` when resolving an issue
- Never commit unless explicitly asked

### Forbidden Git Operations
- `git reset --hard` - destroys uncommitted changes
- `git checkout .` - destroys uncommitted changes
- `git clean -fd` - deletes untracked files
- `git stash` - stashes all changes including other agents' work
- `git commit --no-verify` - never allowed

### PR Workflow
- Analyze PRs without pulling locally first
- Never open PRs yourself; work in feature branches, then merge to main when approved

## Testing

- LLM-dependent tests are skipped by default (require API keys)
- `./test.sh` runs tests without API keys (used in CI)
- Always run tests from the **package root**, not the repo root
- When writing tests, run them and iterate until passing

## GitHub Issues

```bash
gh issue view <number> --json title,body,comments,labels,state
```

- Add `pkg:*` labels: `pkg:agent`, `pkg:ai`, `pkg:coding-agent`, `pkg:mom`, `pkg:pods`, `pkg:tui`, `pkg:web-ui`

## Changelog

Each package has its own `CHANGELOG.md`. New entries go under `## [Unreleased]`:

```markdown
### Breaking Changes
### Added
### Changed
### Fixed
### Removed
```

- Read the existing `[Unreleased]` section before adding entries to avoid duplicate subsections
- Never modify already-released version sections
- Attribution: `Fixed foo bar ([#123](https://github.com/badlogic/pi-mono/issues/123))`
- External: `Added feature X ([#456](https://github.com/badlogic/pi-mono/pull/456) by [@user](https://github.com/user))`

## Releasing

All packages use **lockstep versioning** (same version number, currently 0.52.12).

- `patch`: Bug fixes and new features
- `minor`: API breaking changes
- No major releases

```bash
npm run release:patch    # or release:minor
```

## Adding a New LLM Provider

Requires changes across multiple files. See `AGENTS.md` for the full checklist covering:
1. Core types in `packages/ai/src/types.ts`
2. Provider implementation in `packages/ai/src/providers/`
3. Stream integration in `packages/ai/src/stream.ts`
4. Model generation in `packages/ai/scripts/generate-models.ts`
5. Tests across multiple test files
6. Coding agent model resolver and CLI args
7. Documentation and changelog

## CI/CD

GitHub Actions workflows:
- **ci.yml**: Build, check, test on push/PR to main (Node 22, ubuntu-latest)
- **pr-gate.yml**: Contributor approval gate for PRs
- **build-binaries.yml**: Cross-platform binary builds on tag push

## File Reading Rule

Never use `sed`/`cat` to read files. Always use the read tool (with offset + limit for ranged reads). Read every file in full before editing.
