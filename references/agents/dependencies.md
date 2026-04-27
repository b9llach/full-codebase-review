# Agent: Dependencies

You review the supply chain. Your job: find vulnerable, outdated, unnecessary, or risky dependencies.

## Inputs

`package.json`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `requirements.txt`, `pyproject.toml`, `Pipfile.lock`, `Cargo.lock`, `go.sum`, etc.

## Output

`findings/dependencies.jsonl` and `agent-logs/dependencies.md`.

## Core checklist

### 1. Known vulnerabilities

Run the ecosystem's audit tool if available:

```bash
# Node
npm audit --json
pnpm audit --json
yarn npm audit --json

# Python
pip-audit
safety check

# Rust
cargo audit

# Go
govulncheck ./...
```

Every reported high/critical vulnerability is a finding. Reference the CVE ID.

If the tool isn't available in the review environment, grep for the top 20 dependencies and check known CVE databases via web search for critical findings.

### 2. Version pinning

Flag:
- Dependencies pinned with `^` or `~` on security-sensitive packages (acceptable with lockfile, but review)
- Dependencies pinned to specific versions with no lockfile (drift over CI runs)
- Dependencies pointing to git commits (supply chain risk if the branch moves)
- Dependencies with `file:` or `link:` to siblings in a non-monorepo context (confusing)

### 3. Outdated dependencies

Flag:
- Major versions behind latest (security patches often only on latest)
- Framework version behind current LTS (Next.js, React, Node, Django, Rails)
- Unmaintained packages (last publish > 2 years, open issues accumulating, maintainer inactive)

Check with:
```bash
npm outdated
pnpm outdated
```

### 4. Unused dependencies

Flag packages in `dependencies` / `devDependencies` that aren't imported anywhere.

```bash
# Node
npx depcheck

# Python
pip-extra-reqs
```

Every unused dep is bundle size and attack surface.

### 5. Misplaced dependencies

Flag:
- Runtime dependencies in `devDependencies` (will fail in production build)
- Dev-only dependencies in `dependencies` (bundled unnecessarily)
- Dependencies used only in tests in `dependencies`
- TypeScript types (`@types/*`) in `dependencies` instead of `devDependencies`

### 6. Duplicate versions

Flag:
- Same package at multiple major versions in the dep tree (bloat, confusion)
- Use `npm ls <pkg>` or `pnpm why <pkg>` to trace

### 7. License compliance

If the project has licensing constraints (commercial, internal-only):
- Flag GPL / AGPL / SSPL dependencies if the project can't comply
- Flag packages without declared licenses
- Flag custom / non-standard licenses

Check with:
```bash
npx license-checker --summary
```

### 8. Hand-rolled reimplementations

Flag:
- Custom crypto instead of `node:crypto` / `@noble/*` / libsodium
- Custom JWT handling instead of `jose` or similar
- Custom rate limiter instead of `rate-limiter-flexible`
- Custom validator instead of zod/valibot/yup
- Custom date math (consider `date-fns` / `luxon` — though flag the opposite too: `moment.js` is legacy, prefer alternatives)
- Custom HTTP client when `fetch` / `axios` / `ky` would do

This cuts both ways — sometimes rolling your own is the right call for bundle size. Flag the choice, not the direction; let the user decide.

### 9. Dangerous dependencies

Flag presence of:
- `eval`-based packages
- `node-ipc` and similar "protestware" packages
- Typosquatted or suspicious names (`lodahs`, `express-helper`, etc.)
- Packages that fetch code at runtime
- Packages with known recent supply chain incidents

### 10. Scripts and postinstall

Flag:
- `postinstall` scripts in dependencies (arbitrary code execution on install)
- `package.json` scripts using `curl | sh` patterns
- Scripts that read/write outside the project directory

To see all postinstall scripts:
```bash
npm ls --all --json | jq '.dependencies | .. | .scripts? | objects | select(has("postinstall"))'
```

(Or just grep lockfile for "postinstall".)

### 11. Dependency tree depth

A very deep dep tree (hundreds of transitive deps for a small app) is a smell.

### 12. Ecosystem-specific

**Node:**
- Packages without TypeScript types (if the project is TS) — slows development, no type checking
- ESM vs CJS confusion (check `"type": "module"` in package.json, extensions, interop)
- Packages using deprecated Node APIs (`Buffer()` constructor, `crypto.createCipher`)

**Python:**
- Pinned to Python versions EOL (< 3.10 now has limited support)
- `requirements.txt` vs `pyproject.toml` — which is source of truth?
- Virtual env management (poetry / uv / pipenv / hatch / rye) — flag mixed tools

**Rust:**
- `[patch]` overrides — need rationale
- Default features on heavy crates (often unnecessary)

**Go:**
- `replace` directives in go.mod pointing to forks — need rationale

### 13. CI on dependency changes

Flag:
- No Dependabot / Renovate config
- No `npm audit` / equivalent in CI
- No lockfile check in CI (PRs can change lockfile without the package.json change)

## Process

1. Read all manifest files.
2. Run audit if possible.
3. Run depcheck / unused-deps detection.
4. Cross-reference the top 20 dependencies against recent vulnerability announcements (web search if needed).
5. Write findings.

## Severity calibration

- **Critical**: known CVE, critical severity, on current version, in prod code path
- **High**: known CVE, high severity; unmaintained critical dep; secrets-handling package that's outdated
- **Medium**: outdated but no known CVE; duplicate version bloat; missing audit in CI
- **Low**: unused deps; misplaced deps; minor version drift
- **Nit**: version range style preferences
