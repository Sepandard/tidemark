# tidemark — Roadmap for AI Coding Agents

> This file is the source of truth for building **tidemark**. Read it fully before writing code.
> Work **one phase at a time**, in order. Do not start a phase until every "Done when" item of the previous phase is true.

---

## 1. What tidemark is

tidemark is an open-source CLI (and later an Nx plugin, GitHub Action and MCP server) that **finds code that bypasses a design system** in Angular and React projects.

Example input:

```tsx
<button style={{ background: '#1a73e8' }}>Pay</button>
```

Example output:

```
apps/wallet/src/pay.tsx:12:3   error  raw <button>       → use <Button> from @demo/ui    raw-element
apps/wallet/src/pay.tsx:12:20  warn   hardcoded #1a73e8  → use token --color-primary     hardcoded-color

2 problems (1 error, 1 warning) · design-system adoption: 82%
```

Think of it as ESLint, but for design-system adoption instead of code bugs.

### Positioning (why this project exists)

A similar npm package, `ds-drift`, already exists. It supports React/TS/JS only, with no Angular, Nx, SARIF or MCP support. tidemark must be different from day one:

1. **Angular templates are first-class**, shipped in v0.1 together with React.
2. **Nx-native**: scores per Nx project, runs on affected projects only.
3. **PR gating, not dashboards**: baseline mode, SARIF, GitHub Action.
4. **Agent-facing**: MCP server so AI agents can check their own UI code.

Never copy code from `ds-drift` or any other project. Read for ideas only.

---

## 2. Rules for the agent

- **Scope discipline.** Build only what the current phase lists. If you think something is missing, add it to `## 9. Parking lot` instead of building it.
- **Tests first.** Every rule gets fixture files with known violations and a Vitest test before or alongside the implementation.
- **No false positives over full coverage.** When code cannot be analyzed statically (dynamic `className`, `style={getStyle()}`, spread props), skip it. Never guess.
- **`core` has no I/O assumptions.** `core` takes file contents and config, returns findings. Reading the file system, printing and exit codes belong in `cli`.
- **Parse each file once.** Run all rules on the same AST. Never re-parse per rule.
- **Small commits** using Conventional Commits (`feat(core): ...`, `fix(cli): ...`, `test(rules): ...`).
- **Do not** publish to npm, create GitHub releases, push tags or change repository settings. A human does that.
- **Do not** add dependencies beyond those listed here without writing why in the PR or commit message.
- **Stop and ask the human** about anything in `## 8. Open decisions`.
- Keep the README honest: never document a feature that is not implemented. Planned features go under a "Roadmap" heading.

---

## 3. Tech stack

| Concern | Choice |
|---|---|
| Monorepo | Nx + pnpm workspaces |
| Language | TypeScript, `strict: true`, ESM |
| Node | ≥ 20 |
| JSX/TSX parsing | `@babel/parser` + `@babel/traverse` (plugins: `jsx`, `typescript`) |
| Angular templates | `parseTemplate` from `@angular/compiler` |
| Styles (v0.2+) | `postcss`, `postcss-scss` |
| CLI args | `cac` |
| File globbing | `fast-glob` |
| Config loading | `jiti` (to load `tidemark.config.ts`) |
| Config validation | `zod` |
| Terminal output | `picocolors` |
| Build | `tsup` |
| Tests | Vitest |
| Versioning | Changesets |
| CI | GitHub Actions |

Later phases may swap Babel for `oxc-parser` for speed. Keep parsing behind an interface so that swap only touches the parser layer.

---

## 4. Repository layout

```
tidemark/
├── packages/
│   ├── core/            # parsers, rules, scoring, types (no fs, no console)
│   ├── cli/             # `tidemark` binary — published package name: `tidemark`
│   ├── nx-plugin/       # phase 4
│   ├── github-action/   # phase 3
│   └── mcp/             # phase 5
├── examples/
│   ├── demo-ui/         # fake design system: Button, Input, Card + tokens
│   ├── react-app/       # sample React app with intentional drift
│   └── angular-app/     # sample Angular app with intentional drift
├── fixtures/            # tiny files used by unit tests, one folder per rule
├── docs/
├── ROADMAP.md
├── AGENTS.md            # short pointer to this file + coding conventions
└── README.md
```

Until the human decides on npm scopes (see Open decisions), only `packages/cli` is publishable, as `tidemark`, with `core` bundled into it by `tsup`. All other packages are `"private": true`.

---

## 5. Core contracts

Implement these first. Other code depends on them.

### 5.1 Config (`tidemark.config.ts`)

