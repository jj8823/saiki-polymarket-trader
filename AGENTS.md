## Learned User Preferences
- GitHub username is `jj8823`.
- Prefers the standard Fork workflow (origin for personal fork, upstream for original repo) when managing and syncing with third-party codebases.
- Prefers distributing custom Cursor skills to team members via the `npx skills add` CLI command using their GitHub repository URL.

## Learned Workspace Facts
- Polymarket API integrations must use the V2 SDK (`py_clob_client_v2`) and the Deposit Wallet architecture (requiring `SignatureTypeV2.POLY_1271` for signing).
- Cursor skills are defined statically in local `SKILL.md` files (e.g., under `.claude/skills/`) and do not automatically sync with external API documentation changes.