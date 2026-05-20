# Contributing to poc-hermes-server

Thank you for contributing. This document defines the engineering workflow for this project. All contributions must follow these guidelines to maintain code quality and traceability.

---

## Prerequisites

- Node.js 22 LTS
- Docker + Docker Compose
- Git

```bash
git clone <repo>
cd poc-hermes-server
cp .env.example .env
docker compose up -d
npm install
npm run db:migrate
npm run db:seed
npm run dev
```

---

## Branch Naming

All branches must follow:

```
<type>/<short-description>
```

| Type | When to Use |
|---|---|
| `feat/` | New feature |
| `fix/` | Bug fix |
| `refactor/` | Code restructure (no behavior change) |
| `docs/` | Documentation only |
| `chore/` | Dependency updates, config, tooling |
| `test/` | Tests only |
| `perf/` | Performance improvement |
| `security/` | Security fix |

Examples:
```
feat/websocket-gateway
fix/hal-command-timeout
docs/adr-008-conflict-resolution
security/fix-turn-credential-expiry
```

---

## Commit Convention

This project uses [Conventional Commits](https://www.conventionalcommits.org/).

```
<type>(<scope>): <short description>

[optional body]

[optional footer: BREAKING CHANGE, Closes #issue]
```

**Types**: `feat`, `fix`, `refactor`, `docs`, `chore`, `test`, `perf`, `security`

**Scopes**: `hal`, `api`, `websocket`, `messaging`, `webrtc`, `sync`, `auth`, `db`, `events`, `jobs`, `infra`, `docs`

Examples:
```
feat(hal): add SimulatedRadioDriver with configurable scenarios
fix(websocket): prevent unauthenticated connections past 10s window
security(hal): use execFile with array args for all CLI invocations
docs(adr): add ADR-008 conflict resolution strategy
```

commitlint enforces this format in CI. Configure the git hook locally:
```bash
npm run prepare  # installs husky hooks
```

---

## Pull Request Process

### Before Opening a PR

- [ ] All tests pass: `npm test`
- [ ] No lint errors: `npm run lint`
- [ ] No type errors: `npm run typecheck`
- [ ] No high/critical audit issues: `npm audit`
- [ ] New functionality has tests
- [ ] Documentation updated if behavior changed

### PR Size

Keep PRs small and focused. A PR should do one thing.

- Aim for <400 lines changed
- If a feature requires more, split into multiple sequential PRs
- Large schema changes should be separate from the code that uses them

### PR Description

Use the PR template (`.github/PULL_REQUEST_TEMPLATE.md`). Every PR must describe:
- What changed
- Why it changed
- How it was tested
- Any breaking changes

### Review Requirements

- All PRs require at least 1 approval before merging
- Security-related changes require 2 approvals
- Schema migrations require explicit DBA sign-off comment
- All CI checks must pass

### Merge Strategy

- Squash merge for feature branches (clean history)
- Merge commit for release branches
- No force-pushes to `main`

---

## Architecture Decision Records (ADRs)

Any decision that:
- Changes the technology stack
- Introduces a new dependency
- Changes database schema significantly
- Alters a security model
- Affects the public API contract

...requires an ADR **before** implementation.

Use the template at `docs/governance/ADR-TEMPLATE.md`. Open the ADR as a PR first, get it approved, then implement.

ADR numbering: sequential (`ADR-NNN`). Do not reuse numbers even if an ADR is superseded.

---

## RFCs (Request for Comments)

For significant platform changes that need broader input before commitment, open an RFC using `docs/governance/RFC-TEMPLATE.md`. RFCs are open for 2 weeks of comment before a decision is made.

---

## Security Issues

Do **not** open a public GitHub issue for security vulnerabilities.

Report security issues privately to the project maintainers via the contact defined in `SECURITY.md`. We will respond within 72 hours.

---

## Code Style

- TypeScript strict mode (`strict: true` in tsconfig)
- No `any` types without explicit comment justification
- No `console.log` — use `logger.info/debug/error` (Pino)
- No secrets hardcoded — use environment variables
- All new CLI invocations must use `execFile` with array args
- All new DB queries must use Drizzle ORM (no raw SQL in application code)
- MIME validation from content bytes, not `Content-Type` header

---

## Testing Requirements

| Type | Required For |
|---|---|
| Unit tests | All business logic, repositories, HAL driver methods |
| Integration tests | All API endpoints, WebSocket flows, BullMQ workers |
| Contract tests | Both `SBitxCLIDriver` and `SimulatedRadioDriver` must pass same tests |

Minimum coverage: 80% line coverage for new code (enforced in CI).

---

## Release Process

1. All planned PRs merged to `main`
2. `CHANGELOG.md` updated with release notes
3. Version bumped following [semver](https://semver.org/):
   - Patch: bug fixes
   - Minor: new features, backward-compatible
   - Major: breaking API changes
4. Git tag: `v1.2.3`
5. GitHub Release created from tag
6. Docker image built and pushed to registry
