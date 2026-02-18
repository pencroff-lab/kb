# TypeScript Bundled Application Template

Step-by-step template adapted from the TypeScript library publish template.
Goal: reuse this pattern for any new TypeScript application bundled with Bun.

---

## 1. Project Layout

```
my-app/
├── src/
│   ├── main.ts               # Application entry point
│   ├── core.ts               # Main application code
│   ├── core.test.ts          # Tests (co-located)
│   └── core.bench.ts         # Benchmarks (optional)
├── scripts/
│   └── verify-build.ts       # Post-build artifact validation
├── dist/                     # Build output (git-ignored)
├── tsconfig.json             # IDE / development config (noEmit)
├── biome.json                # Biome linter / formatter config
├── bunfig.toml               # Bun runtime config (test coverage, etc.)
├── package.json
├── .gitignore
└── .github/workflows/ci.yml  # CI pipeline
```

Key principle: `src/main.ts` is the single entry point. Bun's bundler traces the
dependency graph from there to produce one bundled output file.
Tests and benchmarks live alongside source but are excluded from the bundle.

---

## 2. package.json — Essential Fields

```jsonc
{
  "name": "<app-name>",                        // ← your application name
  "version": "0.1.0",
  "description": "<app description>",          // ← short app description
  "private": true,                             // prevent accidental publish
  "license": "Apache-2.0",
  "author": "<Author Name>",                   // ← your name or org
  "repository": {
    "type": "git",
    "url": "https://github.com/org/my-app"
  },

  // ── Module system ──
  "type": "module",                            // ESM by default

  // ── Scripts ──
  "scripts": {
    "start": "bun dist/app.js",
    "dev": "bun --watch src/main.ts",
    "test": "bun test",
    "test:coverage": "bun test --coverage",
    "lint:edit": "bunx biome check --fix src/",
    "lint:type": "tsc --noEmit",
    "prelint:ci": "bun run lint:type",
    "lint:ci": "bunx biome ci src/",
    "lint": "bun run lint:edit && bun run lint:type",
    "prebuild": "rm -rf dist",
    "build": "bun build src/main.ts --outfile dist/app.js --target bun --minify",
    "postbuild": "bun scripts/verify-build.ts"
  },

  // ── Dependencies ──
  "devDependencies": {
    "@biomejs/biome": "2.4.2",                 // exact version
    "@types/bun": "1.3.9",                     // match your bun version (bun --version)
    "@types/sinon": "21.0.0",                  // exact version
    "mitata": "1.0.34",                        // exact version — benchmarking
    "sinon": "21.0.1",                         // exact version — test stubs/spies
    "typescript": "5.9.3"                      // exact version
  }
}
```

### User-provided values

| Placeholder | What to fill in |
|---|---|
| `<app-name>` | Your application name (e.g. `my-service`) |
| `<app description>` | Short description of the application |
| `<Author Name>` | Your name or organisation |

### Field-by-field notes

| Field | Purpose |
|---|---|
| `"private": true` | Prevents accidental `npm publish` / `bun publish` |
| `"type": "module"` | Makes `.js` files ESM by default in Node |
| `"start"` | Runs the bundled application from `dist/` |
| `"dev"` | Runs `src/main.ts` directly with file watcher for development |
| `"build"` | Bundles the app into a single file using Bun's built-in bundler |
| `"lint:type"` | Runs `tsc --noEmit` to type-check without emitting files |
| `"prelint:ci"` | Bun lifecycle hook — runs `lint:type` automatically before `lint:ci` |
| `"lint"` | Combined local command: Biome auto-fix + TypeScript type check |
| `"postbuild"` | Validates the bundled output exists |

### Dependency version policy

- **`@types/bun`** — should match the bun version used in the project. Run `bun --version` and pick the corresponding `@types/bun` release to keep type definitions in sync.
- **All other devDependencies** — use exact versions (no `^` or `~`). This ensures deterministic installs and avoids surprise breakage from patch releases.
- **`typescript`** — listed as a devDependency (not peerDependency) since the app is not consumed by other packages.

### Dev dependency roles

| Package | Purpose |
|---|---|
| `@biomejs/biome` | Linter and formatter (replaces ESLint + Prettier) |
| `@types/bun` | TypeScript definitions for the Bun runtime |
| `@types/sinon` | TypeScript definitions for Sinon |
| `mitata` | Micro-benchmarking library for `*.bench.ts` files |
| `sinon` | Test stubs, spies, and mocks |
| `typescript` | TypeScript compiler (used for type-checking only) |

---

## 3. TypeScript Config — Single File

Unlike the library template (which needs 4 tsconfig files for dual ESM/CJS builds),
an application needs only one. Bun's bundler handles the build — tsc is used solely
for type-checking in the IDE and CI.

### `tsconfig.json`

