---
description: Logging patterns, call signatures, error logging strategy, and naming conventions
globs: "**/logger*.ts, **/log_*.ts"
alwaysApply: false
---

# Logging Guide

This guide covers logging patterns, conventions, and integration for the project.

## Overview

| Aspect | Decision |
|--------|----------|
| Logger type | Callable function with level constants and `child()` |
| Injection | DI via deps object |
| Correlation | Child logger with `runId` binding |
| Dry-run | Child logger with `DRY-RUN` module prefix |
| Error logging | DEBUG at origin, ERROR at boundary only |
| Env var | `LOG_LEVEL` (default: `info`) |

## Call Signatures

The logger supports overloaded call signatures. Levels are bound as readonly properties on every `Logger` instance:

```typescript
// 1 arg: message at INFO level
log('Application started');

// 2 args: level + message
log(log.WARN, 'Connection slow');

// 2 args: message + context object
log('User created', { userId: '123' });

// 2 args: message + detail string (stored as { detail: "..." })
log('Config loaded', '/path/to/config.json');

// 2 args: message + Err (stored as { err: <Err> })
log('Fetch failed', err);

// 3 args: level + message + context/Err
log(log.ERROR, 'Command failed', err);
log(log.INFO, 'Processing', { index: 1 });
```

Child loggers inherit the same signatures:

```typescript
const svcLog = log.child('claude');
svcLog(svcLog.DEBUG, 'Calling CLI');
```

## Log Levels

| Level | Value | Use Case | Example |
|-------|-------|----------|---------|
| `log.TRACE` | `"trace"` | Function entry/exit, loop iterations | `Entering parseMarkdown` |
| `log.DEBUG` | `"debug"` | Variable values, intermediate results, error origin | `Parsed 5 sections` |
| `log.INFO` | `"info"` | Major workflow steps, user-visible events | `Generating note for section 3` |
| `log.WARN` | `"warn"` | Recoverable issues, retry attempts | `CLI timeout, retrying (1/3)` |
| `log.ERROR` | `"error"` | Failures at handler boundary | `Command failed` (logged once) |
| `log.FATAL` | `"fatal"` | Unrecoverable, process exits | `Database connection failed` |

Level hierarchy: TRACE < DEBUG < INFO < WARN < ERROR < FATAL. Messages below the configured level are silently dropped.

## Environment Configuration

```bash
# Log level (default: info, case-insensitive)
LOG_LEVEL=debug
```

Invalid values fall back to `info`.

## Logger DI Pattern

Logger is passed through deps. Only call `createLogger` at the entry point; everywhere else receive `Logger` type via deps:

```typescript
import type { Logger } from "@pencroff-lab/kore";

interface ServiceDeps {
  db: Database;
  log: Logger;
  config: Config;
}

function createHandler(deps: ServiceDeps) {
  const { log, db, config } = deps;

  return async (argv: Args): Promise<void> => {
    const runId = crypto.randomUUID();
    const runLog = log.child('handler', { runId });

    runLog(runLog.INFO, 'Starting', { plan: argv.plan });
    // ...
  };
}
```

## Child Loggers

`child(module, bindings?)` creates a new logger that:
- Accumulates module names into a chain (rendered as `[mod1] [mod2]`)
- Merges bindings into every log entry's context
- Inherits the parent's level filter and transports

```typescript
const log = createLogger('app');
const dbLog = log.child('db');
const queryLog = dbLog.child('query', { pool: 'primary' });

queryLog(queryLog.INFO, 'Executing');
// modules: ["app", "db", "query"], context: { pool: "primary" }
```

## Run Correlation

Use child loggers with `runId` binding so all log lines from the same run can be traced:

```typescript
const runLog = log.child('from_plan', { runId });

runLog(runLog.INFO, 'Starting note generation', { plan: argv.plan });
runLog(runLog.INFO, 'Processing section', { index: 1, title: "Gradient Descent" });
```

## Error Logging Strategy

**Principle:** Log DEBUG at origin, add context via `.wrap()` in intermediate layers, log ERROR once at boundary.

### Service Layer (Origin) - DEBUG Only

```typescript
async function callClaude(prompt: string, ctx: ExecutionContext): Promise<Outcome<string>> {
  return Outcome.fromAsync(async () => {
    const result = await $`claude -p ${prompt}`.nothrow().quiet();

    if (result.exitCode !== 0) {
      const err = Err.from("CLI execution failed", { code: "CLI_ERROR" })
        .withMetadata({ exitCode: result.exitCode });

      // DEBUG at origin - captures technical detail
      ctx.log(ctx.log.DEBUG, 'Claude CLI failed', err);
      throw err;
    }

    return [result.stdout.toString(), null];
  });
}
```

