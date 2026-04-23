---
name: upskill
description: Meta-skill that learns new capabilities from NPM and pip package registries. Use when user asks to integrate with a service (e.g., "I need to use Asana"), mentions an API we don't have a skill for, or explicitly invokes "/skill upskill <package>". Automatically searches NPM and pip, installs packages globally to ~/.claude/, and generates skill documentation. DEFAULT: Any time system needs to use an API, check package registries first.
---

# Upskill - Learn New Capabilities from Package Registries

## Purpose

Meta-skill that automatically learns new capabilities by searching NPM and pip registries, installing packages globally, and generating skill documentation. Enables the system to recursively teach itself new tools.

**How Skills Work:**
- Skills are **documentation/instructions** loaded into context via the Skill tool
- Main context reads the skill, then writes and executes scripts directly
- **No subagents** - execution happens in main context for simplicity and speed
- Skills are lightweight reference material (~20-30 lines), full docs separate

## When to Use

**Auto-trigger on:**
- User asks to integrate with a service: "I need to use Asana", "Connect to Slack"
- User mentions an API/system we don't have a skill for
- Explicit invocation: `/skill upskill <package-name>`
- **DEFAULT**: Any time system needs to use an API -> check package registries first

## Process

### Step 1: Search Both Registries (NPM first, then pip)

```bash
# Search NPM
curl -s "https://registry.npmjs.org/-/v1/search?text=<package-name>&size=20" | jq '.objects[] | {name: .package.name, version: .package.version, description: .package.description, downloads: .package.downloadsLastWeek}'

# Search PyPI (package must exist for direct query)
curl -s "https://pypi.org/pypi/<package-name>/json" | jq '{name: .info.name, version: .info.version, description: .info.summary}'
```

**Selection criteria:**
- Official/maintained packages
- Download counts and maintenance frequency
- TypeScript definitions (NPM) or type stubs (pip)
- Recently updated

### Step 2: Fetch Package Info

**NPM packages:**
```bash
# Get full metadata
curl -s "https://registry.npmjs.org/<package-name>" | jq > npm-metadata.json

# Key fields: .versions, .dist.tarball, .readme, .types, .repository
```

**pip packages:**
```bash
# Get full metadata
curl -s "https://pypi.org/pypi/<package-name>/json" | jq > pypi-metadata.json

# Key fields: .info, .releases, .urls, .project_urls
```

### Step 3: Install Package Globally

**CRITICAL:** Install to `~/.claude/` (not project-specific, not system-wide)

**For NPM packages:**
```bash
cd ~/.claude
npm install <package-name>
# Verifies: ~/.claude/node_modules/<package-name>/
```

**For pip packages:**
```bash
pip install --target ~/.claude/lib <package-name>
# Verifies: ~/.claude/lib/python3.X/site-packages/<package-name>/
```

### Step 4: Analyze API Surface

Extract:
- Main exports/functions/classes
- Authentication method
- Common use cases
- Required environment variables
- Typical code patterns

### Step 5: Generate Skill Files

Create:
```text
~/.claude/skills/<package-name>/
├── SKILL.md
├── full-docs.md
└── metadata.json
```

**SKILL.md** should include:
- Package purpose
- Install location
- Main API functions with 1-line descriptions
- Auth requirements
- Simple example
- Trigger conditions

**metadata.json** should include:
- Package name/version
- Registry source (npm or pip)
- Install path
- Capabilities list
- Auth type
- Last updated

### Step 6: Test Installation

```bash
# NPM
node -e "const pkg = require('<package-name>'); console.log('OK')"

# pip
python3 -c "import <package-name>; print('OK')"
```

### Step 7: Report Success

Tell user:
- Which package was selected and why
- Install path
- Skill path
- Auth setup needed (if any)
- Example usage

## Rules

- Always check NPM first, then pip
- Prefer official, maintained packages
- Install to `~/.claude/`, never project-local
- Generate minimal `SKILL.md`, put verbose docs in `full-docs.md`
- Test before claiming success
- If auth is required, explain exactly how to get credentials
- If both registries have good options, choose the one with better docs/tooling
