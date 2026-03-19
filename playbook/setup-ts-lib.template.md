# TypeScript Library → npm Publish Template

Step-by-step template extracted from `@pencroff-lab/wyhash-ts`.
Goal: reuse this pattern for any new TypeScript library published to npm.

---

## 1. Project Layout

```
my-library/
├── index.ts                  # Re-export entry point (thin barrel file)
├── src/
│   ├── core.ts               # Main library code
│   ├── core.test.ts          # Tests (co-located)
│   └── core.bench.ts         # Benchmarks (optional)
├── scripts/
│   ├── fix-cjs.ts            # Writes CJS package.json for Node compat
│   ├── verify-build.ts       # Post-build artifact validation
│   ├── check-version.ts      # Pre-publish npm version duplicate check
│   ├── check-docs.ts         # Doc budget enforcement (JSDoc ratio checks)
│   ├── fix-docs-links.ts     # Fixes TypeDoc _media/ links to relative paths
│   └── fetch-to-folder.ts    # Fetch remote file to a local folder
├── docs/                     # Documentation (published to npm)
│   ├── api/                  # TypeDoc-generated API docs (markdown)
│   └── guides/               # Hand-written guides
├── dist/                     # Build output (git-ignored, npm-published)
│   ├── esm/                  # ES modules build
│   └── cjs/                  # CommonJS build
├── build/                    # Intermediate artifacts (git-ignored)
├── tsconfig.json             # IDE / development config (noEmit)
├── tsconfig.build.json       # Shared build settings
├── tsconfig.esm.json         # ESM-specific build
├── tsconfig.cjs.json         # CJS-specific build
├── biome.json                # Biome linter / formatter config
├── bunfig.toml               # Bun runtime config (test coverage, etc.)
├── typedoc.json              # TypeDoc configuration (markdown output)
├── cliff.toml                # git-cliff changelog configuration
├── package.json
├── CHANGELOG.md              # Auto-generated changelog (git-cliff)
├── llms.txt                  # LLM-friendly package summary
├── .npmrc
├── .gitignore
└── .github/workflows/ci.yml  # CI + auto-publish
```

Key principle: `index.ts` sits at the root and re-exports from `src/`.
Tests, benchmarks, and specs live alongside source but are excluded from builds.

---

## 2. package.json — Essential Fields