### Intermediate Layer - Wrap Error, No Logging

```typescript
async function generateNote(
  section: Section,
  ctx: ExecutionContext
): Promise<Outcome<GeneratedNote>> {
  const prompt = buildPrompt(section);

  const [output, cliErr] = (await callClaude(prompt, ctx)).toTuple();
  if (cliErr) {
    // Wrap with context, don't log (avoid duplication)
    return Outcome.err(cliErr.wrap(`Failed to generate note for "${section.title}"`));
  }

  const [note, parseErr] = parseNoteOutput(output).toTuple();
  if (parseErr) {
    ctx.log(ctx.log.DEBUG, 'Note parsing failed', parseErr);
    return Outcome.err(parseErr.wrap("Failed to parse generated note"));
  }

  return Outcome.ok(note);
}
```

### Handler Boundary - ERROR Once

```typescript
const handler = async (argv: Args): Promise<void> => {
  const { log } = ctx;

  log(log.INFO, 'Starting note generation', { plan: argv.plan });

  for (const section of sections) {
    log(log.INFO, 'Processing section', { index: section.index, title: section.title });

    const [note, err] = (await generateNote(section, ctx)).toTuple();
    if (err) {
      // ERROR at boundary - single log with full wrapped context
      log(log.ERROR, 'Section processing failed', err);
      continue;
    }

    log(log.INFO, 'Note generated', { path: note.path });
  }

  log(log.INFO, 'Generation complete');
};
```

## Dry-Run Logging Pattern

Use a child logger with `DRY-RUN` module prefix for simulated operations:

```typescript
async function saveNote(
  note: GeneratedNote,
  ctx: ExecutionContext
): Promise<Outcome<string>> {
  const path = resolvePath(note);

  if (ctx.dryRun && ctx.dryRunLog) {
    ctx.dryRunLog(ctx.dryRunLog.INFO, 'Would write note', { path, size: note.content.length });
    return Outcome.ok(path);
  }

  return Outcome.fromAsync(async () => {
    await Bun.write(path, note.content);
    ctx.log(ctx.log.INFO, 'Wrote note', { path });
    return [path, null];
  });
}
```

## Err Rendering in Output

When an `Err` instance is passed as context, the pretty transport:
1. Extracts `err` from the context object (does not include it in the JSON context)
2. Renders it on an indented line below the main log line via `Err.toString({ stack: 3, metadata: true })`
3. Other context keys still appear as JSON on the main line

```
14:30:05.123 ERR [app] [db] Query failed {"table":"users"}
  err: connection refused [DB_ERROR]
```

## Testing with Logger

Use a spy transport to capture `LogEntry` objects for assertion:

```typescript
import sinon from "sinon";
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

For DI in tests, create a stub logger that satisfies the `Logger` type:

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

## Execution Context

```typescript
import type { Logger } from "@pencroff-lab/kore";

interface ExecutionContext {
  runId: string;
  dryRun: boolean;
  log: Logger;               // Run-scoped logger with runId binding
  dryRunLog: Logger | null;  // Only set when dryRun=true
}
```

## Naming Convention

Logger is a function type — use short names: `log`, `ctx.log`, `svcLog`, `runLog`.

| Context | Name | Example |
|---------|------|---------|
| Root / deps field | `log` | `deps.log`, `const log = createLogger('app')` |
| Run-scoped child | `runLog` | `const runLog = log.child('handler', { runId })` |
| Service child | `svcLog` | `const svcLog = log.child('claude')` |
| Dry-run child | `dryRunLog` | `const dryRunLog = runLog.child('DRY-RUN')` |
| Execution context | `ctx.log` | `ctx.log(ctx.log.INFO, 'Processing')` |
| Test mock | `mockLog` | `const mockLog = createMockLog()` |

## Summary

1. **Inject `log` via deps** — never import a global singleton
2. **Use child loggers** for run correlation (`runId`) and module scoping
3. **DEBUG at origin, ERROR at boundary** — avoid duplicate error logs
4. **Wrap errors in intermediate layers** — add context without logging
5. **Dedicated dry-run logger** — prefix with `DRY-RUN` module for simulated ops
6. **FATAL for process exits** — config load, DB init, handler creation failures
7. **Levels on the function** — use `log.INFO`, `log.ERROR` etc.
8. **Short names** — `log`, `ctx.log`, `svcLog`, `runLog` (not `logger`)
9. **`LOG_LEVEL` env var** — controls minimum level, default `info`, case-insensitive
10. **Err auto-formatting** — pass `Err` as context, rendered on indented line below message
