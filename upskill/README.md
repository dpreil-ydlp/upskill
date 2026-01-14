# Upskill - Auto-Learn Capabilities from Package Registries

Meta-skill that enables Claude Code to teach itself new tools by searching NPM and pip registries, installing packages globally, and generating skill documentation automatically.

## What It Does

**Upskill** recursively expands Claude Code's capabilities by:
- Searching NPM and pip for any package/service
- Installing packages globally to `~/.claude/`
- Generating skill documentation automatically
- Creating ready-to-use integration skills

## When to Use

Use upskill when:
- User asks to integrate with a service: "I need to use Asana", "Connect to Slack"
- User mentions an API we don't have a skill for
- Explicitly invoked: `/skill upskill <package-name>`
- **Default behavior**: Check package registries before implementing from scratch

## How It Works

```
User Request → Search Registries → Install Package → Analyze API → Generate Skill → Ready to Use
```

### 1. Search Both Registries

Searches NPM and pip simultaneously to find the best package:

```bash
# NPM search
curl "https://registry.npmjs.org/-/v1/search?text=<query>&size=20"

# pip search (via PyPI API)
curl "https://pypi.org/pypi/<package-name>/json"
```

**Selection criteria:**
- Official/maintained packages
- Download counts and maintenance frequency
- TypeScript definitions (NPM) or type stubs (pip)
- Recent updates

### 2. Install Globally

**Always installs to `~/.claude/`** (not project-specific):

```bash
# NPM packages
cd ~/.claude && npm install <package-name>
# → ~/.claude/node_modules/<package>/

# pip packages
pip install --target ~/.claude/lib <package-name>
# → ~/.claude/lib/python3.X/site-packages/<package>/
```

### 3. Generate Skill Structure

Creates three files per package:

```
~/.claude/skills/<package-name>/
├── SKILL.md              # Main skill (loaded into context, ~20-30 lines)
├── full-docs.md          # Complete API reference
└── metadata.json         # Machine-readable metadata
```

**SKILL.md** (minimal, context-efficient):
- Package overview and installation path
- Quick reference for main functions
- Authentication requirements

**full-docs.md** (comprehensive):
- Full API documentation
- Code examples
- Advanced usage patterns

**metadata.json**:
- Version tracking
- Capabilities and triggers
- Credential requirements

### 4. Handle Authentication

If package requires credentials:

1. Identify auth mechanism (API key, OAuth, token)
2. Provide URL to get credentials
3. Guide user to set environment variable
4. Test credentials before confirming success

```bash
export CLAUDE_<SERVICE>_TOKEN=your_token_here
```

### 5. Test and Confirm

Verifies installation and functionality:

```bash
# Test NPM package
node -e "const pkg = require('<package>'); console.log('OK');"

# Test pip package
python3 -c "import <package>; print('OK')"
```

## Package Selection Strategy

When both NPM and pip have options:

**Prefer NPM if:**
- Official package with better TypeScript definitions
- More downloads/active maintenance
- Recently updated
- Better documentation

**Prefer pip if:**
- Python-only service or domain
- Official SDK maintained by service
- Better Python ecosystem integration

**Ask user if:**
- Tie in quality
- Different capabilities between versions

## Example Workflow

**User**: "I need to work with Notion"

**Upskill execution:**

1. **Search**: Finds `@notionhq/client` on NPM
   - Official package ✓
   - TypeScript definitions ✓
   - 300K weekly downloads ✓

2. **Install**: `npm install @notionhq/client` → `~/.claude/node_modules/`

3. **Analyze**:
   - Main export: `Client`
   - Auth: Requires integration token
   - Key methods: `databases.query`, `pages.retrieve`, etc.

4. **Generate**:
   ```
   ~/.claude/skills/notion/
   ├── SKILL.md
   ├── full-docs.md
   └── metadata.json
   ```

5. **Credentials**:
   ```
   Get token: https://www.notion.so/my-integrations
   Set: export CLAUDE_NOTION_TOKEN=your_token
   ```