```jsonc
{
  "compilerOptions": {
    "lib": ["ES2022"],
    "target": "ES2022",
    "module": "Preserve",
    "moduleDetection": "force",

    // Bundler mode (for IDE / Bun runtime)
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,

    // Strict checks
    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noPropertyAccessFromIndexSignature": false
  }
}
```

**Why only one tsconfig?** The application is bundled by `bun build`, not compiled
by tsc. There is no need for separate ESM/CJS output configs, build-specific
overrides, or declaration generation. tsc stays as a type-checker only.

---

## 4. Build — Bun Bundler

### How it works

```
src/main.ts ── bun build ──→ dist/app.js  (single bundled file)
```

Bun's built-in bundler traces all imports from the entry point and produces a
single self-contained output file. No additional bundler config is needed.

### Build command breakdown

```bash
bun build src/main.ts --outfile dist/app.js --target bun --minify
```

| Flag | Purpose |
|---|---|
| `src/main.ts` | Entry point — bundler traces dependency graph from here |
| `--outfile dist/app.js` | Single output file |
| `--target bun` | Optimizes for Bun runtime (marks Bun APIs as external) |
| `--minify` | Minifies the output for smaller file size |

### Alternative targets

| Target | Use when |
|---|---|
| `--target bun` | Deploying to a Bun runtime environment |
| `--target node` | Deploying to a Node.js environment |
| `--target browser` | Building for the browser |

### Optional flags

| Flag | Purpose |
|---|---|
| `--sourcemap=linked` | Generates a separate `.js.map` file for debugging |
| `--splitting` | Enables code splitting (produces multiple chunks, requires `--outdir` instead of `--outfile`) |

### Development mode

`bun --watch src/main.ts` runs the app directly from source with automatic
restart on file changes. No build step needed during development.

---

## 5. `.gitignore` — Essential Entries

```gitignore
# Build output
dist/

# Dependencies
node_modules/

# Environment
.env
.env.*
**/.env
**/.env.*
!.env.example

# Temporary
tmp/

# IDE
.idea/

# OS
.DS_Store
**/.DS_Store
```

---

## 6. Biome — Linter & Formatter

