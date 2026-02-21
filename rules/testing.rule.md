---
description: Testing patterns, strategies, and mocking conventions for writing tests
globs: "*.test.ts"
alwaysApply: false
---

# Testing Guide

This guide covers testing patterns for internal functions without polluting the public API. The project uses `bun test` as the test framework with Sinon for mocking.

## Overview

When testing modules, avoid exporting internal functions, state, or reset helpers solely for testing purposes. Instead, use the following strategies in order of preference:

1. **Strategy 1: Observable Behavior Testing** (Primary)
2. **Strategy 2: Dependency Injection** (Primary)
3. **Strategy 3: Underscore-Prefixed Properties** (Edge cases only)

## Using `bun test` api

Prefered to write tests using `test` like:

```typescript
import { test } from 'bun:test';

test('creates Err', () => {
  // test code here
});
```

PROHIBITED to use `it` like:

```typescript
import { it } from 'bun:test';
it('should creates Err', () => {

})
```

## Choosing a Strategy

Before writing tests, answer these questions in order:

1. **Can I verify this through an observable effect?** (file write, console output, HTTP call, return value)
   - If yes -> Use Strategy 1

2. **Can I inject the dependency as a parameter?**
   - If yes -> Use Strategy 2

3. **Is the internal function pure (no side effects)?**
   - If yes -> Extract to a separate, exported utility module

4. **Would testing this internal detail break if implementation changes?**
   - If yes -> Test through public API instead

5. **Does the test require resetting module-level state between tests?**
   - If yes -> Refactor to use dependency injection

Only after exhausting these options should you consider Strategy 3.

## Core Rules

### Rule: Test Files Live Next to Source Files

Place test files in the same directory as the code they test:

```
src/
  utils/
    parser.ts
    parser.test.ts      # Next to source
  services/
    auth.ts
    auth.test.ts        # Next to source
```

**Not** in a separate test directory:

```
src/
  utils/
    parser.ts
tests/                  # Avoid separate test folders
  utils/
    parser.test.ts
```

### Rule: Mocking Separation (Bun + Sinon)

**Strict rule:** Bun `mock.module()` is used ONLY for ESM import wiring. Sinon handles ALL behavior definitions and assertions.

| Use Case | Tool | Rationale |
|----------|------|-----------|
| ESM import wiring | `bun:test` `mock.module()` | Sinon cannot mock ESM imports |
| Function stubs | Sinon `stub()` | Rich assertions, call tracking |
| Method spies | Sinon `spy()` | Call tracking without replacing |
| Timers / Date | Sinon `useFakeTimers()` | Unified tick control |
| Call order | Sinon `assert.callOrder()` | Built-in ordering assertions |
| Value assertions | `expect()` from `bun:test` | Standard test framework |

**Pattern:** Sinon stubs define behavior, Bun wires them into ESM imports:

```typescript
import { mock } from "bun:test";
import sinon from "sinon";

// 1. Sinon stubs define ALL behavior
const loadConfigStub = sinon.stub().returns(Outcome.ok(testConfig));
const parseMarkdownStub = sinon.stub().returns(Outcome.ok(sections));

// 2. Bun wires ESM imports to Sinon stubs
mock.module("../../src/config/loader", () => ({
  loadConfig: loadConfigStub,
}));

mock.module("../../src/services/markdown/parser", () => ({
  parseMarkdown: parseMarkdownStub,
}));

// 3. All mock assertions via Sinon
sinon.assert.calledOnce(loadConfigStub);
sinon.assert.calledWith(parseMarkdownStub, sinon.match.string);
sinon.assert.callOrder(loadConfigStub, parseMarkdownStub);
```

### Rule: Always Use Sinon Sandboxes

Use Sinon sandboxes for automatic cleanup of stubs, spies, and timers. Declare the sandbox in `beforeEach` and restore in `afterEach`.

