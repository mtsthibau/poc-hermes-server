## Description

<!-- 
Describe what this PR does and why.
Link to the issue(s) this resolves.

Closes #
Related: #
-->

---

## Type of Change

<!-- Mark the types that apply -->

- [ ] `feat` — New feature
- [ ] `fix` — Bug fix
- [ ] `security` — Security fix
- [ ] `refactor` — Code restructure (no behavior change)
- [ ] `perf` — Performance improvement
- [ ] `test` — Tests only
- [ ] `docs` — Documentation only
- [ ] `chore` — Dependency update, config, tooling
- [ ] Breaking change (existing functionality changes)
- [ ] Database migration included

---

## What Changed

<!--
Summarize the technical changes. Be specific.
For reviewers who need to understand the change without running it.
-->

---

## How Was It Tested

<!-- 
Describe how you tested this change.
- Unit tests added? Which scenarios?
- Integration tests? Which endpoints/flows?
- Manual testing? What did you do?
- Tested with RADIO_DRIVER=simulated? With real hardware?
-->

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Tested with `RADIO_DRIVER=simulated`
- [ ] Tested with real hardware (if applicable)
- [ ] Manual testing performed — describe below:

---

## Security Considerations

<!-- 
Required for any PR touching: auth, HAL CLI, file handling, WebSocket, RBAC, secrets.
Check all that apply and confirm they were addressed.
-->

- [ ] No new CLI invocations (or new ones use `execFile` with array args)
- [ ] No user input concatenated into SQL, shell commands, or file paths
- [ ] No new secrets introduced (or secrets added to env vars only, not code)
- [ ] No RBAC changes (or RBAC changes reviewed in description above)
- [ ] `npm audit` shows no new high/critical vulnerabilities

---

## Database Changes

<!-- Skip if no DB changes -->

- [ ] New migration file added under `drizzle/migrations/`
- [ ] Migration is backward-compatible (no destructive changes to existing data)
- [ ] Indexes added for new query patterns
- [ ] Rollback plan documented in migration file comments

---

## Breaking Changes

<!-- Skip if not a breaking change -->

- [ ] API endpoint changed — describe in description
- [ ] WebSocket protocol changed — `websocket-protocol.md` updated
- [ ] Environment variable added/renamed — `.env.example` updated

---

## Checklist

- [ ] `npm run lint` passes
- [ ] `npm run typecheck` passes  
- [ ] `npm test` passes
- [ ] `npm audit` shows no new high/critical vulnerabilities
- [ ] Branch is up to date with `main`
- [ ] Conventional commit messages used
- [ ] Documentation updated (if behavior changed)
- [ ] ADR created (if architectural decision made)
