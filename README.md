# kb

Actionable guides, templates, and reference notes for `@pencroff-lab` projects.

## Structure

```
kb/
├── playbook/   # Step-by-step templates for common workflows
└── rules/      # Coding conventions and best practices
```

### Playbooks

Playbooks are executable guides. Reference them in agent prompts:

```
Execute @pencroff-lab/kb/playbook/setup-ts-lib.template.md
```

| Playbook | Description |
|----------|-------------|
| [setup-ts-lib.template](playbook/setup-ts-lib.template.md) | TypeScript library setup and npm publish workflow (dual ESM/CJS, CI/CD) |
| [setup-ts-app.template](playbook/setup-ts-app.template.md) | TypeScript bundled application setup with Bun (single bundle, CI/CD) |

### Rules

Rules are conventions applied during development. They can be used as agent rules with `globs` and `alwaysApply` front matter.

| Rule | Description |
|------|-------------|
| [testing](rules/testing.rule.md) | Testing patterns, mocking conventions, and strategies (bun test + Sinon) |
| [logging](rules/logging.rule.md) | Logging patterns, call signatures, error logging strategy, and naming conventions |
| [logging-test](rules/logging-test.rule.md) | Logger mocking and assertion patterns for tests |

## Usage

Reference a playbook or rule by its path within the `@pencroff-lab/kb` repo:

- **Playbook** — `@pencroff-lab/kb/playbook/<name>.md` — follow as a step-by-step guide
- **Rule** — `@pencroff-lab/kb/rules/<name>.md` — apply as coding conventions

## License

Apache-2.0
