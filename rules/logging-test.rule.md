---
description: Logger mocking and assertion patterns for tests
globs: "src/**/*.test.ts, src/**/*.spec.ts"
alwaysApply: false
---

# Logging in Tests

## Overview

| Approach | When to use |
|----------|-------------|
| Spy transport (`createSpyLogger`) | Testing code that produces log output — assert on captured `LogEntry` objects |
| Mock logger (`createMockLog`) | DI into code under test — logger is a dependency, not the subject |

## Spy Transport Pattern

Use when testing that code logs the right level, message, or context. Creates a real logger with a capturing transport:

```typescript
import type { LogEntry, LogTransport } from "@pencroff-lab/kore";
import { createLogger } from "@pencroff-lab/kore";

function createSpyLogger(module?: string) {
  const entries: LogEntry[] = [];
  const spyTransport: LogTransport = { write(e) { entries.push(e); } };
  const logger = createLogger(module, {
    transports: [spyTransport],
    level: "trace",
  });
  return { logger, entries };
}

// Usage
const { logger, entries } = createSpyLogger('test');
logger(logger.INFO, 'Starting');

expect(entries).toHaveLength(1);
expect(entries[0]?.level).toBe("info");
expect(entries[0]?.message).toBe("Starting");
```

Set `level: "trace"` so all log calls are captured regardless of level.

## Mock Logger Pattern

Use when logger is just a dependency passed via DI. Creates a Sinon stub satisfying the `Logger` type:

```typescript
import sinon from "sinon";
import type { Logger } from "@pencroff-lab/kore";

function createMockLog(): Logger {
  const log = sinon.stub() as unknown as Logger;
  Object.assign(log, {
    TRACE: "trace", DEBUG: "debug", INFO: "info",
    WARN: "warn", ERROR: "error", FATAL: "fatal",
    child: sinon.stub().returns(log),
  });
  return log;
}

// Usage
const mockLog = createMockLog();
const deps = { log: mockLog, db: mockDb, config: testConfig };

const handler = createHandler(deps);
await handler({ plan: "./test.md" });

sinon.assert.calledWith(
  mockLog as unknown as sinon.SinonStub,
  mockLog.INFO,
  sinon.match("Starting")
);
```

## Naming

Use `mockLog` for mock/stub loggers in tests:

```typescript
const mockLog = createMockLog();
```

## Summary

1. **Spy transport** — real logger, captured entries, assert on `LogEntry` fields
2. **Mock logger** — Sinon stub, DI into deps, assert on call args
3. **Always set `level: "trace"`** in spy transport to capture all levels
4. **`child()` returns self** in mock — keeps assertions simple
5. **Name test loggers `mockLog`** — consistent with project naming convention