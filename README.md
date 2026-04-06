# my plugins

Personal Claude Code plugin with workflow skills and safety hooks.

## Components

| Name | Type | Description |
|------|------|-------------|
| `adaptive-prompt-copilot` | Skill | Interviews you adaptively and generates cognitively-grounded prompts |
| `devlog` | Skill | Writes a detailed devlog entry for the latest commit |
| `plugin-scout` | Skill | Browses configured marketplaces and recommends plugins for your workflow |
| `secrets-sentinel` | Hook | Blocks writes to secrets files (.env, keys, credentials) |
| `danger-guard` | Hook | Confirms before running destructive shell and git commands |

## Installation

**1. Register the marketplace** in `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "workflow-plugins": {
      "source": {
        "source": "github",
        "repo": "ogngnaoh/workflow-plugins"
      }
    }
  }
}
```

**2. Install the plugin:**

```
/plugin install personal-tools
```

**3. Verify** by running `/plugin list` — you should see `personal-tools` listed.

## Skills Reference

### `adaptive-prompt-copilot`

Universal prompt engineering co-pilot. Interviews you about your goal, then generates a structured, cognitively-grounded prompt tailored to it.

**Trigger phrases:** "help me learn", "teach me", "write me a prompt", "I want to understand", "create a prompt", "I'm stuck on", or when you're frustrated with shallow AI responses.

---

### `devlog`

Generates a detailed session-context devlog entry for the latest commit — what changed, why, what was tried, what failed, and lessons learned.

**Trigger phrases:** "devlog", "log this commit", "write a devlog entry", "document what we did".

---

### `plugin-scout`

Browses all configured Claude Code plugin marketplaces and recommends which plugins to install based on your workflow needs.

**Trigger phrases:** "what plugins should I use", "find me plugins for X", "which plugins would help", "what's available in the marketplace".

## Hooks Reference

### `secrets-sentinel`

**Trigger:** PreToolUse on Edit, Write, Bash

Blocks any write to sensitive files. Allows reads and allows template variants.

**Blocked patterns:** `.env`, `.env.*`, `credentials.json`, `secrets.json`, `service-account*.json`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa`, `id_ed25519`

**Allowed exceptions:** `.env.example`, `.env.sample`, `.env.template`

---

### `danger-guard`

**Trigger:** PreToolUse on Bash

Intercepts destructive commands and requires explicit confirmation before proceeding.

**Guarded commands:**
- `rm -rf` (except safe targets: `node_modules`, `dist`, `build`, `__pycache__`, `.cache`, `coverage`)
- `git reset --hard`
- `git push --force`
- `git branch -D`
- `DROP TABLE`, `DROP DATABASE`
- `docker system prune`
- `chmod 777`

## Updating

```
/plugin update hoang-workflows
```

## Development

Clone the marketplace repo and modify skills/hooks locally:

```bash
git clone https://github.com/ogngnaoh/workflow-plugins
cd workflow-plugins
```

Skills live in `skills/<skill-name>/`, hooks config is in `hooks/hooks.json`. After editing, reload Claude Code for changes to take effect.
