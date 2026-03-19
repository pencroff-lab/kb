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
│   ├── verify-build.ts       # Post-build artifact validation
│   └── fetch-to-folder.ts    # Fetch remote file to a local folder
├── dist/                     # Build output (git-ignored)
├── tsconfig.json             # IDE / development config (noEmit)
├── biome.json                # Biome linter / formatter config
├── bunfig.toml               # Bun runtime config (test coverage, etc.)
├── cliff.toml                # git-cliff changelog configuration
├── package.json
├── CHANGELOG.md              # Auto-generated changelog (git-cliff)
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
    "postbuild": "bun scripts/verify-build.ts",
    "changelog": "git-cliff -o CHANGELOG.md",
    "changelog:latest": "git-cliff --latest --strip header",
    "bump_version": "git-cliff --bump -o CHANGELOG.md",
    "bump_echo": "echo 'Update package.json to version' && git-cliff --bumped-version",
    "release": "bun run test:coverage && bun run lint && bun run build && bun run bump_version && bun run bump_echo"
  },

  // ── Dependencies ──
  "dependencies": {
    "@pencroff-lab/kore": "0.3.1",             // exact latest version
    "lodash": "4.17.23"                        // exact latest version
  },
  "devDependencies": {
    "@biomejs/biome": "2.4.2",                 // exact latest version
    "@types/bun": "1.3.9",                     // match your bun version (bun --version)
    "@types/lodash": "4.17.17",                // exact latest version
    "@types/sinon": "21.0.0",                  // exact latest version
    "git-cliff": "2.12.0",                     // exact latest version — changelog generation
    "mitata": "1.0.34",                        // exact latest version — benchmarking
    "sinon": "21.0.1",                         // exact latest version — test stubs/spies
    "typescript": "5.9.3"                      // exact latest version
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
| `"changelog"` | Generates full `CHANGELOG.md` from git history via git-cliff |
| `"changelog:latest"` | Outputs only the latest release changelog (for release notes) |
| `"bump_version"` | Bumps changelog with next version via git-cliff |
| `"bump_echo"` | Prints the next version to bump `package.json` to |
| `"release"` | Full release pipeline: test → lint → build → bump version |

### Dependency version policy

- **`@types/bun`** — should match the bun version used in the project. Run `bun --version` and pick the corresponding `@types/bun` release to keep type definitions in sync.
- **All other dependencies** — use exact latest versions (no `^` or `~`). Check the latest version on npm before installing. This ensures deterministic installs and avoids surprise breakage from patch releases.
- **`typescript`** — listed as a devDependency (not peerDependency) since the app is not consumed by other packages.

### Dev dependency roles

| Package | Purpose |
|---|---|
| `@biomejs/biome` | Linter and formatter (replaces ESLint + Prettier) |
| `@types/bun` | TypeScript definitions for the Bun runtime |
| `@types/lodash` | TypeScript definitions for Lodash |
| `@types/sinon` | TypeScript definitions for Sinon |
| `git-cliff` | Changelog generation from conventional commits |
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

## 9. git-cliff Setup

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

## 10. Release Workflow

### Release (local — prepares the next version)

```bash
# Run the full release pipeline:
# test:coverage → lint → build → bump_version → bump_echo
bun run release

# The release script outputs the next version to set in package.json.
# Update package.json version, commit, and push to main.
```

The `release` script runs tests, linting, build, and bumps the changelog — then
tells you what version to set in `package.json`.

---

## 11. Build Script (`scripts/`)

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

## 12. Install Dependencies

Install the runtime dependencies:

```bash
bun add -E @pencroff-lab/kore lodash
```

Install all dev dependencies in one command:

```bash
bun add -E -d @biomejs/biome @types/bun @types/lodash @types/sinon git-cliff mitata sinon typescript
```

Verify `@types/bun` matches your Bun version (`bun --version`) and update if needed.

---

## 13. Checklist for New Application

- [ ] Copy project structure (src/main.ts, tsconfig.json, biome.json, bunfig.toml)
- [ ] Update package.json: name, description, author, keywords, repository
- [ ] Ensure `"private": true` is set in package.json
- [ ] Install dependencies (see [section 12](#12-install-dependencies))
- [ ] Set up `biome.json` — run `bun run lint:edit` to verify
- [ ] Set up `bunfig.toml` with test root and coverage threshold
- [ ] Set up `cliff.toml` — configure changelog generation (see [section 9](#9-git-cliff-setup))
- [ ] Copy `.github/workflows/ci.yml`
- [ ] Create `scripts/fetch-to-folder.ts` (see [section 14](#14-fetch-to-folder-helper-script))
- [ ] Copy Claude Code rules (see [section 15](#15-claude-code-rules--testing-guide) and [section 16](#16-claude-code-rules--logging-guide))
- [ ] Create LICENSE file (e.g. Apache-2.0, MIT)
- [ ] Create README.md with application description
- [ ] Write application code in `src/`, tests in `src/*.test.ts`
- [ ] Run `bun test`, `bun run test:coverage`, `bun run lint:ci`, and `bun run build` locally
- [ ] Verify `bun run build` produces `dist/app.js` with no errors
- [ ] Verify `bun run start` runs the application correctly
- [ ] Run `bun run changelog` — verify changelog generates without errors
- [ ] Push to `main` — CI runs lint, tests, and build

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

## 16. Claude Code Rules — Logging Guide

The logging rules live at `.claude/rules/logging.rule.md` and `.claude/rules/logging-test.rule.md`, sourced from the shared knowledge base. To copy or update them, run:

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
| Changelog | git-cliff | git-cliff |
| Build scripts | fix-cjs, verify-build, check-version, check-docs, fix-docs-links | verify-build only |