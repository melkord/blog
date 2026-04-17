---
title: "From 40 to 16 minutes: migrating a NestJS test suite from Jest to Bun"
description: "How we parallelized a Bitbucket Pipeline and migrated the test runner from Jest to Bun in a NestJS monorepo. Two rounds of optimization, five incremental PRs, and a few surprises along the way."
pubDate: 2026-01-04
tags: ["bun", "testing", "ci-cd", "nestjs"]
---

Our CI pipeline took 40 minutes. For a team of five, that meant a lot of staring at Bitbucket waiting for green. Every PR, every fix, every "just a small change" came with a 40-minute tax.

This post covers two rounds of optimization: restructuring the pipeline to run stages in parallel, then migrating the test runner from Jest to Bun. What changed, what broke, and how I structured the work to avoid blocking the team.

## The setup

TypeScript monorepo with Yarn workspaces. NestJS backend, two Nuxt frontends, a media microservice, a React Native app, and two shared libraries. About 1600 unit tests and 200 E2E tests across 302 files. CI on Bitbucket Pipelines.

The original pipeline was a single QA step that did everything sequentially:

```
QA (single step, 2x instance, ~40 min):
├── yarn install
├── build ALL workspaces
├── validate scripts
├── lint ENTIRE codebase
├── unit tests
└── e2e tests
```

No parallelism, no conditional optimization. The build compiled frontends even when the changes were backend-only. Lint analyzed the entire codebase even for a one-line fix.

The sequential step breakdown:

| Phase | Time | Note |
|-------|------|------|
| yarn install | ~1m 30s | |
| Build | ~5m 20s | Frontend the bottleneck (5:19), workspaces in parallel via `foreach -Apv` |
| Lint | ~8m 40s | Frontend the slowest (8:40), entire codebase |
| Unit tests | ~7m 46s | 1446 backend tests (457s), 105 shared, 39 shared-backend, 4 ms-media |
| E2E tests | ~7m 24s | 211 tests (444s), sequential due to shared database |

## Phase 1: Pipeline parallelization

Before touching the test runner, the most obvious optimization was in the pipeline structure itself: splitting the monolithic step into independent parallel steps.

```
install-build (2x):                 parallel (all 1x):
├── yarn install                    ├── lint (changed files only)
├── optimized build                 ├── unit-tests
└── validate scripts                └── e2e-tests
    ↓ artifacts: dist/ ─────────────→
```

The first step compiles and passes artifacts to the subsequent steps via cache. Lint, unit tests, and E2E tests run in parallel on 1x instances (cheaper). Total time becomes the slowest step, not the sum of all of them.

