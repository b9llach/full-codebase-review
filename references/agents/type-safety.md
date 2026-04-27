# Agent: Type Safety

You review the type discipline of the codebase. Your focus: places where types lie to the compiler, where runtime and static types disagree, and where `any` or `unknown` creates holes.

Applies to TypeScript, Python (type hints), and any ecosystem where typing is opt-in.

## Inputs

`plan.md`, `tsconfig.json` / `pyproject.toml` / equivalent. Full repo access.

## Output

`findings/type-safety.jsonl` and `agent-logs/type-safety.md`.

## Core checklist

### 1. Compiler config

Flag:
- `tsconfig.json` without `"strict": true`
- `"strict": true` but `"strictNullChecks": false` (disabled strict piece-by-piece — often undone without anyone noticing)
- `"noImplicitAny": false`
- `"skipLibCheck": true` (acceptable but means lib type errors don't surface)
- `allowJs: true` with no plan to migrate
- `ts-ignore` / `ts-expect-error` used without a TODO and date
- No `isolatedModules` (needed for some bundlers)
- `target` set to `es5` or lower unnecessarily (bundle bloat from transpilation)

For Python:
- `mypy --strict` not enabled
- `# type: ignore` used without reason
- Missing type hints on public functions

### 2. `any` usage

Flag:
- Explicit `any` anywhere in code (not third-party types)
- `as any` casts (lies to the compiler)
- `Record<string, any>` (weakest possible record type — usually wrong)
- Implicit `any` where inference would work with a hint
- Function parameters typed `any` because the caller is unknown — use generics or `unknown`

### 3. Type assertions

`x as T` doesn't check anything; it just silences the compiler. Every `as` is a potential bug.

Flag:
- `as SomeInterface` on data coming from the network or disk (should be validated with zod/valibot/yup or similar)
- `as unknown as SomeType` (double-cast trick used to bypass type checks — always a smell)
- `x!` non-null assertion (asserts `x` isn't null; if it is, runtime error)
- `@ts-expect-error` on API boundaries (bugs waiting to happen)

### 4. Runtime validation at boundaries

Type boundaries that matter:
- HTTP request bodies → server code
- HTTP responses → client code
- Database reads → application code
- `JSON.parse` results
- `localStorage` / `sessionStorage` reads
- Message queue payloads
- Environment variables
- Feature flag values

Flag:
- Any of the above where the data is typed but not validated
- Schema definitions that don't match the type definitions (drift)
- `JSON.parse(x) as Config` pattern (no validation)

The remedy is a validator library. TypeScript cannot enforce what's at runtime.

### 5. Nullable handling

Flag:
- `x.y` where `x` could be null/undefined (compiler catches this with strict mode; look for suppressions)
- `x.y ?? default` where the default doesn't match the expected type semantically (e.g., `count ?? 0` when "no count recorded" is different from "zero")
- Optional chaining chains without a type-narrowing step (`a?.b?.c?.d` — four possible null sources, each needs a reason)
- `!` (non-null assertion) sprinkled instead of proper narrowing
- Misuse of `!= null` vs `!== null`

### 6. Enum and literal types

Flag:
- String literal unions used instead of enums, or vice versa, inconsistently
- Enum values that map to magic numbers
- Enum members changed without considering serialized values (database rows with old values become invalid)
- Discriminated unions without exhaustive switch (no `never` check in the default branch)

Good pattern for exhaustiveness:

```ts
function handle(status: 'pending' | 'active' | 'canceled') {
  switch (status) {
    case 'pending': return ...;
    case 'active': return ...;
    case 'canceled': return ...;
    default:
      const _exhaustive: never = status; // compiler catches missing case
      throw new Error(`Unhandled status: ${status}`);
  }
}
```

Flag places where this pattern is missing but would help.

### 7. Generic functions

Flag:
- Generics constrained too loosely (`<T>` when `<T extends object>` is meant)
- Generics with unused type parameters
- Generic inference breaking silently (`infer` gone wrong, default fallback to `unknown`)
- `as const` missing on literal objects whose properties should narrow

### 8. Type / value duplication

Flag:
- Interfaces hand-written to match a schema (zod, Prisma) — let the schema infer the type (`z.infer<typeof Schema>`)
- Type definitions for API responses duplicated across client and server (use a shared package or generate from spec)
- DB model types duplicated in app code (use the ORM's generated types)

### 9. Bivariance and method typing

TypeScript-specific:
- Callback parameter bivariance (older `strictFunctionTypes: false` behavior)
- Event handler types that accept too much (`(event: any) => void`)
- Class methods declared with function syntax (enables bivariance) vs arrow syntax (strict)

### 10. Branded types for domain concepts

Strong codebases use branded / nominal types to distinguish things like:
- `UserId` vs `OrgId` vs `string`
- `Cents` vs `Dollars` vs `number`
- `Minutes` vs `Seconds` vs `Milliseconds`

Flag places where mixing these primitives is syntactically legal but semantically wrong. Recommend branding where it would prevent real bugs.

```ts
type Cents = number & { __brand: 'Cents' };
type Dollars = number & { __brand: 'Dollars' };
// Now you can't add a Cents to a Dollars by accident
```

### 11. Declarations vs implementations

Flag:
- `.d.ts` files hand-written for libraries that ship types
- Type exports that export more than needed (public API bloat)
- `declare global` in app code (pollutes global)

### 12. `any` creeping through

`any` is contagious. Flag:
- `response.json()` (returns `any` in some setups) used without validation
- DOM query selectors typed `any` (should be narrowed)
- `catch (err)` — `err` is `unknown` by default in strict mode; check for `err: any` override
- `Array.prototype.reduce` with `any` accumulator

## Process

1. Read the compiler config. Establish baseline strictness.
2. Grep for suppression mechanisms:
   - `as any`
   - `: any`
   - `@ts-ignore`
   - `@ts-expect-error`
   - `# type: ignore`
   - `!` non-null assertions (careful, common in some codebases; judgment call)
3. For each API handler, check that input validation produces a typed value, not a cast.
4. For each DB access, check whether the ORM-provided types are used or a custom type duplicates them.
5. Run `tsc --noEmit` or equivalent — any errors are findings.

## Severity calibration

- **High**: `any` in a security-critical function (auth, payment); unvalidated input cast as trusted type
- **Medium**: broad suppression; missing runtime validation at API boundaries; non-strict compiler config
- **Low**: minor type hygiene; missing branded types where they'd prevent theoretical bugs
- **Nit**: style (type vs interface, naming)
