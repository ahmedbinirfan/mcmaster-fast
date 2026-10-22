# mcmaster-fast

Cursor agent skill for making a website feel like [mcmaster.com](https://www.mcmaster.com/): HTML the browser can paint immediately, a shell that stays mounted, and the next page fetched on intent.

## Install

**Cursor (personal, all projects):**

```bash
mkdir -p ~/.cursor/skills/mcmaster-fast
curl -fsSL https://raw.githubusercontent.com/ahmedbinirfan/mcmaster-fast/main/SKILL.md -o ~/.cursor/skills/mcmaster-fast/SKILL.md
curl -fsSL https://raw.githubusercontent.com/ahmedbinirfan/mcmaster-fast/main/techniques.md -o ~/.cursor/skills/mcmaster-fast/techniques.md
curl -fsSL https://raw.githubusercontent.com/ahmedbinirfan/mcmaster-fast/main/recipes.md -o ~/.cursor/skills/mcmaster-fast/recipes.md
```

**One project:** copy `SKILL.md`, `techniques.md`, and `recipes.md` into `.cursor/skills/mcmaster-fast/`.

Cursor loads the skill from `SKILL.md`. The other two files are the recipes it reads while applying a rung.

## What it covers

HTML-first render, app-shell navigation, intent prefetch, layered caching, critical CSS, page-specific JS, and layout stability. Stack-specific mappings are in `recipes.md`.