```ts
import { defineConfig } from 'tidemark';

export default defineConfig({
  designSystem: {
    packages: ['@demo/ui'],          // imports from here count as on-system
    allowedPaths: ['libs/shared/ui'], // local re-export barrels that also count
  },
  include: ['apps/**/*.{tsx,jsx,ts,html}'],
  exclude: ['**/*.spec.*', '**/node_modules/**'],
  replacements: {
    react:   { button: 'Button', input: 'Input' },
    angular: { button: 'ds-button', input: 'ds-input' },
  },
  tokens: {
    colors: { '--color-primary': '#1a73e8', '--color-danger': '#d93025' },
  },
  rules: {
    'raw-element': 'error',
    'hardcoded-color': 'warn',
    'deprecated-component': 'off',
  },
});
```

Validate with `zod`. Invalid config → clear error message naming the bad key, exit code 2.

### 5.2 Types

```ts
export type Severity = 'error' | 'warn' | 'off';
export type Language = 'jsx' | 'angular-template' | 'style';

export interface Finding {
  ruleId: string;
  file: string;          // path relative to cwd
  line: number;          // 1-based
  column: number;        // 1-based
  message: string;
  suggestion?: string;   // e.g. "use <Button> from @demo/ui"
  severity: Exclude<Severity, 'off'>;
}

export interface RuleContext {
  file: string;
  language: Language;
  ast: unknown;          // narrowed per language inside the rule
  source: string;
  config: ResolvedConfig;
  report(finding: Omit<Finding, 'ruleId' | 'file' | 'severity'>): void;
}

export interface Rule {
  id: string;
  languages: Language[];
  description: string;
  check(ctx: RuleContext): void;
}
```

### 5.3 Pipeline

```
resolve config → glob files → for each file: detect language → parse ONCE
→ run every enabled rule whose `languages` matches → collect findings
→ apply ignore comments → compute score → reporter
```

### 5.4 Score

```
adoption = onSystemUsages / (onSystemUsages + offSystemUsages)
```

- `onSystemUsages`: elements/components imported from a design-system package or allowed path, or Angular elements whose tag is a configured design-system selector.
- `offSystemUsages`: findings from `raw-element` (and later `shadow-component`).
- Report as a whole-number percentage. If the denominator is 0, report `n/a`.

### 5.5 Ignore comments

- JSX/TS: `// tidemark-ignore-next-line` and `{/* tidemark-ignore-next-line */}`
- HTML: `<!-- tidemark-ignore-next-line -->`
- Optional rule id: `// tidemark-ignore-next-line raw-element`

---

## 6. Phases

### Phase 0 — Scaffold (target: 1 week)

- [ ] Nx + pnpm workspace with `packages/core` and `packages/cli`
- [ ] TypeScript strict, ESLint, Prettier, Vitest configured
- [ ] `examples/demo-ui`: React `Button`, `Input`, `Card` and Angular `ds-button`, `ds-input`, `ds-card` (minimal implementations are fine)
- [ ] `examples/react-app` and `examples/angular-app` with a few intentional violations
- [ ] GitHub Actions CI: install, lint, typecheck, test on every PR
- [ ] `AGENTS.md` pointing to this file
- [ ] README skeleton: one-line pitch, "status: early development", roadmap section

**Done when:** `pnpm nx run-many -t lint test build` passes in CI on an empty `core` and `cli`.

---

### Phase 1 — v0.1: React + Angular core (target: 3–4 weeks)

This is the first public release. Angular support is part of it on purpose.

**Core**
- [ ] Config loading (`jiti`) and validation (`zod`), `defineConfig` helper
- [ ] Language detection by extension: `.tsx/.jsx` → jsx; `.html` → angular-template; `.ts` → check for `@Component` with inline `template`
- [ ] JSX parser adapter (Babel)
- [ ] Angular parser adapter: `.html` files and inline `template:` strings extracted from `@Component` decorators. Map template line/column back to the `.ts` file for inline templates.
- [ ] Rule runner, finding collection, ignore comments, scoring

**Rules**
- [ ] `raw-element` (jsx): flags `JSXOpeningElement` whose name is a lowercase key in `replacements.react`. Does not flag elements inside files that belong to a design-system package.
- [ ] `raw-element` (angular-template): flags elements whose tag is a key in `replacements.angular`.
- [ ] Import awareness (jsx): a capitalized component counts as on-system only when its binding resolves (via Babel scope) to an import from `designSystem.packages` or `allowedPaths`. Handle aliases (`import { Button as Btn }`) and namespace imports (`import * as UI`).
- [ ] `hardcoded-color` (jsx): hex (`#rgb`, `#rrggbb`, `#rrggbbaa`), `rgb()`, `rgba()`, `hsl()` inside `style` attribute objects and string literals passed to `style`. If the value exactly matches a token, the suggestion names that token.
- [ ] `hardcoded-color` (angular-template): same checks inside `style="..."` and `[style.*]` static values.

**CLI**
- [ ] `tidemark scan [paths...]` with `--config`, `--format pretty|json`, `--max-warnings <n>`
- [ ] `tidemark init` writes a starter `tidemark.config.ts`
- [ ] Exit codes: `0` clean, `1` errors found (or warnings above `--max-warnings`), `2` config or runtime error
- [ ] Pretty output grouped by file, summary line with counts and adoption %