```jsonc
{
  "name": "<@scope/package-name>",            // ← your scoped package name
  "version": "0.1.0",
  "description": "<package description>",     // ← short library description
  "license": "Apache-2.0",
  "author": "<Author Name>",                  // ← your name or org
  "repository": {
    "type": "git",
    "url": "https://github.com/org/my-library"
  },
  "keywords": ["..."],

  // ── Module system ──
  "type": "module",                         // ESM by default

  // ── Entry points (legacy + modern) ──
  "main": "./dist/cjs/index.js",            // require()
  "module": "./dist/esm/index.js",          // import (bundler hint)
  "types": "./dist/esm/index.d.ts",         // TypeScript types

  // ── Conditional exports (Node 12.20+) ──
  "exports": {
    ".": {
      "types": "./dist/esm/index.d.ts",
      "import": "./dist/esm/index.js",
      "require": "./dist/cjs/index.js"
    }
  },

  // ── Tree-shaking hint for bundlers ──
  "sideEffects": false,

  // ── What goes into the tarball ──
  "files": ["dist", "docs", "llms.txt", "CHANGELOG.md"],

  // ── Scripts ──
  "scripts": {
    "test": "bun test",
    "test:coverage": "bun test --coverage",
    "lint:edit": "bunx biome check --fix src/",
    "lint:type": "tsc --noEmit",
    "prelint:ci": "bun run lint:type",
    "lint:ci": "bunx biome ci src/",
    "check:docs": "bun scripts/check-docs.ts",
    "lint": "bun run lint:edit && bun run lint:type && bun run check:docs",
    "prebuild": "rm -rf dist",
    "build": "bun run build:esm && bun run build:cjs",
    "build:esm": "tsc --project tsconfig.esm.json",
    "build:cjs": "tsc --project tsconfig.cjs.json && bun scripts/fix-cjs.ts",
    "postbuild": "bun scripts/verify-build.ts",
    "pregen_docs": "rm -rf docs/api/_media",
    "gen_docs": "typedoc && bun scripts/fix-docs-links.ts",
    "changelog": "git-cliff -o CHANGELOG.md",
    "changelog:latest": "git-cliff --latest --strip header",
    "bump_version": "git-cliff --bump -o CHANGELOG.md",
    "bump_echo": "echo 'Update package.json to version' && git-cliff --bumped-version",
    "release": "bun run test:coverage && bun run lint && bun run build && bun run gen_docs && bun run bump_version && bun run bump_echo",
    "prepublish_pkg": "bun scripts/check-version.ts && bun run build",
    "publish_pkg": "bun publish --access public"
  },

  // ── Dependencies ──
  "devDependencies": {
    "@biomejs/biome": "2.4.2",              // exact latest version
    "@types/bun": "1.3.9",                  // match your bun version (bun --version)
    "@types/sinon": "21.0.0",               // exact latest version
    "git-cliff": "2.12.0",                  // exact latest version — changelog generation
    "mitata": "1.0.34",                     // exact latest version — benchmarking
    "sinon": "21.0.1",                      // exact latest version — test stubs/spies
    "typedoc": "0.28.17",                   // exact latest version — API doc generation
    "typedoc-plugin-markdown": "4.10.0"     // exact latest version — markdown output
  },
  "peerDependencies": {
    "typescript": "5.9.3"
  }
}
```

### User-provided values

| Placeholder | What to fill in |
|---|---|
| `<@scope/package-name>` | Your npm scope and package name (e.g. `@pencroff-lab/kore`) |
| `<package description>` | Short description shown on npmjs.com |
| `<Author Name>` | Your name or organisation |

### Field-by-field notes

| Field | Purpose |
|---|---|
| `"type": "module"` | Makes `.js` files ESM by default in Node |
| `"main"` | CJS entry for `require()` consumers |
| `"module"` | ESM entry for bundlers (Webpack, Rollup, esbuild) |
| `"types"` | TypeScript declaration entry |
| `"exports"` | Modern conditional exports — Node resolves `import`/`require` automatically |
| `"sideEffects": false` | Tells bundlers all modules are safe to tree-shake when unused |
| `"files": [...]` | Published to npm: build output, generated docs, changelog, and LLM summary |
| `"lint:type"` | Runs `tsc --noEmit` to type-check without emitting files |
| `"prelint:ci"` | Bun lifecycle hook — runs `lint:type` automatically before `lint:ci` |
| `"check:docs"` | Checks JSDoc doc budget ratios — fails if over budget |
| `"lint"` | Combined local command: Biome auto-fix + TypeScript type check + doc budget |
| `"build:cjs"` | CJS compile + writes `dist/cjs/package.json` with `"type": "commonjs"` for Node compat |
| `"postbuild"` | Validates required build artifacts exist; warns if LICENSE or README.md is missing |
| `"gen_docs"` | Generates TypeDoc API docs + fixes `_media/` links to relative `src/` paths |
| `"pregen_docs"` | Cleans `_media/` directory before doc generation |
| `"changelog"` | Generates full `CHANGELOG.md` from git history via git-cliff |
| `"changelog:latest"` | Outputs only the latest release changelog (for release notes) |
| `"bump_version"` | Bumps changelog with next version via git-cliff |
| `"bump_echo"` | Prints the next version to bump `package.json` to |
| `"release"` | Full release pipeline: test → lint → build → gen docs → bump version |
| `"prepublish_pkg"` | Checks npm registry for version duplicate, then runs full build |

### Dependency version policy

