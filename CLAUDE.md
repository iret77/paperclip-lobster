# paperclip-plugin-lobster — Project Context

## Git Workflow

### Branch Protection (MANDATORY)
- **NEVER push directly to `main`** — all changes go through pull requests
- Create a feature branch for every change: `feat/<topic>`, `fix/<topic>`, `refactor/<topic>`
- Push the feature branch, then create a PR via `gh pr create`
- Only the user (Christian) may override this rule by explicitly saying "push to main" or "direkt auf main"
- If unsure, always create a PR

### Branch Naming
- `feat/<topic>` — new features
- `fix/<topic>` — bug fixes
- `refactor/<topic>` — refactoring
- `docs/<topic>` — documentation only
- `chore/<topic>` — build, config, tooling

### Commit Style
- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- Always end with: `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
- German communication with user, English in code/commits

### PR Process
- PR title: short, conventional commit style (e.g. `feat: add constraint engine`)
- PR body: use the pull request template (What/Why/How + checklist)
- After creating PR, report the URL to the user

## Tech Stack
- TypeScript / Node.js (ES modules)
- Paperclip Plugin API v1
- pnpm for package management

## NEVER Rules
- **Never push to main without explicit user override**
- **Never use localhost** — dev access is via Tailscale (`devhost`)
- **Never commit secrets** (.env, credentials, API keys)
