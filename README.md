# mcmaster-fast

mcmaster-fast is a Cursor agent skill: written instructions for the coding agent, not an application. When you ask Cursor to make a website feel fast, the skill tells it to follow the same order of work as [mcmaster.com](https://www.mcmaster.com/). Send HTML the browser can paint immediately, keep the page shell on screen, and fetch the next page when someone is about to open it.

Nothing is compiled, and there is no server to start.

## What you need

- [Cursor](https://cursor.com)
- No extra language runtime, database, or API key

Cursor reads `SKILL.md` when a request matches the skill. While it works, it also reads `techniques.md` and `recipes.md`.

## Download and use

For every project on your computer:

```bash
git clone https://github.com/ahmedbinirfan/mcmaster-fast.git ~/.cursor/skills/mcmaster-fast
```

For one project only:

```bash
mkdir -p .cursor/skills
git clone https://github.com/ahmedbinirfan/mcmaster-fast.git .cursor/skills/mcmaster-fast
```

Start a new Cursor chat so it picks up the new skill. Then ask it to make a site feel instant, or mention McMaster-Carr, hover prefetch, or an app shell. You do not run a start command. The next website task in Cursor is how the skill runs.