- **`@types/bun`** — should match the bun version used in the project. Run `bun --version` and pick the corresponding `@types/bun` release to keep type definitions in sync.
- **All other devDependencies** — use exact latest versions (no `^` or `~`). Check the latest version on npm before installing. This ensures deterministic installs and avoids surprise breakage from patch releases.

### Dev dependency roles

| Package | Purpose |
|---|---|
| `@biomejs/biome` | Linter and formatter (replaces ESLint + Prettier) |
| `@types/bun` | TypeScript definitions for the Bun runtime |
| `@types/sinon` | TypeScript definitions for Sinon |
| `git-cliff` | Changelog generation from conventional commits |
| `mitata` | Micro-benchmarking library for `*.bench.ts` files |
| `sinon` | Test stubs, spies, and mocks |
| `typedoc` | API documentation generation from JSDoc/TSDoc |
| `typedoc-plugin-markdown` | TypeDoc output as markdown files (instead of HTML) |

---

## 3. TypeScript Configs — 3-file Hierarchy

### 3a. `tsconfig.json` — IDE & development (no emit)

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

**Why `noEmit: true`?** This config is for IDE tooling and `bun test` — it never
produces files. The build configs below override this.

### 3b. `tsconfig.build.json` — Shared build settings

```jsonc
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noEmit": false,                        // Override: actually emit
    "moduleResolution": "node",             // Override: Node resolution for consumers
    "allowImportingTsExtensions": false,    // Override: must emit .js not .ts
    "verbatimModuleSyntax": false           // Override: allow emit transforms
  },
  "include": ["index.ts"],
  "exclude": [
    "node_modules", "dist", "build", "release", "tmp",
    "**/*.test.ts", "**/*.spec.ts", "**/*.bench.ts"
  ]
}
```

**Why `include: ["index.ts"]`?** The barrel file at the root imports from `src/`,
so tsc follows the dependency graph automatically. No need to glob `src/**`.

**Why `moduleResolution: "node"`?** Published packages must resolve without
bundler-specific heuristics. Consumers may use Node, Webpack, or anything else.

### 3c. `tsconfig.esm.json` — ESM output

```jsonc
{
  "extends": "./tsconfig.build.json",
  "compilerOptions": {
    "module": "ES2022",
    "outDir": "./dist/esm"
  }
}
```

### 3d. `tsconfig.cjs.json` — CJS output

```jsonc
{
  "extends": "./tsconfig.build.json",
  "compilerOptions": {
    "module": "CommonJS",
    "outDir": "./dist/cjs"
  }
}
```

**Result**: Two parallel builds from the same source, both with `.d.ts` and source maps.

---

## 4. Entry Point Pattern & Tree-Shaking

### `index.ts` (root)

```ts
export * from './src/core';
export * from './src/utils';
```

This thin barrel file is the only thing `tsconfig.build.json` includes.
tsc walks imports to compile the full dependency tree.

### Tree-shaking: how it works with barrel re-exports

Barrel re-exports (`export * from`) **are tree-shakable** by modern bundlers
(Rollup, esbuild, Vite, Webpack 5). When a consumer writes:

```ts
import { wyhash } from '@scope/my-lib';
```

The bundler traces only the used export. Functions and modules never referenced
are dropped from the final bundle — **if** two conditions are met:

1. **No side effects** — modules must not execute logic at the top level
   (no global mutations, no `console.log()`, no polyfill injection).
   Pure `const` declarations and `function`/`class` exports are fine.
2. **`"sideEffects": false`** in `package.json` — signals to bundlers that
   entire unreferenced modules can be safely eliminated.

Without `"sideEffects": false`, bundlers must conservatively keep every imported
module in case it does something on load.

### Rules for what to re-export from the barrel

Only re-export **logically connected components** that are expected to be used
together. The grouping rule:

1. **Cohesion** — modules in the same barrel should form a coherent API surface.
   If a consumer imports one thing, they will likely need the others.