[Biome](https://biomejs.dev/) replaces ESLint + Prettier with a single, fast tool.

### `biome.json`

**Important:** Install Biome first (`bun add -d @biomejs/biome`), then set the `$schema` version to match the installed version. Run `bunx biome --version` to confirm.

```jsonc
{
  "$schema": "https://biomejs.dev/schemas/<installed-version>/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true          // respects .gitignore
  },
  "files": {
    "ignoreUnknown": false,
    "includes": ["src/**/*.ts", "scripts/**/*.ts"]  // lint source + build scripts
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "tab",
    "lineWidth": 80
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double"
    }
  },
  "assist": {
    "enabled": true,
    "actions": {
      "source": {
        "organizeImports": "on"
      }
    }
  }
}
```

### Scripts

| Script | Command | When to use |
|---|---|---|
| `lint:edit` | `bunx biome check --fix src/` | Local development — applies safe auto-fixes (formatting, import sorting, lint fixes) |
| `lint:type` | `tsc --noEmit` | Type-check all TypeScript files without emitting output |
| `prelint:ci` | `bun run lint:type` | Bun lifecycle hook — runs automatically before `lint:ci` |
| `lint:ci` | `bunx biome ci src/` | CI pipeline — fails on any issue, never writes files |
| `lint` | `bun run lint:edit && bun run lint:type` | Combined local command — Biome auto-fix then type check |

`lint` is the everyday command: run it before committing to fix formatting,
auto-fixable lint issues, and catch type errors in one pass. `lint:ci` is the
gatekeeper: add it to CI to enforce that all code passes without modification.
The `prelint:ci` hook ensures TypeScript type errors also fail the CI lint step.

---

## 7. Test Coverage — `bunfig.toml`

### `bunfig.toml`

```toml
[test]
root = "src"
coverageThreshold = 0.83
```

| Key | Purpose |
|---|---|
| `root` | Tells `bun test` to discover tests starting from `src/` |
| `coverageThreshold` | Minimum coverage ratio (0.0–1.0). `bun test --coverage` fails if below this |

### Script

```jsonc
"test:coverage": "bun test --coverage"
```

Run `bun run test:coverage` to execute tests with coverage reporting. The
`coverageThreshold` in `bunfig.toml` makes the command fail if coverage drops
below 83%, useful as a CI gate.

---

## 8. CI/CD — GitHub Actions (Bun)

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: oven/bun:1.2-slim
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install dependencies
        run: bun install --frozen-lockfile

      - name: Lint
        run: bun run lint:ci

      - name: Run tests
        run: bun run test:coverage

      - name: Build
        run: bun run build
```

### Pipeline flow

```
PR opened → lint → test + coverage → build
                                       ↓
Push to main → lint → test + coverage → build
```

No publish or tagging steps — the application is deployed, not published to npm.
Add deployment steps (Docker push, SSH deploy, etc.) after the build step as needed.

---

## 9. Build Script (`scripts/`)

### `scripts/verify-build.ts` — Post-build validation

Checks that the bundled output exists. Runs automatically as `postbuild`.

```typescript
import { existsSync } from "node:fs";
import { resolve } from "node:path";

const root = resolve(import.meta.dir, "..");

const required: string[] = [
  "dist/app.js",
];

let hasErrors = false;

for (const file of required) {
  const fullPath = resolve(root, file);
  if (!existsSync(fullPath)) {
    console.error(`MISSING (required): ${file}`);
    hasErrors = true;
  }
}

if (hasErrors) {
  console.error("Build verification failed.");
  process.exit(1);
}

console.log("Build verification passed.");
```

---

## 10. Checklist for New Application

- [ ] Copy project structure (src/main.ts, tsconfig.json, biome.json, bunfig.toml)
- [ ] Update package.json: name, description, author, keywords, repository
- [ ] Ensure `"private": true` is set in package.json
- [ ] Install devDependencies — verify `@types/bun` matches your `bun --version`
- [ ] Set up `biome.json` — run `bun run lint:edit` to verify
- [ ] Set up `bunfig.toml` with test root and coverage threshold
- [ ] Copy `.github/workflows/ci.yml`
- [ ] Create LICENSE file (e.g. Apache-2.0, MIT)
- [ ] Create README.md with application description
- [ ] Write application code in `src/`, tests in `src/*.test.ts`
- [ ] Run `bun test`, `bun run test:coverage`, `bun run lint:ci`, and `bun run build` locally
- [ ] Verify `bun run build` produces `dist/app.js` with no errors
- [ ] Verify `bun run start` runs the application correctly
- [ ] Push to `main` — CI runs lint, tests, and build

---

## 11. Claude Code Rules — Testing Guide

The testing guide lives at `.claude/rules/testing.md` and is sourced from the shared knowledge base. To copy or update it, run:

```bash
bun -e "
const res = await fetch('https://raw.githubusercontent.com/pencroff-lab/kb/refs/heads/main/rules/testing.md');
if (!res.ok) throw new Error('Fetch failed: ' + res.status);
const text = await res.text();
const fs = await import('node:fs');
fs.mkdirSync('.claude/rules', { recursive: true });
fs.writeFileSync('.claude/rules/testing.md', text);
console.log('Written .claude/rules/testing.md (' + text.length + ' bytes)');
"
```

Or with Node.js:

```bash
node -e "
fetch('https://raw.githubusercontent.com/pencroff-lab/kb/refs/heads/main/rules/testing.md')
  .then(r => { if (!r.ok) throw new Error('Fetch failed: ' + r.status); return r.text(); })
  .then(text => {
    const fs = require('node:fs');
    fs.mkdirSync('.claude/rules', { recursive: true });
    fs.writeFileSync('.claude/rules/testing.md', text);
    console.log('Written .claude/rules/testing.md (' + text.length + ' bytes)');
  });
"
```

This is a one-time setup step — the file is committed to the repo and does not auto-update.

---

## Appendix: Design Decisions

### Why Bun?
- Fast test runner with native TypeScript support
- Built-in bundler — no need for Webpack, esbuild, or Rollup config
- `bun --watch` for development with automatic restart
- `bun install --frozen-lockfile` for reproducible CI

### Why `bun build` (not tsc) for applications?
- Produces a single bundled file — simpler deployment
- Tree-shakes and minifies automatically
- Resolves all imports into one artifact — no `node_modules` needed at runtime
- tsc is still used for type-checking via `tsc --noEmit`

### Why single tsconfig?
- The library template needs 4 tsconfig files to produce dual ESM/CJS output with declarations
- An application only needs type-checking — the bundler handles the rest
- Less config to maintain, fewer moving parts

### Why `"private": true`?
- Applications are deployed, not published to npm
- `"private": true` prevents accidental `npm publish` or `bun publish`
- No need for `"exports"`, `"main"`, `"types"`, or `"files"` fields

### Comparison: Library vs Application Template

| Concern | Library | Application |
|---|---|---|
| Build tool | tsc (dual ESM/CJS) | `bun build` (single bundle) |
| tsconfig files | 4 (dev + build + esm + cjs) | 1 (dev only) |
| Output | `dist/esm/` + `dist/cjs/` | `dist/app.js` |
| Entry point | Root `index.ts` barrel | `src/main.ts` |
| npm publish | Yes (automated via CI) | No (`"private": true`) |
| `.npmrc` | Required (auth token) | Not needed |
| `"exports"` field | Yes (conditional imports) | Not needed |
| `"sideEffects"` | `false` (tree-shaking hint) | Not needed |
| `"files"` | `["dist"]` (tarball allowlist) | Not needed |
| Build scripts | fix-cjs, verify-build, check-version | verify-build only |