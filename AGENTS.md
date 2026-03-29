# Agent Instructions (for Codex and other agents)

## Git Workflow (MANDATORY)

- **NEVER push directly to `main`** — all changes go through pull requests
- Create a feature branch for every change: `feat/<topic>`, `fix/<topic>`, `refactor/<topic>`, `docs/<topic>`, `chore/<topic>`
- Push the feature branch, then create a PR
- Only the human operator may override this rule by explicitly requesting a direct push to main

## Commit Style

- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`
- English in code and commits

## Rules

- Never commit secrets (.env, credentials, API keys)
- Never force-push
- Never use localhost — dev access is via Tailscale (`devhost`)