```typescript
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import sinon from "sinon";

describe("feature", () => {
  let sandbox: sinon.SinonSandbox;

  beforeEach(() => {
    sandbox = sinon.createSandbox();
  });

  afterEach(() => {
    sandbox.restore();
  });

  test("default behavior", () => {
    const stub = sandbox.stub(someModule, "method").returns("default");

    expect(someModule.method()).toBe("default");
    sinon.assert.calledOnce(stub);
  });

  test("edge case behavior", () => {
    sandbox.stub(someModule, "method").returns("edge-case");

    expect(someModule.method()).toBe("edge-case");
  });
});
```

This pattern:
- Automatically restores all stubs/spies/timers in `afterEach`
- Prevents mock leakage between tests
- No manual `mockRestore()` calls needed

---

## Strategy 1: Observable Behavior Testing

Test internal behavior by verifying side effects through mocked dependencies. This approach tests what the code *does*, not how it's structured internally.

### When to Use

- Testing cleanup functions (verify `fs.rmSync` was called)
- Testing event handlers (capture and trigger via mocked `process.on`)
- Testing logging behavior (verify logger was called)
- Any case where internal state produces observable external effects

### Pattern

```typescript
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import sinon from "sinon";
import * as fs from "node:fs";

describe("cleanup behavior", () => {
  let sandbox: sinon.SinonSandbox;
  let handlers: Record<string, (...args: never[]) => unknown>;

  beforeEach(() => {
    sandbox = sinon.createSandbox();
    handlers = {};
    sandbox.stub(process, "on").callsFake(
      (event: string, handler: (...args: never[]) => unknown) => {
        handlers[event] = handler;
        return process;
      }
    );
    sandbox.stub(fs, "rmSync");
  });

  afterEach(() => {
    sandbox.restore();
  });

  test("removes temporary files on process exit", () => {
    const result = createExtensionLoader("test", "/path/to/file.tar.gz");

    // Trigger the captured handler
    handlers["exit"]?.();

    // Verify the observable effect via Sinon
    sinon.assert.called(fs.rmSync as sinon.SinonStub);
  });

  test("handles missing file gracefully", () => {
    (fs.rmSync as sinon.SinonStub).throws(new Error("ENOENT: no such file"));

    const result = createExtensionLoader("test", "/path/to/missing.tar.gz");

    expect(() => handlers["exit"]?.()).not.toThrow();
  });
});
```

### ESM Module Wiring with `mock.module()`

Use `mock.module()` **exclusively** for replacing ESM imports. Behavior is always defined by Sinon stubs wired in.

**Important:** Module mocks persist across tests, so structure your test file carefully.

```typescript
import { afterAll, beforeAll, describe, expect, mock, test } from "bun:test";
import sinon from "sinon";

describe("extension loader with mocked fs", () => {
  // 1. Create Sinon stubs for behavior
  const existsSyncStub = sinon.stub().returns(false);
  const rmSyncStub = sinon.stub();

  beforeAll(() => {
    // 2. Wire stubs into ESM module
    mock.module("node:fs", () => ({
      existsSync: existsSyncStub,
      rmSync: rmSyncStub,
    }));
  });

  afterAll(() => {
    mock.restore();
    existsSyncStub.reset();
    rmSyncStub.reset();
  });

  test("handles missing file gracefully", async () => {
    const { createExtensionLoader } = await import("./load_extensions");

    createExtensionLoader("test", "./missing.tar.gz");

    // Assertions via Sinon
    sinon.assert.calledWith(existsSyncStub, "./missing.tar.gz");
  });

  test("skips cleanup when file does not exist", async () => {
    const { createExtensionLoader } = await import("./load_extensions");

    createExtensionLoader("test", "./missing.tar.gz");

    sinon.assert.notCalled(rmSyncStub);
  });
});
```

### When to Use Sinon vs `mock.module()`

| Use Sinon when... | Use `mock.module()` when... |
|-------------------|----------------------------|
| Defining stub/spy behavior | Replacing ESM import bindings |
| Asserting call counts and args | Module loaded at import time |
| Controlling timers and dates | Mocking built-in modules (fs, path) |
| Tracking call order | Wiring Sinon stubs into ESM graph |

