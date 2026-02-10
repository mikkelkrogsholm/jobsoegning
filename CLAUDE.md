# Claude Code Instructions

## Project overview

This is a job search automation project for Danish job portals. It uses Claude Code skills with `playwright-cli` for browser automation to search job sites, extract listings, and present structured results.

## Key directories

- `.claude/skills/` — Claude Code skills (one per job portal)

## Skills

Each job portal has a skill in `.claude/skills/<portal>-search/`:
- `SKILL.md` — The skill definition with workflow instructions
- `url-reference.md` — URL patterns and filter documentation (if applicable)

## Playwright-cli patterns

All browser automation uses `playwright-cli`. Key rules:

1. **Always snapshot before clicking** — element refs change between snapshots
2. **Never press Escape in filter panels** — it closes the panel and loses state
3. **After autocomplete selection, press Tab** to dismiss the dropdown
4. **Use named sessions** (`-s=<name>`) when running multiple browsers
5. **Prefer URL construction** over UI interaction when possible — faster and more reliable
6. **Use `run-code`** for JavaScript extraction: `playwright-cli run-code 'async page => { ... }'`

## Adding a new job portal

1. Explore the site with `playwright-cli` and document findings
2. Create the skill: `.claude/skills/<portal>-search/SKILL.md`
3. Follow the format of `.claude/skills/jobindex-search/SKILL.md`

## Language

- Code, skills, and technical docs: English
- User-facing text (README, CONTRIBUTING): Danish

## Do not

- Do not commit `.playwright-cli/` session data
- Do not commit temporary result files (e.g. `results-scrolled.yaml`)
- Do not hardcode element refs — they change per session