**Quality**
- [ ] Fixtures + tests for every rule, both positive (should flag) and negative (should not flag) cases
- [ ] Tests for aliases, namespace imports, ignore comments, inline Angular templates
- [ ] Running `tidemark scan` on `examples/react-app` and `examples/angular-app` produces the expected findings (snapshot test)
- [ ] README: install, quick start, config reference, rule list, example output, comparison note ("supports Angular templates and Nx")

**Done when:** a human can run `npx tidemark scan` in both example apps and get correct output, all tests pass, and the README matches real behavior. The human then publishes `0.1.0`.

---

### Phase 2 — v0.2: Styles, tokens, Tailwind (target: 2–3 weeks)

- [ ] Style parser adapter: `.css`, `.scss` via PostCSS; Angular `styles: [...]` / `styleUrl(s)`
- [ ] `hardcoded-color` (style): flags color literals in declarations; ignores values inside `var(...)`
- [ ] Token sources: inline object (existing), W3C Design Tokens JSON file, or a CSS file of custom properties
- [ ] Near-match suggestions: if a hardcoded color is perceptually close to a token (use a simple ΔE threshold, configurable), suggest that token
- [ ] Tailwind arbitrary values: flag `bg-[#...]`, `text-[#...]`, etc. in static `className` / `class` strings
- [ ] Adoption score per top-level folder (`--group-by folder`)

**Done when:** style files and Tailwind classes are covered by tests and documented.

---

### Phase 3 — v0.3: CI integration (target: 2 weeks)

- [ ] **Baseline mode:** `tidemark baseline` writes `.tidemark-baseline.json`; `tidemark scan --baseline` reports only findings not in the baseline. Match findings by rule + file + a stable fingerprint of the code snippet, not by line number.
- [ ] `--format sarif` (SARIF 2.1.0) so GitHub shows findings inline on PRs
- [ ] `packages/github-action`: runs tidemark, uploads SARIF, posts or updates one PR comment with adoption change vs. base branch and new findings count
- [ ] Docs page: "Adopting tidemark in an existing codebase"

**Done when:** the action runs on a test PR in this repo and posts a correct comment.

---

### Phase 4 — v0.4: Nx plugin (target: 1–2 weeks)

- [ ] `packages/nx-plugin` with an inferred `tidemark` target for every project that has matching files
- [ ] Results cached by Nx (declare correct inputs: source files, config, tokens)
- [ ] `nx affected -t tidemark` works
- [ ] Adoption score per Nx project in the summary

**Done when:** an Nx workspace example shows per-project scores and cache hits on a second run.

---

### Phase 5 — v0.5: MCP server (target: 1–2 weeks)

- [ ] `packages/mcp` using the official MCP TypeScript SDK, stdio transport
- [ ] Tools:
  - `check_snippet({ code, language })` → findings for an unsaved code snippet
  - `find_replacement({ element, framework })` → the design-system component to use, with import path
  - `get_project_score({ path })` → adoption % and top offenders
- [ ] README section showing an agent fixing its own off-system code

**Done when:** the server works in at least one MCP-capable agent with documented setup steps.

---

### Phase 6 — v0.6: Autofix (later)

- [ ] `tidemark scan --fix` for safe cases only: replace `<button>` with the configured component and add the import when the replacement accepts the same props
- [ ] Never autofix when attributes are dynamic or unknown
- [ ] Dry-run diff output (`--fix-dry-run`)

---

## 7. Known pitfalls

| Pitfall | Handling |
|---|---|
| A raw `<button>` inside the design system's own `Button` | Exclude files resolved inside `designSystem.packages` / `allowedPaths` from `raw-element` |
| Local re-export barrels (`libs/ui/index.ts`) | `designSystem.allowedPaths` |
| Dynamic values (`cx(a, b)`, `style={fn()}`, `{...props}`) | Skip silently |
| Inline Angular templates | Extract string, parse, map positions back to the `.ts` file |
| Huge monorepos | Parse once per file; later: worker threads or `oxc-parser` |
| Line-number baselines break on every edit | Fingerprint findings by snippet, not line |
| SVG/canvas/email templates using raw elements legitimately | Users add them to `exclude` |

---

## 8. Open decisions (ask the human)

- npm scope: publish extra packages as `@tidemark/*` (requires npm org) or `tidemark-*`?
- License: MIT (default unless told otherwise)
- Minimum supported Angular version for template parsing
- Default severity for `hardcoded-color`: `warn` or `error`?
- Whether to support Vue/Svelte at all (current answer: no, not before v1.0)

---

## 9. Parking lot

Ideas that are explicitly **out of scope** until v0.6 is done. Add new ideas here instead of building them.

- `shadow-component` rule (local components duplicating design-system ones)
- `deprecated-component` / deprecated props rule
- Off-scale spacing and font-size detection
- HTML report / dashboard
- Trend tracking over time
- Vue / Svelte support
- VS Code extension