**PROHIBITED:** Using `mock.module()` with `bun:test` `mock()` functions for behavior. Always wire Sinon stubs into `mock.module()`.

### Key Points

- Mock at the dependency boundary, not inside the module under test
- Verify effects (function calls, state changes) rather than internal structure
- Sinon stubs define behavior, Bun `mock.module()` wires ESM imports
- Use `beforeAll`/`afterAll` for `mock.module()`, Sinon sandbox in `beforeEach`/`afterEach`
- Call `mock.restore()` in `afterAll` to clean up module mocks

---

## Strategy 2: Dependency Injection

Pass dependencies as optional parameters with sensible defaults. This makes internal state testable without exposing it globally.

### When to Use

- Testing code that maintains internal state (registries, caches, counters)
- Testing code that depends on singletons or global state
- When you need to isolate tests from each other completely
- When mocking would require too many layers of indirection

### Pattern

```typescript
// Production code
const _defaultRegistry = new Set<() => void>();

export function createExtensionLoader(
  name: string,
  tarPath: string,
  options?: {
    registry?: Set<() => void>;
  }
): Outcome<Extension> {
  const registry = options?.registry ?? _defaultRegistry;

  const cleanup = () => {
    // Perform cleanup work
    if (existsSync(tarPath)) {
      rmSync(tarPath, { recursive: true });
    }
    // Remove self from registry
    registry.delete(cleanup);
  };

  registry.add(cleanup);

  // ... rest of implementation
  return Outcome.ok({ name, cleanup });
}
```

```typescript
// Test code
import { describe, expect, test } from "bun:test";

describe("createExtensionLoader", () => {
  test("registers cleanup handler", () => {
    // Create isolated test state
    const testRegistry = new Set<() => void>();

    // Inject test dependency
    const result = createExtensionLoader("test", "path.tar.gz", {
      registry: testRegistry,
    });

    // Verify internal behavior through injected dependency
    expect(testRegistry.size).toBe(1);
  });

  test("cleanup removes itself from registry when called", () => {
    const testRegistry = new Set<() => void>();

    createExtensionLoader("test", "path.tar.gz", {
      registry: testRegistry,
    });

    // Get and execute the cleanup
    const [cleanup] = testRegistry;
    cleanup();

    // Verify cleanup removed itself
    expect(testRegistry.size).toBe(0);
  });

  test("multiple loaders register independent cleanups", () => {
    const testRegistry = new Set<() => void>();

    createExtensionLoader("ext1", "path1.tar.gz", { registry: testRegistry });
    createExtensionLoader("ext2", "path2.tar.gz", { registry: testRegistry });

    expect(testRegistry.size).toBe(2);

    // Clean up first one
    const [firstCleanup] = testRegistry;
    firstCleanup();

    expect(testRegistry.size).toBe(1);
  });
});
```

### Benefits

- Tests are isolated - no shared global state
- No need for `_reset*` functions
- Clear contract between production and test code
- Default behavior unchanged for production use

### Guidelines

- Use an `options` object for optional parameters (extensible)
- Always provide sensible defaults for production use
- Name the internal default with underscore prefix: `_defaultRegistry`
- Document that the injection point exists for testing

---

## Strategy 3: Underscore-Prefixed Properties (Edge Cases)

Expose internal functionality via underscore-prefixed properties on returned objects. This signals "internal use" while keeping the main export clean.

### When to Use

**Only use this strategy when ALL of the following are true:**

1. **The behavior is inherently unobservable** — There's no external effect (file, network, console, return value) that can verify correctness. The logic is purely internal state manipulation with no boundary crossing.

2. **The dependency is not a natural injection point** — Injecting it would require passing implementation details through multiple layers that have no business knowing about them, or would distort the public API's purpose.

3. **The internal detail is architecturally stable** — The behavior you're exposing is unlikely to change during refactoring. It represents a stable invariant, not an implementation quirk.

