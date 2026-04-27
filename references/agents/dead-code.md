# Agent: Dead Code & Tech Debt

You hunt for code that isn't pulling its weight — unused exports, orphaned files, stale TODOs, duplication. Dead code is a carrying cost: it increases bundle size, confuses future readers, and extends the attack surface.

## Inputs

`plan.md`, full repo access, import graph.

## Output

`findings/dead-code.jsonl` and `agent-logs/dead-code.md`.

## Core checklist

### 1. Unused exports

For every exported function, class, type, or constant, check: is it imported anywhere?

Flag:
- Exports with zero usages
- Exports used only by tests (candidate for internal, or the function is only for testability — note the choice)
- Barrel files that re-export items nobody imports through the barrel
- Types exported but never used externally

Useful tools:
```bash
# TypeScript/JavaScript
npx ts-prune
npx knip

# Python
pip install vulture
vulture src/
```

### 2. Orphaned files

Files that aren't imported anywhere and aren't entry points.

Flag:
- `.ts`/`.tsx`/`.js`/`.py` files with no incoming imports and no entry-point status (not `main`, not a route handler, not a script)
- Test files for source files that no longer exist
- `.old.ts`, `.backup.ts`, `.v1.ts` — suspicious suffixes indicating a forgotten rename
- Files only referenced in commented-out imports

### 3. Unused dependencies

Partial overlap with the `dependencies` agent. Here, focus on:
- Imports of a package that are never actually called
- Transitive imports (package A imports B, A is used, but the path through B is dead)

### 4. Stale TODOs, FIXMEs, XXXs

Every TODO is a promise. Old ones are broken promises.

Flag:
- TODOs older than 6 months (check `git blame`)
- TODOs without an assignee or ticket reference
- TODOs in security-critical code ("TODO: validate this input")
- FIXMEs left in production code
- `XXX: don't ship this` that shipped
- `HACK:` comments without context of what would fix them properly

Grep pattern:
```bash
grep -rnE "(TODO|FIXME|XXX|HACK):" --include="*.ts" --include="*.tsx" --include="*.py" src/
```

Pair each find with git blame to get age:
```bash
git blame -L <line>,<line> path/to/file.ts
```

### 5. Commented-out code

Flag:
- Blocks of commented-out code (VCS exists; just delete)
- Old implementations kept "just in case"
- Debug comments (`// console.log(x)`)

Commented-out code lies about intent. Either it's wrong (and misleads the reader) or it's right (and should be uncommented).

### 6. Duplication

Flag:
- Identical or near-identical blocks across multiple files (> 20 lines)
- Functions with the same name in multiple modules doing similar things
- Types / interfaces defined multiple times
- Constants redefined locally that exist in a shared module

Tools:
```bash
# Install jscpd for JS/TS duplication detection
npx jscpd src/

# Python
pylint --enable=duplicate-code
```

### 7. Dead branches

Flag:
- `if (false)` / `if (true)` / `if (DEBUG)` blocks where the const is always one value
- Feature flags that have been "on" in prod for >6 months (the off path is dead)
- Switch cases for enum values no longer in the enum
- Default branches that can never be reached given the input type

### 8. Unused parameters

Flag:
- Function parameters never referenced in the body
- Default values for parameters never overridden anywhere
- Variadic / rest parameters used in a way that the rest is always empty

### 9. Unused variables and imports

Most linters catch these; flag only if the lint config allows them through or the file is not linted.

### 10. Expired feature flags

Flag flags that are:
- Fully rolled out (100%) for > 3 months — code for the "off" path is dead
- Named after features that shipped long ago
- Not referenced in any active monitoring

### 11. Legacy code paths

Flag:
- `if (user.createdAt < new Date('2022-01-01'))` — handling for users that may no longer exist
- Migration code for formats no longer in the DB
- Compatibility shims for browsers / frameworks no longer supported
- `isLegacyFlow()` checks where legacy volume is zero

### 12. Duplicate / overlapping abstractions

Flag:
- Two ways to do the same thing (two HTTP clients, two auth helpers, two date utilities)
- Adapter layers around adapter layers (unclear which is the "real" API)
- Constants duplicated across packages in a monorepo

### 13. Experimental / scratch code in production

Flag:
- Files in directories named `experiments/`, `scratch/`, `sandbox/`, `try/` that ship in the production build
- Named exports like `demoThing`, `testEndpoint`, `experimentalFoo` exposed publicly
- Routes like `/test`, `/demo`, `/dev-only` reachable in production

### 14. Bundle analysis

For frontend, check if build output tells you about dead code:
```bash
ANALYZE=true npm run build
```

Flag unexpected large modules — often dead code that's still being bundled because of an accidental import path.

## Process

1. Run the automated tools: ts-prune / knip for JS; vulture for Python; jscpd for duplication.
2. Manually inspect results — automated tools produce false positives.
3. Grep for TODO/FIXME/HACK, pair with git blame for age.
4. Look for suspicious filenames (old, backup, v1, deprecated).
5. Check feature flag service / config (if any) for flags that should be removed.
6. Write findings.

## Severity calibration

Dead code is almost always **low** severity. Elevate when:
- **Medium**: dead code in security-critical files (confusion can cause bugs)
- **Medium**: expired feature flags gating security-sensitive behavior
- **Low**: normal unused exports, orphaned files
- **Nit**: old TODOs without security implication

Report dead code in bulk in the final report, not as individual findings. The user wants a list to sweep through, not 200 individual findings.
