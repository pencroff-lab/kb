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
├── dist/                     # Build output (git-ignored, npm-published)
│   ├── esm/                  # ES modules build
│   └── cjs/                  # CommonJS build
├── build/                    # Intermediate artifacts (git-ignored)
├── tsconfig.json             # IDE / development config (noEmit)
├── tsconfig.build.json       # Shared build settings
├── tsconfig.esm.json         # ESM-specific build
├── tsconfig.cjs.json         # CJS-specific build
├── package.json
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
  "name": "@scope/my-library",
  "version": "0.1.0",
  "description": "...",
  "license": "Apache-2.0",
  "author": "YourName",
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
  "files": ["dist"],

  // ── Scripts ──
  "scripts": {
    "test": "bun test",
    "build": "bun run build:esm && bun run build:cjs",
    "prebuild": "rm -rf dist",
    "build:esm": "tsc --project tsconfig.esm.json",
    "build:cjs": "tsc --project tsconfig.cjs.json",
    "prepublish_pkgOnly": "bun run build",
    "publish_pkg": "bun publish --access public"
  },

  // ── Dependencies ──
  "devDependencies": {
    "@types/bun": "..."
  },
  "peerDependencies": {
    "typescript": "5.x"
  }
}
```

### Field-by-field notes

| Field | Purpose |
|---|---|
| `"type": "module"` | Makes `.js` files ESM by default in Node |
| `"main"` | CJS entry for `require()` consumers |
| `"module"` | ESM entry for bundlers (Webpack, Rollup, esbuild) |
| `"types"` | TypeScript declaration entry |
| `"exports"` | Modern conditional exports — Node resolves `import`/`require` automatically |
| `"sideEffects": false` | Tells bundlers all modules are safe to tree-shake when unused |
| `"files": ["dist"]` | Only the `dist/` folder is published to npm (no src, tests, tmp) |
| `"prepublish_pkgOnly"` | Runs `build` before `bun publish` to ensure dist is fresh |

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
!.env.example

# Temporary
tmp/

# IDE
.idea/

# OS
.DS_Store
```

Note: `dist/` is git-ignored but npm-published via `"files": ["dist"]`.

---

## 7. CI/CD — GitHub Actions (Bun)

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

      - name: Run tests
        run: bun test

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
PR opened → test
             ↓
Push to main → test → publish to npm → tag git commit
```

### Required GitHub secrets

| Secret | Purpose |
|---|---|
| `NPM_TOKEN` | npm automation token with publish scope |
| `REPO_TAG_TOKEN` | GitHub PAT with `contents:write` for tag creation |

---

## 8. Publish Workflow (Manual)

```bash
# 1. Bump version
# Edit package.json version manually or use `npm version patch|minor|major`

# 2. Build
bun run build

# 3. Verify dist contents
ls dist/esm/ dist/cjs/

# 4. Dry run
bun publish --access public --dry-run

# 5. Publish
bun run publish_pkg
```

The CI pipeline automates steps 2-5 on every push to `main`.

---

## 9. Dual ESM/CJS Build — How It Works

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

---

## 10. Checklist for New Library

- [ ] Copy project structure (index.ts, src/, 4 tsconfigs)
- [ ] Update package.json: name, description, keywords, repository, `"sideEffects": false`
- [ ] Set up `.npmrc` with your scope
- [ ] Create npm automation token, add as `NPM_TOKEN` secret
- [ ] Create GitHub PAT for tagging, add as `REPO_TAG_TOKEN` secret
- [ ] Copy `.github/workflows/ci.yml`, update package name in verify step
- [ ] Write library code in `src/`, tests in `src/*.test.ts`
- [ ] Run `bun test` and `bun run build` locally
- [ ] Push to `main` — CI handles publish + tagging
- [ ] Verify on npmjs.com: `https://www.npmjs.com/package/@scope/name`

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

### Why `"files": ["dist"]` instead of `.npmignore`?
- Allowlist is safer than denylist — only published files are explicitly listed
- Prevents accidental inclusion of tests, docs, IDE configs, secrets