4. **You've exhausted composition alternatives** — You cannot restructure the code into smaller, independently testable units without artificial fragmentation.

### Red Flags

If any of these apply, reconsider using Strategy 3:

- You're exposing `_internal` because mocking feels tedious (tedium isn't complexity)
- The internal function could reasonably be a separate utility
- You're testing *how* something works rather than *what* it guarantees
- The underscore property would need to change if you refactor internals

### Pattern

```typescript
// Production code - expose via underscore property
export function createExtensionLoader(
  name: string,
  tarPath: string,
): Outcome<Extension> {
  const cleanup = () => {
    // internal cleanup logic
  };

  return Outcome.ok({
    name,
    setup,
    _cleanup: cleanup,  // Underscore signals "internal"
  });
}
```

```typescript
// Test code - define a test-only type for type safety
type TestableExtension = Extension & {
  _cleanup: () => void;
};

test("cleanup handles missing file gracefully", () => {
  const result = createExtensionLoader("test", "path.tar.gz");
  const extension = result.value as TestableExtension;

  // Access internal via underscore property with type safety
  expect(() => extension._cleanup()).not.toThrow();
});
```

### Rules

- Use underscore prefix: `_cleanup`, `_internal`, `_debug`
- Document with `@internal` JSDoc tag
- Never use in production code
- Define test-only types that extend the public type (avoid `any`)
- Prefer this over exporting from module scope

### What NOT to Do

```typescript
// BAD: Exporting internal state at module level
export const cleanupRegistry = new Set<() => void>();
export let handlersRegistered = false;
export function _resetCleanupRegistry(): void { /* ... */ }

// BAD: Tests importing internal state
import { cleanupRegistry, _resetCleanupRegistry } from "./module";
```

---

## Async Testing Patterns

### Testing Async Functions

Use `async/await` in tests with Sinon stubs for async dependencies:

```typescript
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import sinon from "sinon";

describe("async operations", () => {
  let sandbox: sinon.SinonSandbox;

  beforeEach(() => {
    sandbox = sinon.createSandbox();
    sandbox.stub(global, "fetch").resolves(
      new Response(JSON.stringify({ data: "test" }))
    );
  });

  afterEach(() => {
    sandbox.restore();
  });

  test("fetches and processes data", async () => {
    const result = await fetchUserData("user-123");

    sinon.assert.calledWith(
      global.fetch as sinon.SinonStub,
      "/api/users/user-123"
    );
    expect(result.data).toBe("test");
  });

  test("handles fetch errors", async () => {
    (global.fetch as sinon.SinonStub).rejects(new Error("Network error"));

    const result = await fetchUserData("user-123");

    expect(result.isErr()).toBe(true);
  });
});
```

### Testing with Outcome Type

When functions return `Outcome<T>`, test both success and error paths using tuple extraction:

```typescript
import { describe, expect, test } from "bun:test";
import { Err } from "./err";

describe("parseConfig", () => {
  test("returns Ok with valid config", async () => {
    const [config, err] = (await parseConfig("valid.json")).toTuple();

    expect(err).toBeNull();
    expect(config).toEqual({ port: 3000 });
  });

  test("returns Err with invalid JSON", async () => {
    const [_, err] = (await parseConfig("invalid.json")).toTuple();

    expect(Err.isErr(err)).toBe(true);
    expect(err?.message).toContain("Invalid JSON");
  });

  test("returns Err when file not found", async () => {
    const [_, err] = (await parseConfig("missing.json")).toTuple();

    expect(err).not.toBeNull();
    expect(err?.hasCode("FILE_NOT_FOUND")).toBe(true);
  });
});
```

### Testing Timers and Delays

Use Sinon fake timers for time-dependent code:

```typescript
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import sinon from "sinon";

describe("cache expiration", () => {
  let clock: sinon.SinonFakeTimers;

  beforeEach(() => {
    clock = sinon.useFakeTimers(new Date("2024-01-01T00:00:00Z"));
  });

  afterEach(() => {
    clock.restore();
  });

  test("cache expires after TTL", () => {
    const cache = createCache({ ttlMs: 60000 });
    cache.set("key", "value");

    // Advance time by 61 seconds
    clock.tick(61_000);

    expect(cache.get("key")).toBeUndefined();
  });
});
```

### Async Dependency Injection

For complex async chains, inject the async dependency using Sinon stubs:

```typescript
// Production code
export async function processQueue(
  options?: {
    fetchFn?: typeof fetch;
  }
): Promise<Outcome<ProcessResult>> {
  const fetchFn = options?.fetchFn ?? fetch;

  const response = await fetchFn("/api/queue");
  // ... process response
}

// Test code
test("processes queue items", async () => {
  const mockFetch = sinon.stub().resolves(
    new Response(JSON.stringify([{ id: 1 }, { id: 2 }]))
  );

  const result = await processQueue({ fetchFn: mockFetch });

  expect(result.isOk()).toBe(true);
  sinon.assert.calledOnce(mockFetch);
});
```

---

## Parametrized Tests

Use `test.each` with object format for readable test names and structured data:

```typescript
import { describe, expect, test } from "bun:test";
import sinon from "sinon";

describe("ClaudeService.generate", () => {
  test.each([
    { scenario: "CLI timeout", exit: 124, stderr: "timeout", code: "CLI_TIMEOUT" },
    { scenario: "CLI crash", exit: 1, stderr: "segfault", code: "CLI_ERROR" },
    { scenario: "empty output", exit: 0, stderr: "", code: "EMPTY_RESPONSE" },
  ])("returns $code on $scenario", async ({ exit, stderr, code }) => {
    const shell = createMockShell();
    shell.run.resolves({
      exitCode: exit,
      stdout: Buffer.from(""),
      stderr: Buffer.from(stderr),
    });

    const service = createClaudeService({ shell, logger: mockLogger });
    const [_, err] = (await service.generate("test prompt")).toTuple();

    expect(err).not.toBeNull();
    expect(err?.hasCode(code)).toBe(true);
    sinon.assert.calledOnce(shell.run);
  });
});
```

### Validation Schema Tests

Parametrized tests work well for validating multiple inputs against schemas:

```typescript
describe("ConfigSchema", () => {
  test.each([
    { field: "project.name", value: "", reason: "empty string" },
    { field: "llm.generator.tool", value: "openai", reason: "unsupported tool" },
    { field: "llm.generator.timeout_seconds", value: -1, reason: "negative timeout" },
  ])("rejects invalid $field ($reason)", ({ field, value }) => {
    const config = buildTestConfig({ [field]: value });
    const [_, err] = validateConfig(config).toTuple();

    expect(err).not.toBeNull();
  });
});
```

---

## Decision Matrix

| Scenario | Recommended Strategy |
|----------|---------------------|
| Testing side effects (file I/O, logging) | Strategy 1 (Sinon stubs) |
| Testing state management (registries, caches) | Strategy 2 (DI) |
| Testing event handlers | Strategy 1 (Capture handlers) |
| Testing cleanup functions | Strategy 1 or Strategy 2 |
| Testing async operations | Strategy 1 or Strategy 2 |
| Singleton behavior | Strategy 2 (Inject instance) |
| Pure transformations | Extract to separate utility module |
| ESM import replacement | `mock.module()` with Sinon stubs |

**Note:** "Extract to separate utility module" means creating a new exported module (e.g., `src/utils/validators.ts`) that can be tested independently — not exporting internal functions from the original module.

---

## Anti-Patterns to Avoid

### Exporting Reset Functions

```typescript
// BAD
export function _resetForTesting(): void {
  globalState = initialState;
}
```

**Why:** Pollutes public API, creates hidden test dependencies, tests become order-dependent.

**Fix:** Use dependency injection or isolate state per test.

### Exporting Internal State

```typescript
// BAD
export const internalCache = new Map();
```