2. **No kitchen sinks** — do not dump unrelated utilities into one barrel.
   If two modules serve different use cases and have different audiences,
   consider separate entry points via `"exports"` sub-paths:
   ```jsonc
   "exports": {
     ".": { "import": "./dist/esm/index.js" },
     "./utils": { "import": "./dist/esm/utils.js" }
   }
   ```
3. **Avoid deep barrel chains** — one level of re-export is fine. Barrel files
   that re-export from other barrel files create resolution overhead and can
   confuse older bundlers.
4. **Keep modules side-effect-free** — no top-level `console.log()`, no
   global state mutation, no `addEventListener()`. This is what makes
   tree-shaking actually work.

### Example: good vs bad barrel grouping

```ts
// GOOD — logically connected, consumer uses both together
export * from './src/wyhash';      // hash function
export * from './src/wyhash_bun';  // same hash, alternative impl

// BAD — unrelated concerns in one barrel
export * from './src/wyhash';      // hash function
export * from './src/logger';      // unrelated utility
export * from './src/http-client'; // completely different domain
```

---

## 5. npm Registry Auth — `.npmrc`

```ini
@your-scope:registry=https://registry.npmjs.org/
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

- Scoped to your org (`@your-scope`)
- Token injected via environment variable (never committed)
- Works with both `npm publish` and `bun publish`

---

## 6. `.gitignore` — Essential Entries

```gitignore
# Build output
dist/
build/

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

Note: `dist/` is git-ignored but npm-published via `"files": ["dist"]`.

---

## 7. Biome — Linter & Formatter

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

## 8. Test Coverage — `bunfig.toml`

### `bunfig.toml`

```toml
[test]
root = "src"
coverageThreshold = 0.83

[test.reporter]
dots = true
```

| Key | Purpose |
|---|---|
| `root` | Tells `bun test` to discover tests starting from `src/` |
| `coverageThreshold` | Minimum coverage ratio (0.0–1.0). `bun test --coverage` fails if below this |
| `dots` | Uses dots reporter for compact test output (one dot per test) |

### Script

```jsonc
"test:coverage": "bun test --coverage"
```

Run `bun run test:coverage` to execute tests with coverage reporting. The
`coverageThreshold` in `bunfig.toml` makes the command fail if coverage drops
below 83%, useful as a CI gate.

---

## 9. CI/CD — GitHub Actions (Bun)

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test_publish:
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

      - name: Check docs
        run: bun run check:docs

      - name: Run tests
        run: bun run test:coverage

      - name: Publish to npm
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: bun run publish_pkg
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: Create git tag from version
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.REPO_TAG_TOKEN }}
          script: |
            const fs = require('node:fs');
            const { version } = JSON.parse(fs.readFileSync('./package.json', 'utf8'));
            const tag = `v${version}`;
            core.info(`Ensuring tag ${tag} points at ${context.sha}`);
            try {
              await github.rest.git.getRef({
                owner: context.repo.owner,
                repo: context.repo.repo,
                ref: `tags/${tag}`,
              });
              core.info(`Tag ${tag} already exists; skipping.`);
            } catch (error) {
              if (error.status !== 404) throw error;
              await github.rest.git.createRef({
                owner: context.repo.owner,
                repo: context.repo.repo,
                ref: `refs/tags/${tag}`,
                sha: context.sha,
              });
              core.info(`Created tag ${tag}`);
            }

      - name: Verify published version
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: bun info @your-scope/my-library
        env:
          NPM_TOKEN: "---"
```

### Pipeline flow

```
PR opened → lint → test + coverage
                         ↓
Push to main → lint → test + coverage → publish to npm → tag git commit
```

### Required GitHub secrets

| Secret | Purpose |
|---|---|
| `NPM_TOKEN` | npm automation token with publish scope |
| `REPO_TAG_TOKEN` | GitHub PAT with `contents:write` for tag creation |

---

## 10. Release & Publish Workflow

### Release (local — prepares the next version)

```bash
# Run the full release pipeline:
# test:coverage → lint → build → gen_docs → bump_version → bump_echo
bun run release