6. **Confirm**:
   ```
   ✓ Learned @notionhq/client v2.2.3 (NPM)
   ✓ Skill: ~/.claude/skills/notion/
   ✓ Runtime: Node.js
   ✓ Auth: Set CLAUDE_NOTION_TOKEN
   ```

**Now the skill is available for use:**

```bash
/skill notion list databases
```

## How Generated Skills Work

When a generated skill is invoked:

1. **Skill tool** loads `~/.claude/skills/<package>/SKILL.md` into context
2. **Main context** reads the skill documentation
3. **Main context** writes a script (e.g., `query-notion.js`)
4. **Main context** executes the script directly: `node query-notion.js`
5. **Results** returned immediately

**No subagents** - simple, fast, direct execution in main context.

## Registry APIs

### NPM Registry

```bash
# Search packages
curl "https://registry.npmjs.org/-/v1/search?text=<query>&size=20"

# Get package info
curl "https://registry.npmjs.org/<package-name>"

# Get specific version
curl "https://registry.npmjs.org/<package-name>/<version>"
```

### PyPI Registry

```bash
# Get package info
curl "https://pypi.org/pypi/<package-name>/json"

# Search (no simple API, use web search or alternative)
```

## Type Parsing

### NPM (TypeScript)
- Parse `.d.ts` files for function signatures
- Extract exported classes and interfaces
- Look for JSDoc comments
- Fall back to README examples if no types

### pip (Python)
- Parse `.pyi` type stub files
- Extract docstrings from modules
- Inspect function signatures
- Fall back to README examples

## Error Handling

| Error | Solution |
|-------|----------|
| Package not found | Try alternative registry, suggest similar names |
| Install fails | Check permissions, verify network, show error |
| No types available | Warn but proceed, use README examples |
| Auth required | Prompt for credentials, provide setup instructions |

## Important Notes

- **Always install to `~/.claude/`** (never project-specific)
- Keep `SKILL.md` minimal (~20-30 lines) for context efficiency
- Full documentation goes in `full-docs.md`
- Test before confirming success
- Inform user of alternatives in other registry
- Document auth requirements clearly
- Update version cache for all packages
- **Skills are documentation, execution is direct and simple**

## Directory Structure

```
~/.claude/
├── node_modules/          # Globally installed NPM packages
├── lib/                   # Globally installed pip packages
├── skills/                # All skills (generated + manual)
│   ├── upskill/          # This meta-skill
│   ├── asana/            # Generated skill example
│   ├── slack/            # Generated skill example
│   └── ...
└── skill-cache/          # Version tracking
    └── package-versions.json
```

## Related Files

- `SKILL.md` - Main skill documentation (loaded into context)
- `metadata.json` - Machine-readable package metadata
- `full-docs.md` - Complete API reference (if needed)

## Version Cache Format

```json
{
  "<package-name>": {
    "installed": "X.Y.Z",
    "latest": "X.Y.Z",
    "last_checked": "2025-01-13T10:00:00Z",
    "update_available": false,
    "critical": false,
    "breaking": false
  }
}
```

## Metadata Format

```json
{
  "name": "<package-name>",
  "package": "<package-name>",
  "version": "X.Y.Z",
  "runtime": "node" | "python",
  "registry": "npm" | "pypi",
  "last_checked": "2025-01-13T10:00:00Z",
  "last_updated": "2025-01-13T10:00:00Z",
  "capabilities": ["cap1", "cap2"],
  "triggers": ["keyword1", "keyword2"],
  "credential_env": null | "CLAUDE_<SERVICE>_TOKEN",
  "install_path": "~/.claude/node_modules/<package>/",
  "requires_auth": false | true
}
```

## Usage Examples

```bash
# Explicit invocation
/skill upslack slack

# Implicit usage (when mentioning an API)
"I need to integrate with Asana"
→ Automatically searches, installs, and generates skill

# Search for specific functionality
"Find a package for PDF generation"
→ Searches both registries, presents options
```

## Contributing

To add new capabilities to upskill:
1. Edit `SKILL.md` with improved instructions
2. Add registry search patterns as needed
3. Update type parsing logic for new languages
4. Test with real package installations

## License

Part of Claude Code skills system. Follows main project license.