Three conditional optimizations on top: lint only on files changed in the PR (from 8m40s down to a few seconds for small PRs, using `git diff` to filter and `xargs eslint` per workspace), skip backend tests if the changes are frontend-only, and conditional frontend build (just typecheck if files aren't touched).

Result: from 40 minutes to 25 minutes and 30 seconds. The bottleneck had shifted to the test step (~8 minutes, unchanged).

## Phase 2: From Jest to Bun

The bottleneck was Jest. Not the framework itself, but its execution model: one worker per file, SWC transpilation for every file, startup overhead repeated hundreds of times. Locally, unit tests ran in about 2 minutes with Jest. With Bun, 5 seconds.

That number was too interesting to ignore.

### The spike

Before touching anything in production, I did a spike on a separate branch. The goal was to figure out whether Bun could run the test suite without rewriting everything.

Short answer: yes, but not for free.

The fundamental problem is that Bun and Jest have opposite execution models. Jest isolates each test file in a separate process. Bun runs them all in a single process with a shared module cache. This difference changes everything about how mocks work.

| Aspect | Jest | Bun |
|--------|------|-----|
| Process | One worker per file (isolation) | Single process (shared cache) |
| Module cache | Reset between files | Shared across all files |
| `jest.mock()` | Hoisting, file-scoped | Leaks globally |
| Transpiler | SWC / ts-jest | Native (built-in) |
| ESM | CJS-first, simulates ESM | ESM-first, strict on named exports |
| Config | `jest.config.ts` | `bunfig.toml` |
| Setup | `setupFilesAfterEnv` | `[test].preload` |

The spike produced a concrete list of incompatibilities and a migration plan in 5 PRs.

### The strategy: 5 incremental PRs

I couldn't do a big bang on 222 files and hope for the best. But I also couldn't keep Jest and Bun running in parallel for too long: some transformations only work with one runner and break the other.

The solution was to structure the PRs so that the first 4 were backward-compatible with Jest (they could be merged and deployed normally), and only the fifth one would do the actual switch and cleanup.

#### PR 1: Removing jest-when

`jest-when` is a library that depends on Jest internals not available in Bun. Instead of writing a custom replacement, I replaced everything with Jest's native APIs.

The pattern `when(spy).calledWith(args).mockResolvedValue(result)` did two things at once: configured the mock and implicitly verified the arguments. Replacing it meant separating those two responsibilities:

```typescript
// Before (jest-when)
when(service.findById).calledWith(userId).mockResolvedValue(user);

// After (native)
service.findById.mockImplementation((id) => {
  if (id === userId) return Promise.resolve(user);
  throw new Error(`Unexpected argument: ${id}`);
});
```

And adding explicit assertions where needed:

```typescript
expect(service.findById).toHaveBeenCalledWith(userId);
```

21 files touched. Also added an ESLint rule to ban future imports of jest-when.

Compatible with Jest. Merged and released without issues.

#### PR 2: Fixing expect() syntax

Bun implements almost all of Jest's `expect` API, but there are divergences in the edge cases. The spike had identified three, spread across about 13 files. Few compared to the 400+ total files, which confirms that Bun's compatibility is solid for the vast majority of cases. But these three had to be fixed before the switch.

**1. Arrow functions with rejects.** Jest accepts both `expect(() => asyncFn()).rejects` and `expect(asyncFn()).rejects`. Bun only wants the second form: it expects a Promise, not a function that returns a Promise.

```typescript
expect(() => asyncFn()).rejects.toThrow();  // Jest ok, Bun no
expect(asyncFn()).rejects.toThrow();         // both ok
```

**2. toStrictEqual on class instances.** Bun is stricter with deep comparison. If you compare a domain object (class instance) with an object literal that has the same properties, `toStrictEqual` fails because the prototypes differ. Jest treated them as equivalent.

```typescript
const result = new UserDo({ name: 'Alice', role: 'admin' });
const expected = { name: 'Alice', role: 'admin' };

expect(result).toStrictEqual(expected);  // Jest ok, Bun fails
expect(result).toEqual(expected);         // both ok
```

In the codebase this pattern was common: tests compared domain objects returned by use cases with literals. The fix was switching to `toEqual`, which verifies structural equality without checking prototypes.

**3. ReadableStream.from().** Not implemented in Bun. Tests that verified file downloads used it to create test streams. Replaced with `new Response(data).body`, which works on both runtimes.

Compatible with Jest. Merged without issues.

#### PR 3: Type imports for strict ESM

Bun is ESM-first and strict on named exports. If you import a name that only exists as a TypeScript type (not in the compiled JS), Bun crashes at runtime:

```
SyntaxError: Export named 'RawBodyRequest' not found in module '@nestjs/common/index.js'
```

The fix is adding the `type` keyword to type-only imports:

```typescript
// Before
import { Injectable, RawBodyRequest } from '@nestjs/common';

// After
import { Injectable, type RawBodyRequest } from '@nestjs/common';
```

I applied the correction across the entire backend with the help of the ESLint rule `@typescript-eslint/consistent-type-imports` with `fixStyle: 'inline-type-imports'`, which also has an autofix. This matters: without the ESLint rule, every new file could reintroduce the problem.

**The limit of the ESLint rule: NestJS dependency injection.**

The `consistent-type-imports` rule has a blind spot with NestJS. In a constructor with dependency injection, the class token is used at runtime by the container to resolve the dependency. The import isn't type-only, it's a value needed at runtime:

```typescript
import { PrismaService } from '@/prisma/prisma.service';

@Injectable()
export class UserRepository {
  // PrismaService is needed at runtime for DI, the import without `type` is correct
  constructor(private readonly prisma: PrismaService) {}
}
```

Here the ESLint rule correctly doesn't flag anything. The problem is the opposite case: when an imported type is only used as a type annotation in a parameter that looks like DI but isn't (for example a generic type, an interface, or a non-injected parameter). The rule doesn't have enough context to distinguish these cases from legitimate ones, so it doesn't trigger where it should. These imports need to be verified manually during review.

Compatible with Jest. Merged without issues.

#### PR 4: jest.mock() → jest.spyOn()

This is the most important transformation and the one that gave me the most trouble during the spike.

**The problem.** `jest.mock()` replaces an entire module in the process cache. With Jest this isn't an issue: each test file runs in a separate worker, so the mock is automatically file-scoped. When the worker terminates, the mock disappears with it.

With Bun it's different. All test files run in the same process and share the same module cache. When a file calls `jest.mock()`, the original module gets replaced in the global cache. Test files that run after it will see the mock instead of the real module. There's no way to restore it: `jest.restoreAllMocks()` restores spies, not modules replaced with `jest.mock()`.

In practice, the execution order of test files becomes significant. A test that passes on its own might fail when it runs after another file that mocked the same module. The most insidious kind of bug: intermittent and order-dependent.

**The solution.** Switch from `jest.mock()` (module replacement) to `jest.spyOn()` on namespace imports (single function replacement). The spy is reversible with `restoreAllMocks()`:

```typescript
// Before (Jest-only, irreversible global mock)
jest.mock('@/notifications/sendNotification', () => ({
  sendNotification: jest.fn(),
}));
import { sendNotification } from '@/notifications/sendNotification';

// After (compatible with both, reversible)
import * as NotificationModule from '@/notifications/sendNotification';

beforeAll(() => {
  jest.spyOn(NotificationModule, 'sendNotification')
    .mockImplementation(jest.fn() as never);
});

afterAll(() => {
  jest.restoreAllMocks();
});
```

The `afterAll` with `restoreAllMocks` is the critical piece. It restores the original function in the namespace, so the next test file sees the real module. Without it, the spy survives and contaminates everything that runs after.

**An important distinction: `clearAllMocks` vs `resetAllMocks` vs `restoreAllMocks`.** In the codebase I use `@automock/jest` for the NestJS TestBed. `resetAllMocks()` also resets implementations configured by automock, breaking the TestBed mocks. `clearAllMocks()` instead only resets the call history (how many times it was called, with what arguments) but preserves implementations. So the correct pattern is:

- `afterEach`: `jest.clearAllMocks()` (clean call history between tests)
- `afterAll`: `jest.restoreAllMocks()` (restore original modules between files)

17 files migrated. I also added two ESLint rules `no-restricted-syntax` in test files to prevent regressions:

```javascript
'no-restricted-syntax': ['error',
  {
    selector: "CallExpression[callee.object.name='jest'][callee.property.name='mock']",
    message: 'jest.mock() leaks globally in bun (single process). '
           + 'Use jest.spyOn() on a namespace import.'
  },
  {
    selector: "CallExpression[callee.object.name='jest'][callee.property.name='resetAllMocks']",
    message: 'jest.resetAllMocks() breaks @automock/jest TestBed mocks. '
           + 'Use jest.clearAllMocks() to clear call history.'
  }
]
```

**A bug discovered during this phase.** One test file used `jest.mock('../constants')` to override a constant (a country blacklist for a service). With Jest, the mock was isolated in the file's VM. During tests with Bun on the spike, the mocked value leaked to other test files that imported the same constant, causing intermittent and non-reproducible failures: tests passed or failed depending on execution order. With the conversion to `jest.spyOn()` the problem disappeared, because the spy gets restored by `restoreAllMocks()` in `afterAll`. This kind of latent bug is an interesting argument in favor of the migration: Bun's single-process model makes isolation problems explicit that Jest was hiding.

Compatible with Jest. Merged without issues.

#### PR 5: Runner switch, cleanup, and the last mocks

This one couldn't be incremental. There were 10 tests that used `jest.mock()` on namespace imports where SWC compiled the properties as non-configurable. With Jest + SWC, `jest.spyOn()` on these properties failed silently: the spy was created but didn't intercept calls. With Bun, `spyOn` on namespace imports works correctly because there's no SWC in the way. So these 10 mocks had to be migrated at the same time as the runner switch: with Jest they would have been broken, with Bun they worked. No way to make them backward-compatible.

I also included the Jest cleanup in the same PR for convenience, since the switch made all the old tooling useless.

**Bun configuration.** Two `bunfig.toml` files, one for unit tests and one for E2E, with different preloads:

```toml
# bunfig.toml (unit tests)
[test]
preload = ["./apps/backend/bun.test.setup.ts"]

# bunfig.e2e.toml (E2E tests)
[test]
preload = ["./apps/backend/bun.test.setup.e2e.ts"]
```

**The bcrypt problem.** `bcrypt` is a module with native C++ bindings that won't load in Bun. It needs to be mocked in the preload, but with different strategies for unit and E2E. In the unit preload, a stub that always returns fake hashes and `true` for `compare()` is enough: unit tests don't verify actual hashing. In the E2E preload you need a working implementation because login flows do real hashing, so I used `bcryptjs` (a pure-JS reimplementation of bcrypt):

```typescript
// unit preload
mock.module('bcrypt', () => ({
  hash: jest.fn().mockResolvedValue('$2b$10$mockhash'),
  compare: jest.fn().mockResolvedValue(true),
  genSalt: jest.fn().mockResolvedValue('$2b$10$mocksalt'),
}));

// e2e preload
mock.module('bcrypt', () => require('bcryptjs'));
```

**The @Transactional problem.** The global setup solves a specific NestJS + Prisma issue. The `@Transactional()` decorator instantiates a `PrismaClient` at class definition time, not at execution time. With Jest each file has its own process, so the crash from a missing database connection is isolated and masked by mocks. With Bun, single process: the first class decorated with `@Transactional` that gets imported (even indirectly) tries to connect to the database and fails.

The solution is to mock the decorator as a no-op in the preload, before any test runs:

```typescript
import { mock } from 'bun:test';

mock.module('./src/common/transaction.decorator', () => ({
  Transactional:
    () =>
    (_target: object, _propertyName: string | symbol, descriptor: PropertyDescriptor): PropertyDescriptor =>
      descriptor, // No-op: returns the descriptor unchanged
}));
```

This makes `@Transactional()` a decorator that does nothing: it receives the method descriptor and returns it as-is, without wrapping the method in a Prisma transaction.

**E2E setup.** The E2E preload has one addition: it suppresses console output. The application logger (Winston) writes to `process.stdout` directly, not through `console.log`, so app logs during E2E tests flood the output. The setup mocks `console.log`, `console.error`, `console.warn`, `console.info`, and `console.debug` with `jest.fn()`.

**Global mock cleanup.** The setup also registers cleanup hooks:

```typescript
afterEach(() => {
  jest.clearAllMocks();  // Clean call history between tests
});

afterAll(() => {
  jest.restoreAllMocks(); // Restore original modules between files
});
```

**Updated npm scripts.**

```json
{
  "test": "bun test $(find src -name '*.spec.ts' ! -name '*.e2e.spec.ts')",
  "test:e2e": "bun test -c bunfig.e2e.toml --max-concurrency 1 $(find src -name '*.e2e.spec.ts')"
}
```

E2E tests run with `--max-concurrency 1` because they share the database.

**CI: installing Bun in the Docker container.**

```yaml
- curl -fsSL https://bun.sh/install | BUN_INSTALL=/usr/local bash -s "bun-v1.3.11"
```

Pinned to 1.3.11. Version 1.3.10 has a regression with `reflect-metadata` that causes hundreds of failures in NestJS decorators (`IdentifierNotFoundError`, `ReferenceError: Cannot access 'X' before initialization`). This needs to be verified before every runtime update.

**Jest cleanup.** In the same PR I removed everything that was Jest-only: configuration files (`jest.config.ts`, `jest.config.e2e.ts`, `.swcrc`, `jest.globalSetup.ts`, `jest.setupAfterEnv.e2e.ts`) and devDependencies (`@swc/core`, `@swc/jest`, `jest-when`, `ts-jest`).

## The numbers

| Metric | Original | Post parallelization | Post Bun |
|--------|----------|---------------------|----------|
| CI pipeline | 40 min | 25 min 26s | 16 min 13s |
| Unit tests (local) | ~2 min | ~2 min | ~5s |
| E2E tests (local) | ~10 min | ~10 min | ~5 min |

| Metric | Value |
|--------|-------|
| DevDependencies removed | 4 (`@swc/core`, `@swc/jest`, `jest-when`, `ts-jest`) |
| Config files removed | 5 |
| Test files modified | 222 |

## What I learned

**The single process changes everything.** Most of the migration work wasn't about Bun itself, but about the fact that tests are no longer isolated. With Jest, every test file is a sandbox: you can call `jest.mock()`, mutate global state, and everything disappears when the worker terminates. With Bun, those design choices surface. Mocks that leak, imports that crash at runtime, decorators that instantiate database clients at class-load. None of these were new bugs. They were technical debt masked by Jest's isolation. In a way, the migration improved test quality: it made tests more explicit about what they mock and more disciplined about cleaning up after themselves.

**The spike was the most important step.** I dedicated an entire branch to running the full test suite with Bun, without worrying about code cleanliness. The goal was just to catalog the incompatibilities. That branch became the de facto specification for the 5 subsequent PRs. Without the spike, I would have discovered problems one at a time during the actual migration, with the team blocked at every surprise. With the spike, I already knew exactly how many files I needed to touch, which patterns were problematic, and where the points of no return would be.

**Incremental PRs worked, but they require discipline.** The first 4 PRs were compatible with Jest and could be merged independently. The team continued working normally, merging features and fixes while I proceeded with the migration. The big bang of PR 5 was contained because most of the work was already in develop. The risk with this approach is drift: while you're working on PRs 3 and 4, someone might introduce new `jest.mock()` calls or imports without `type` in develop. The ESLint rules I added in the first PRs also protected against this: anyone writing new code was blocked by the linter before the migration was even complete.

**ESLint rules are the real long-term protection.** A migration without guardrails is an invitation to regress. After a few months, nobody remembers why `jest.mock()` is banned or why `type` is needed in imports. A code comment gets ignored. A wiki document gets forgotten. An ESLint rule that blocks the commit with a message like "jest.mock() leaks globally in bun (single process). Use jest.spyOn() on a namespace import." is impossible to ignore. I added rules for every problematic pattern discovered during the spike: ban on `jest.mock()`, ban on `jest.resetAllMocks()`, enforcement of `consistent-type-imports`, ban on imports from `jest-when`. Each one with a message that explains the why, not just the what.

**Version pinning is not optional.** Bun 1.3.10 introduced a regression in `reflect-metadata` handling that caused 309 failures: `IdentifierNotFoundError` and `ReferenceError: Cannot access 'X' before initialization` in NestJS decorators. I pinned to 1.3.11 in the CI pipeline and in the development Dockerfile. Before every update, the rule is: run the entire test suite with the new version locally, verify that the number of passing tests is the same. Don't trust the semver of a runtime that moves fast.

**The real cost of the migration isn't technical.** Changing 222 test files is mechanical work. The real cost is in communication: explaining to the team why they can't use `jest.mock()` anymore, why lint has new rules that weren't there yesterday, why their test that used to pass now fails in CI. The migration guide I wrote during the spike (an internal markdown with before/after examples for every pattern) was probably the most useful deliverable of the entire project. Not for me, but for whoever would write tests after me.

**This is not evangelism.** Bun is not perfect. The single-process model is a real tradeoff. E2E tests have to run with `--max-concurrency 1` because they share the database. Some Node.js APIs aren't implemented (`ReadableStream.from()`). The ecosystem of plugins and integrations is smaller. But for the specific use case of "running tests in a NestJS backend", the speed gain was significant enough that the tradeoffs were acceptable. The CI pipeline went from 40 to 16 minutes. The local unit test feedback loop from 2 minutes to 5 seconds. That changes the way you work: you run tests more often, you run them before pushing, you run them while developing. It's not just a lower number on a dashboard.

*The current bottleneck is E2E tests: they run with `--max-concurrency 1` because they share a single PostgreSQL instance. About 5 minutes both in CI and locally, the slowest step in the pipeline. Parallelizing them requires eliminating database contention. Options range from template databases (`CREATE DATABASE ... TEMPLATE`) to schema isolation per test file to transactions with rollback. The most interesting long-term solution is PGlite: a PostgreSQL compiled to WebAssembly that runs in-process, with instant setup and one instance per test file. Unlike in-memory SQLite, it's a real PostgreSQL, so no risk of query divergence.*