# The release script outputs the next version to set in package.json.
# Update package.json version, commit, and push to main.
```

The `release` script runs tests, linting, build, regenerates API docs,
and bumps the changelog — then tells you what version to set in `package.json`.

### Publish (CI — automatic on push to main)

```bash
# Manual publish (if needed):
# 1. Check version is not already published
bun scripts/check-version.ts

# 2. Build (includes CJS fix and post-build verification)
bun run build

# 3. Dry run
bun publish --access public --dry-run

# 4. Publish
bun run publish_pkg
```

The CI pipeline automates publish + tagging on every push to `main`.
`prepublish_pkg` runs the version check and build automatically before `publish_pkg`.

---

## 11. Dual ESM/CJS Build — How It Works

```
index.ts ─┬─ tsc (tsconfig.esm.json) ──→ dist/esm/*.js   (ES modules)
           │                                dist/esm/*.d.ts (declarations)
           │
           └─ tsc (tsconfig.cjs.json) ──→ dist/cjs/*.js   (CommonJS)
                                           dist/cjs/*.d.ts (declarations)
```

Consumers get the right format automatically:
- `import { hash } from '@scope/lib'` → `dist/esm/index.js`
- `const { hash } = require('@scope/lib')` → `dist/cjs/index.js`
- TypeScript → `dist/esm/index.d.ts` (via `"types"` field)

### CJS compatibility fix

The root `package.json` declares `"type": "module"`, which makes Node.js treat
all `.js` files as ESM. This breaks `require()` for files in `dist/cjs/` even
though they contain valid CommonJS syntax.

The fix: `scripts/fix-cjs.ts` writes a `dist/cjs/package.json` with
`{"type": "commonjs"}` after the CJS build. Node's module resolution checks the
nearest `package.json` when determining module type, so this override takes
effect for everything in `dist/cjs/`.

This runs automatically as part of `build:cjs` and is verified by `postbuild`.

---

## 12. Build Scripts (`scripts/`)

All scripts use `import.meta.dir` (bun API) to resolve paths relative to the
script file, so they work regardless of the working directory.

### `scripts/fix-cjs.sh.ts` — CJS compatibility

Writes `dist/cjs/package.json` with `{"type": "commonjs"}` after the CJS build.
Runs automatically as part of `build:cjs`.

```typescript
import { existsSync, writeFileSync } from "node:fs";
import { resolve } from "node:path";

const cjsDir = resolve(import.meta.dir, "..", "dist", "cjs");

if (!existsSync(cjsDir)) {
  console.error("dist/cjs/ does not exist. Run build:cjs first.");
  process.exit(1);
}

const pkgPath = resolve(cjsDir, "package.json");
writeFileSync(pkgPath, `${JSON.stringify({ type: "commonjs" }, null, "\t")}\n`);
console.log("Wrote dist/cjs/package.json with type: commonjs");
```

### `scripts/verify-build.sh.ts` — Post-build validation

Checks that required dist artifacts exist (error if missing) and warns about
optional files (LICENSE, README.md). Runs automatically as `postbuild`.

```typescript
import { existsSync } from "node:fs";
import { resolve } from "node:path";

const root = resolve(import.meta.dir, "..");

const required: string[] = [
  "dist/esm/index.js",
  "dist/esm/index.d.ts",
  "dist/cjs/index.js",
  "dist/cjs/package.json",
];

const optional: Array<{ path: string; label: string }> = [
  { path: "LICENSE", label: "LICENSE" },
  { path: "README.md", label: "README.md" },
];

let hasErrors = false;

for (const file of required) {
  const fullPath = resolve(root, file);
  if (!existsSync(fullPath)) {
    console.error(`MISSING (required): ${file}`);
    hasErrors = true;
  }
}

for (const { path, label } of optional) {
  const fullPath = resolve(root, path);
  if (!existsSync(fullPath)) {
    console.warn(`WARNING: ${label} is missing — recommended for npm packages`);
  }
}

if (hasErrors) {
  console.error("Build verification failed.");
  process.exit(1);
}

console.log("Build verification passed.");
```

### `scripts/check-docs.sh.ts` — Doc budget enforcement

Scans `src/**/*.ts` files (excluding tests and barrel indexes) and enforces JSDoc
documentation budget ratios. Tiered by file size (threshold: 100 non-blank lines).
The first `@module` JSDoc block is excluded from the ratio.

| File type | Small (< 100 lines) | Normal (≥ 100 lines) |
|-----------|---------------------|----------------------|
| `*.types.ts` | ≤ 80% | ≤ 50% |
| Implementation `*.ts` | ≤ 50% | ≤ 35% |

Additional rules enforced:
- **`*.types.ts`**: no file-level JSDoc (TypeDoc won't render it), `@example` blocks max 5 lines
- **Implementation `*.ts`**: no `@example` blocks (use `*.examples.test.ts` + `@see` link)

Runs as part of `bun run check:docs` and `bun run lint`. Exits with code 1 on errors.

### `scripts/fix-docs-links.sh.ts` — TypeDoc link fixer

After TypeDoc generates docs, it copies `@see` file references into a `_media/`
directory. This script replaces those `_media/` links with relative paths back to
`src/` and removes the `_media/` directory. Runs automatically as part of `gen_docs`.

### `scripts/check-version.sh.ts` — Pre-publish version check

Queries the npm registry to verify the current version is not already published.
Runs automatically as part of `prepublish_pkg` before `publish_pkg`.

- **200** (version exists) → exit 1 with "bump version" message
- **404** (available) → exit 0
- **Network error** → warning only (publish will fail with its own error)

```typescript
import { readFileSync } from "node:fs";
import { resolve } from "node:path";