**Why:** Exposes implementation details, allows external mutation, breaks encapsulation.

**Fix:** Inject cache as optional parameter with default.

### Testing Private Methods Directly

```typescript
// BAD - extracting private method just for testing
export function _validateInput(input: string): boolean { /* ... */ }
```

**Why:** Tests implementation, not behavior. Refactoring breaks tests.

**Fix:** Test through public API that uses the validation.

### Over-Mocking

```typescript
// BAD - mocking everything with bun mock()
mock.module("./module", () => ({
  functionA: mock(() => "a"),
  functionB: mock(() => "b"),
  functionC: mock(() => "c"),
}));
```

**Why:** Tests don't verify real behavior, just mock interactions.

**Fix:** Mock only external dependencies (fs, network, etc.), test real module code.

### Mixing Mock Tools for Behavior

```typescript
// BAD - using bun:test mock() for behavior alongside Sinon
import { mock, spyOn } from "bun:test";
import sinon from "sinon";

const behaviorStub = mock(() => "result");  // Should be sinon.stub()
const spy = spyOn(obj, "method");           // Should be sinon.spy()
```

**Why:** Inconsistent assertion APIs, harder to maintain, violates strict separation rule.

**Fix:** Use Sinon for all behavior and assertions, `mock.module()` only for ESM wiring.

---

## Bun Test Features Reference

### Useful Test Utilities

```typescript
import {
  test,
  describe,
  expect,
  beforeAll,
  afterAll,
  beforeEach,
  afterEach,
  mock,          // ONLY for mock.module() ESM wiring
} from "bun:test";
import sinon from "sinon";  // ALL behavior/assertions

// Skip a test
test.skip("not yet implemented", () => {});

// Focus on specific test (only runs this test)
test.only("debug this test", () => {});

// Skip entire describe block
describe.skip("disabled feature", () => {});

// Timeout for slow tests (default: 5000ms)
test("slow operation", async () => {
  // ...
}, 10000);

// Todo test (placeholder)
test.todo("implement later");

// Parametrized tests
test.each([
  { input: "a", expected: "A" },
  { input: "b", expected: "B" },
])("converts $input to $expected", ({ input, expected }) => {
  expect(input.toUpperCase()).toBe(expected);
});
```

### Running Tests

```bash
# Run all tests
bun test

# Run specific file
bun test src/utils/parser.test.ts

# Run tests matching pattern
bun test --test-name-pattern "handles errors"

# Watch mode
bun test --watch

# Coverage (minimum 83%)
bun run test:coverage
```

---

## Integration vs Unit Tests

### Unit Tests

Test individual functions or modules in isolation:

- Mock all external dependencies via Sinon
- Fast execution
- Co-located with source files: `parser.test.ts`

### Integration Tests

Test multiple modules working together:

- Minimal mocking (only external services)
- May use real filesystem, databases (in-memory SQLite)
- Name with `.integration.test.ts` suffix
- Place in same directory as the primary module being tested

```
src/
  services/
    user_service.ts
    user_service.test.ts              # Unit tests
    user_service.integration.test.ts  # Integration tests
```

### When to Write Integration Tests

- Critical user-facing workflows
- Database operations
- File system operations with multiple steps
- API endpoint handlers

---

## Best Practices Summary

1. **Prefer testing observable behavior** over internal state
2. **Use dependency injection** for stateful components
3. **Mock at boundaries** (I/O, network, external services)
4. **Sinon for behavior, Bun for ESM wiring** - strict separation
5. **Use Sinon sandboxes** for automatic mock cleanup
6. **Keep tests isolated** - no shared mutable state between tests
7. **Place test files next to source files** - not in separate test directories
8. **Document internal access points** with `@internal` JSDoc
9. **Use Strategy 3 sparingly** - only when other strategies truly don't fit
10. **Refactor if testing is hard** - difficult tests often signal design issues
11. **Test error paths** - especially with `Outcome` types, test both success and failure
12. **Use `test.each`** for parametrized tests with object format