const pkgPath = resolve(import.meta.dir, "..", "package.json");
const pkg = JSON.parse(readFileSync(pkgPath, "utf-8")) as {
  name: string;
  version: string;
};
const { name, version } = pkg;

const url = `https://registry.npmjs.org/${name}/${version}`;

try {
  const res = await fetch(url);

  if (res.ok) {
    console.error(`Version ${version} of ${name} already exists on npm.`);
    console.error("Bump the version in package.json before publishing.");
    process.exit(1);
  }

  if (res.status === 404) {
    console.log(`Version ${version} of ${name} is available for publishing.`);
    process.exit(0);
  }

  console.warn(`Unexpected response from npm registry: HTTP ${res.status}`);
} catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  console.warn(`Could not reach npm registry: ${message}`);
}
```

---

## 13. Checklist for New Library

- [ ] Copy project structure (index.ts, src/, 4 tsconfigs, biome.json, bunfig.toml)
- [ ] Update package.json: name, description, author, keywords, repository, `"sideEffects": false`
- [ ] Install devDependencies (exact latest versions) — verify `@types/bun` matches your `bun --version`
- [ ] Set up `biome.json` — run `bun run lint:edit` to verify
- [ ] Set up `bunfig.toml` with test root, coverage threshold, and dots reporter
- [ ] Set up `.npmrc` with your scope
- [ ] Set up `typedoc.json` — configure entry points and output (see [section 17](#17-typedoc-setup))
- [ ] Set up `cliff.toml` — configure changelog generation (see [section 18](#18-git-cliff-setup))
- [ ] Create npm automation token, add as `NPM_TOKEN` secret
- [ ] Create GitHub PAT for tagging, add as `REPO_TAG_TOKEN` secret
- [ ] Copy `.github/workflows/ci.yml`, update package name in verify step
- [ ] Create `scripts/fetch-to-folder.ts` (see [section 14](#14-fetch-to-folder-helper-script))
- [ ] Create `scripts/check-docs.ts` and `scripts/fix-docs-links.ts` (see [section 12](#12-build-scripts-scripts))
- [ ] Copy testing rules (see [section 15](#15-claude-code-rules--testing-guide))
- [ ] _(Optional)_ If logging is needed — install `@pencroff-lab/kore` and copy logging rules (see [section 16](#16-claude-code-rules--logging-guide-optional))
- [ ] Create LICENSE file (e.g. Apache-2.0, MIT) — **must exist before publish**
- [ ] Create README.md with package description and usage examples
- [ ] Create `llms.txt` with LLM-friendly package summary
- [ ] Write library code in `src/`, tests in `src/*.test.ts`
- [ ] Run `bun test`, `bun run test:coverage`, `bun run lint:ci`, and `bun run build` locally
- [ ] Verify `bun run build` shows no errors and no missing-file warnings
- [ ] Run `bun run gen_docs` — verify docs generate without errors
- [ ] Run `bun scripts/check-version.ts` — verify version is available on npm
- [ ] Push to `main` — CI handles publish + tagging
- [ ] Verify on npmjs.com: `https://www.npmjs.com/package/@scope/name`

---

## 14. Fetch-to-Folder Helper Script

Create `scripts/fetch-to-folder.ts` — a reusable helper that downloads a remote file into a local folder.

### `scripts/fetch-to-folder.ts`

```typescript
import { mkdirSync, writeFileSync } from "node:fs";
import { resolve } from "node:path";

const [filename, sourceUrl, targetFolder] = process.argv.slice(2);

if (!filename || !sourceUrl || !targetFolder) {
  console.error(
    "Usage: bun scripts/fetch-to-folder.ts <filename> <source_url> <target_folder>",
  );
  process.exit(1);
}

const res = await fetch(sourceUrl);
if (!res.ok) {
  console.error(`Fetch failed: ${res.status} ${res.statusText}`);
  process.exit(1);
}

const text = await res.text();
mkdirSync(targetFolder, { recursive: true });
const dest = resolve(targetFolder, filename);
writeFileSync(dest, text);
console.log(`Written ${dest} (${text.length} bytes)`);
```

Review the script, then use it in the steps below.

---

## 15. Claude Code Rules — Testing Guide

The testing guide lives at `.claude/rules/testing.rule.md` and is sourced from the shared knowledge base. To copy or update it, run:

```bash
bun scripts/fetch-to-folder.ts testing.rule.md \
  https://raw.githubusercontent.com/pencroff-lab/kb/refs/heads/main/rules/testing.rule.md \
  .claude/rules
```

---

## 16. Claude Code Rules — Logging Guide (Optional)

> **Ask the user:** "Does this library need structured logging? This adds `@pencroff-lab/kore` as a dependency."
>
> If **no** — skip this section entirely.

If **yes** — install the dependency and copy the logging rules:

```bash
bun add -E @pencroff-lab/kore
```

```bash
bun scripts/fetch-to-folder.ts logging.rule.md \
  https://raw.githubusercontent.com/pencroff-lab/kb/refs/heads/main/rules/logging.rule.md \
  .claude/rules

bun scripts/fetch-to-folder.ts logging-test.rule.md \
  https://raw.githubusercontent.com/pencroff-lab/kb/refs/heads/main/rules/logging-test.rule.md \
  .claude/rules
```

These are one-time setup steps — the files are committed to the repo and do not auto-update.

---

## 17. TypeDoc Setup

TypeDoc generates API documentation as markdown files from JSDoc/TSDoc comments.

### `typedoc.json`

```jsonc
{
  "$schema": "https://typedoc.org/schema.json",
  "plugin": ["typedoc-plugin-markdown"],
  "entryPoints": [
    "./src/types/core.ts"                   // ← list your module entry points
  ],
  "out": "docs/api",
  "tsconfig": "tsconfig.json",
  "basePath": "./src",
  "name": "<@scope/package-name>",          // ← your package name
  "readme": "none",
  "outputFileStrategy": "modules",
  "flattenOutputFiles": false,
  "entryFileName": "README.md",
  "hidePageHeader": false,
  "hideBreadcrumbs": false,
  "excludePrivate": true,
  "excludeInternal": true,
  "sourceLinkTemplate": "../../src/{path}#L{line}",
  "disableGit": true
}
```

### Key settings

| Setting | Purpose |
|---|---|
| `plugin: ["typedoc-plugin-markdown"]` | Output markdown instead of HTML |
| `entryPoints` | List each module's implementation file (not `*.types.ts`) — TypeDoc only renders `@module` from entry points |
| `out: "docs/api"` | Output directory for generated docs |
| `sourceLinkTemplate` | Relative paths from `docs/api/` back to `src/` — avoids hardcoded GitHub commit URLs |
| `disableGit: true` | Required for `sourceLinkTemplate` to use relative paths |
| `excludePrivate` / `excludeInternal` | Keeps internal implementation details out of published docs |

### Doc generation pipeline

```
pregen_docs (clean _media/) → typedoc (generate) → fix-docs-links.sh.ts (fix _media/ links)
```

Run with `bun run gen_docs`. The `pregen_docs` script runs automatically before `gen_docs`.

---

## 18. git-cliff Setup

git-cliff generates changelogs from conventional commits.

### `cliff.toml`

```toml
[changelog]
header = "# Changelog\n"
body = """

{% if version %}\
## [{{ version | trim_start_matches(pat="v") }}] - {{ timestamp | date(format="%Y-%m-%d") }}
{% else %}\
## [Unreleased]
{% endif %}\
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits | unique(attribute="message") %}
- {{ commit.message | split(pat="\n") | first | upper_first }} ({{ commit.id | truncate(length=7, end="") }})\
{% endfor %}
{% endfor %}
"""

[git]
conventional_commits = true
filter_unconventional = false
commit_parsers = [
    { message = "^feat", group = "Features" },
    { message = "^fix", group = "Bug Fixes" },
    { message = "^doc", group = "Documentation" },
    { message = "^perf", group = "Performance" },
    { message = "^refactor", group = "Refactor" },
    { message = "^chore", skip = true },
    { message = "^ci", skip = true },
    { message = "^\\.$", skip = true },
    { message = "^$", skip = true },
    { message = ".*", group = "Other" },
]
```

### Usage

| Command | Purpose |
|---|---|
| `bun run changelog` | Regenerate full `CHANGELOG.md` from entire git history |
| `bun run changelog:latest` | Output only the latest release notes (useful for GitHub releases) |
| `bun run bump_version` | Bump changelog with the next semver version |
| `bun run bump_echo` | Print the next version — update `package.json` to match |

The `release` script runs `bump_version` + `bump_echo` as part of the full pipeline.

---

## Appendix: Design Decisions

### Why Bun?
- Fast test runner with native TypeScript support
- `bun publish` works like `npm publish` but faster
- `bun install --frozen-lockfile` for reproducible CI

### Why tsc (not bundler) for library builds?
- Produces clean, unbundled output consumers can tree-shake
- Generates `.d.ts` declarations and source maps natively
- No bundler config to maintain

### Why dual ESM + CJS?
- ESM: modern bundlers (Vite, esbuild, Rollup) and Node with `--experimental-modules` or `"type": "module"`
- CJS: legacy Node apps, Jest default config, older toolchains
- Conditional exports route consumers automatically

### Why `"sideEffects": false`?
- Without it, bundlers assume every module might execute code on import
- With it, entire unreferenced modules are dropped — not just unused exports
- Only safe when modules have no top-level side effects (our case: only `const` + `function` exports)

### Why `"files": [...]` instead of `.npmignore`?
- Allowlist is safer than denylist — only published files are explicitly listed
- Prevents accidental inclusion of tests, IDE configs, secrets
- Includes `dist`, `docs`, `llms.txt`, and `CHANGELOG.md` — consumers get build output, documentation, and changelog in the package