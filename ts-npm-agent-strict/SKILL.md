---
name: ts-npm-agent-strict
description: ts-npm-agent-strict.
---

# TypeScript + npm, tuned for agent-driven development

The primary reader and editor of this codebase is an LLM agent. Configure the toolchain so the agent's mistakes arrive as `tsc` errors, `oxlint` errors, and test failures — the only feedback that appears without anyone asking for it.

Four premises drive every choice below.

- **Every mistake a flag can catch, a flag should catch.** An agent emits plausible-wrong code faster than anyone reviews it. Static checks are the only reviewer that runs on every edit.
- **The program must be legible one file at a time.** Inferred export signatures, ambient globals, and `tsc`-only path aliases are invisible to grep, therefore invisible to an agent that navigates by search. Force them written down.
- **Checker output is a machine channel.** Truncation, ANSI escapes, and volume the agent learns to skim are all corruption in that channel.
- **Prefer a checked element over a comment.** A type, a `satisfies`, an `assertNever`, or a `@ts-expect-error` carries the same claim and fails the build when it stops being true.

Size and shape follow from those: files run as long as their subject, unions as wide as the domain, generics as deep as the type demands, lines to 120 columns. The enforcement is an absence — `max-lines`, `max-depth`, `max-params`, and the complexity rules live in the `style` and `pedantic` categories that `.oxlintrc.json` turns off.

The organizing rule: **every convention here names the tool that enforces it.** A convention nothing enforces belongs in the code as a type, not in a config file.

## Before you configure anything

Five answers pick the route. Ask for whatever is not already evident; do not guess.

1. **Who emits the JS that ships** — `tsc`, a bundler, or a runtime that strips types?
2. **Who resolves that output** — Node, a browser via bundler, or another package's consumers? Independent of (1), and the usual source of broken configs.
3. **Lowest supported runtime** — current or evergreen keeps `target` and `lib` at `esnext`; an older floor pins them down. See the axis note under the route table.
4. **JSX framework**, if any.
5. **Test runner**, and where test files live.

In a monorepo, edit the workspace the user names; ask if it isn't named. Environments without a route below — Deno, Workers, Bun-only, browser-without-bundler — take the strict core and their own answers to (2) and (3), plus the emit block that follows from (1): a config that emits needs `rootDir`, `outDir`, and `noEmitOnError`; one that does not needs `noEmit: true`.

## Toolchain

```bash
npm add -D typescript@^6 oxlint oxlint-tsgolint prettier
```

**Pin TypeScript to 6, not 7.** TypeScript 7 is the Go rewrite and its package ships no JS compiler API, so everything that imports `typescript` stops working: typescript-eslint, ts-morph, and the Volar-based language tooling behind Vue, Svelte, Astro, and MDX.

**Write the config as if it were already TypeScript 7 anyway.** Every option removed in 7 is already an error in 6 — `baseUrl` (TS5101), `moduleResolution: node` (TS5107), `outFile`, `downlevelIteration`, `target: es5`, `module: amd`/`umd`, `esModuleInterop: false`, `allowSyntheticDefaultImports: false` — silenceable only with `ignoreDeprecations`. Never silence them. `oxlint-tsgolint` runs the type-aware lint rules on its own bundled TypeScript 7 engine and never loads your `typescript` package, so a config that still uses removed options type-checks but cannot be type-aware linted. Keeping the config 7-clean is what points both tools at one program, and makes a later compiler bump config-neutral.

**Write out any default whose absence would read as deliberate loosening.** Current members: `strict` and `module` (already true and `esnext` in 6), `allowJs`, `useDefineForClassFields`, and `target` — whose implicit default floats exactly like an explicit `esnext` does, but only the written form reads as chosen rather than forgotten. `rootDir` is not a default at all: an emitting config with sources under `src/` errors TS5011 until you write it.

`--checkers`, `--builders`, and `--singleThreaded` belong to the TypeScript 7 native compiler; passing them to `tsc` 6 fails.

## Standing policy

Override only with a stated reason, and a loosening names the check that takes over the coverage it removes — as [CommonJS](#commonjs-the-one-exception) does.

1. **Correctness** — `strict: true` plus every flag in [What each strict flag buys](#what-each-strict-flag-buys).
2. **Legibility without inference** — `isolatedDeclarations`, `moduleDetection: "force"`, an explicit `types` array, `verbatimModuleSyntax` + `isolatedModules`, `erasableSyntaxOnly`.
3. **Output as a machine channel** — `noErrorTruncation: true`, and `--pretty false` on every agent-facing script.
4. **One check, at full strength, everywhere** — each config covers a different program, and every one runs identically on your machine and in CI.

## The strict core

Identical in every route. Copy it verbatim, then add the route block.

```json
{
  "compilerOptions": {
    "skipLibCheck": true,
    "moduleDetection": "force",
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "useDefineForClassFields": true,
    "resolveJsonModule": true,
    "allowJs": false,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false,
    "noUncheckedSideEffectImports": true,

    "noErrorTruncation": true
  }
}
```

## Pick the route

```
Who emits the JS that ships?
├── A bundler, or a runtime that strips types ─► §1  Bundled app
└── tsc
    ├── You run it (Node, ESM) ────────────────► §2  Node ESM app
    └── Someone else consumes it (npm)
        ├── Single package ────────────────────► §3  Library
        └── Monorepo ──────────────────────────► §4  Project references
```

`module` and `moduleResolution` are co-dependent and come from who resolves the output. Take one row; do not improvise.

| Scenario | `module` | `moduleResolution` |
| --- | --- | --- |
| Bundled app | `preserve` | `bundler` |
| Node.js native ESM | `nodenext` | `nodenext` |
| Library emitted by tsc | `nodenext` | `nodenext` |
| Library emitted by a bundler | `preserve` | `bundler` |
| CommonJS-authored source | `commonjs` | `bundler` |

`target` and `lib` are a separate axis, and the policy is **`esnext` for both, written out**: emit the syntax you wrote and check against the full current standard library — downleveling is the bundler's or the runtime's job, not `tsc`'s. Add `"dom"` to `lib` for browser code. The one stated-reason deviation is a floor below current — an app stuck on an old Node, or a library whose oldest supported consumer lags: pin both to that floor (Node 18 → `es2022`, Node 20 → `es2023`, Node 22 → `es2024`), because a pinned `lib` is then what stands between a green build and a missing built-in at runtime. `esnext` trades that guard away on the premise that the floor is current.

§2 and §3 need `"type": "module"` in `package.json`. Without it `nodenext` reads `.ts` sources as CommonJS and the strict core turns every ESM import into TS1287/TS1295.

**Never use `bundler` for output that `tsc` emits and Node loads.** `bundler` permits extensionless relative imports and `preserve` keeps them verbatim, so `export { x } from "./utils"` compiles clean and dies at `node dist/index.js` with `ERR_MODULE_NOT_FOUND`. Only [running the output](#run-the-output) catches it.

`nodenext` requires an extension on relative imports (TS2835). Either write `.js` yourself, or write `.ts` and set `rewriteRelativeImportExtensions: true`, which rewrites that suffix on emit.

### §1 Bundled app (Vite, Next.js, Rspack, Bun)

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "preserve",
    "moduleResolution": "bundler",
    "lib": ["esnext", "dom"],
    "jsx": "react-jsx",
    "types": [],
    "rootDir": "./src",
    "noEmit": true,
    "allowImportingTsExtensions": true
  },
  "include": ["src"],
  "exclude": ["dist", "build", "coverage"]
}
```

Drop `"dom"` for non-browser code. On `jsx` the framework's own docs win: React and Preact use `react-jsx` (Preact adds `"jsxImportSource": "preact"`), Solid uses `"jsx": "preserve"` with `"jsxImportSource": "solid-js"`. `types: []` fits an app with no ambient globals — otherwise list exactly what it needs (`["node"]`, `["vite/client"]`).

### §2 Node.js ESM application

Match `@types/node` to the Node major you actually run (`npm add -D @types/node@24`): with `lib` at `esnext`, `@types/node` is the one thing tying the typed API surface to a real runtime, and a newer one happily type-checks Node APIs yours lacks.

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["esnext"],
    "types": ["node"],
    "rootDir": "./src",
    "outDir": "./dist",
    "sourceMap": true,
    "noEmitOnError": true
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

`noEmitOnError` keeps a failed check from leaving runnable output that the agent chains into and then debugs at the wrong layer. That `exclude` requires [`tsconfig.test.json`](#5-additional-programs) in the same change.

### §3 Publishable library (emitted by tsc)

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "lib": ["esnext"],
    "types": [],
    "rootDir": "./src",
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "isolatedDeclarations": true,
    "noEmitOnError": true,
    "allowImportingTsExtensions": false
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.spec.ts", "dist"]
}
```

`esnext` holds only while your oldest supported consumer tracks current runtimes; a lagging consumer floor is the axis deviation — pin `target` and `lib` to it. `isolatedDeclarations` lives in the main config here, since a library emits declarations anyway. In `package.json` `exports`, put `types` first in each conditional block, then verify it mechanically with `attw --pack .` and `publint --strict` rather than by eye — a `prepack` script makes npm run both on every `pack` and `publish`.

### §4 Monorepo with project references

Root `tsconfig.json` — `"files": []` stops the root from compiling sources the leaves already build:

```json
{ "files": [], "references": [{ "path": "./packages/core" }, { "path": "./packages/app" }] }
```

`tsconfig.base.json` holds the strict core and uses `${configDir}` so paths resolve against the *extending* config:

```json
{ "compilerOptions": { "rootDir": "${configDir}/src", "outDir": "${configDir}/dist" } }
```

Each leaf extends the base and its own route row — `target` / `lib` / `module` / `types` — and lists in `references` exactly the internal packages it imports. Copying an entry into the package it points at fails `tsc -b` with TS6202 (circular graph).

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "isolatedDeclarations": true
  },
  "include": ["src/**/*"],
  "references": [{ "path": "../core" }]
}
```

`tsc -p` on a solution root does **not** traverse `references`: it checks nothing and exits 0 no matter what is broken in a leaf. Only `tsc -b` walks the graph.

```json
{
  "scripts": {
    "typecheck": "tsc -b --pretty false",
    "clean": "tsc -b --clean"
  }
}
```

Leaves emit their own declarations under `isolatedDeclarations`, so this is the declaration check too, and `check:decl` is dropped from the chain here.

## §5 Additional programs

For §1–§3. Each is a different program, holding files the main config excludes. `extends` merges rather than resets, so restate every value that matters — the blocks below do.

```json
// tsconfig.declarations.json — §1 and §2 only; §3 already emits declarations
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": false,
    "declaration": true,
    "emitDeclarationOnly": true,
    "isolatedDeclarations": true,
    "outDir": "./node_modules/.cache/decl"
  }
}
```

```json
// tsconfig.test.json — required wherever the main config excludes tests
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "noEmit": true, "rootDir": "./", "types": ["node", "vitest/globals"] },
  "include": ["test/**/*"],
  "exclude": ["dist"]
}
```

Three inherited values must be overridden here, not assumed: `include` in an extending config *replaces* the base's, while `exclude` and `rootDir` merge through — left alone, the base's `**/*.test.ts` exclude drops every test file from the program, and files under `test/` error TS6059 for sitting outside the inherited `rootDir: "./src"`. Set `types` to your runner's globals package.

`include` names only `test/**/*`: the sources a test imports are pulled in by module resolution anyway, and the rest are already covered by `typecheck` — under stricter ambient globals, since this program adds the runner's. A source file that reaches for `describe` should fail, and it only does so in the program that lacks it.

## CommonJS: the one exception

`module: "commonjs"` + `moduleResolution: "bundler"` is valid, but `verbatimModuleSyntax` and `erasableSyntaxOnly` are **mutually exclusive** in CommonJS-authored source: verbatim mode requires `import x = require(…)` / `export =`, and `erasableSyntaxOnly` bans exactly that syntax. ESM `export` gives TS1287; `export =` gives TS1294. Pick one:

- **Author ESM, ship CJS (preferred).** Keep §2 or §3 for type-checking and let a bundler produce the CJS artifact. The strict core stays intact.
- **Author CJS with `import =`.** Keep `verbatimModuleSyntax`, set `"erasableSyntaxOnly": false`.
- **Author CJS with ESM syntax and downlevel.** Keep `erasableSyntaxOnly`, set `"verbatimModuleSyntax": false`.

Both CJS-authored routes also need a CommonJS package boundary — a `package.json` without `"type": "module"`, or `.cts` sources emitting `.cjs`. Without one the emit is CommonJS while Node parses the `.js` as ESM and dies at load on `exports`, with `tsc` exiting 0.

## What each strict flag buys

`strict` bundles nine flags, `noImplicitAny` and `strictNullChecks` among them. Every flag below is additional.

**Catches wrong code**

- **`noUncheckedIndexedAccess`** — `arr[i]` and `obj[k]` become `T | undefined`. The highest-yield flag against agent-written code. Fixing the resulting errors adds branches, so run the tests.
- **`noPropertyAccessFromIndexSignature`** — index-signature properties must be read as `obj["k"]`. Together with the above, "declared property" and "dynamic lookup" become syntactically distinct, so a hallucinated property name is a compile error instead of `undefined` at runtime.
- **`exactOptionalPropertyTypes`** — `{ a: undefined }` and `{}` stop being interchangeable.
- **`noImplicitOverride`** — a renamed base method becomes an error, not a silently orphaned override.
- **`noImplicitReturns`** / **`noFallthroughCasesInSwitch`** / **`allowUnreachableCode: false`** / **`allowUnusedLabels: false`** — control-flow slips become errors rather than editor-only hints.
- **`noUnusedLocals` / `noUnusedParameters`** — catches half-applied refactors, a characteristic agent failure. Prefix a genuinely-unused interface parameter with `_` rather than disabling the flag.
- **`noUncheckedSideEffectImports`** — a typo'd `import "./setup"` is otherwise silently ignored.

**Legible without inference**

- **`isolatedDeclarations`** — an export whose declaration cannot be produced from its own file must be annotated; trivial cases still infer (`export const n = 1`). It removes cross-file inference at the module boundary, so an agent reads the public API from one file. Needs `declaration` or `composite` (TS5069).
- **`moduleDetection: "force"`** — every file is a module; removes the case where a file with no imports quietly shares global scope.
- **`types: [...]`** — every ambient global has one declared source. Otherwise `describe` or `process` appears from nowhere and no grep explains it.
- **`verbatimModuleSyntax`** — type-only imports must say `import type`.
- **`erasableSyntaxOnly`** — bans `enum`, runtime `namespace`, parameter properties, `import =`/`export =`, `<T>x`. Each hides runtime behavior behind type-looking syntax. Replacements for `enum` (`as const` object, literal union) differ in runtime shape, so pick deliberately and test. Frameworks built on parameter properties (NestJS constructor injection, TypeORM) cannot satisfy it: drop the flag for that workspace and put back what the linter can still carry — `"typescript/no-namespace": "error"` for runtime namespaces. Parameter properties are the point of the exemption, and no oxlint rule bans `enum` declarations, so that half of the flag has no replacement: say so where the flag is dropped rather than implying it is covered.
- **`resolveJsonModule`** — an imported `.json` gets a real type instead of `any`.
- **`useDefineForClassFields`** — implicitly true at `target` ≥ ES2022, but it changes field-initialization semantics, so pin it.
- **`allowJs: false`** — keep untyped JS out of the program.

**Consumable output**

- **`noErrorTruncation`** — `tsc` otherwise truncates long types with `... N more ...`, and an agent handed a truncated type cannot fix the error, so it guesses.
- **`--pretty false`** on every agent-facing script — one error per line, no ANSI, greppable.
- **`skipLibCheck: true`** — a broken `.d.ts` in a dependency is not a mistake any edit to `src/` can fix, so reporting it on every run spends the channel on something the agent cannot act on.

## oxlint, the only linter

The linter's consumer is the agent: error-level findings are real bugs, style is the formatter's problem, and the output is machine-parseable.

`.oxlintrc.json`:

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["typescript", "import", "promise", "node"],
  "categories": {
    "correctness": "error",
    "suspicious": "warn",
    "perf": "warn",
    "style": "off",
    "pedantic": "off"
  },
  "rules": {
    "no-console": "off"
  },
  "ignorePatterns": ["node_modules", "dist", "build", "coverage", "*.min.js"]
}
```

- **`$schema`** — the config itself is machine-verified: a typo'd key or a wrong value type is flagged by any schema-aware editor or validator instead of being silently ignored.
- **`plugins` overwrites the default set rather than extending it** — `typescript`, `import`, `promise`, `node` cover the TS/Node project surface.
- **`correctness: error` versus `suspicious`/`perf: warn`** grades confidence, not consequence. Every run denies warnings, so both block; the severity tells the agent which findings are certainly wrong and which are heuristics worth reading before obeying. `style` and `pedantic` stay off: style is the formatter's job, and a lint pass whose output is mostly taste teaches the agent to skim the channel.
- **`no-console: "off"`** — agent debugging routinely inserts `console.log`, and flagging it slows the inner loop. Strip it at release time with a deliberate pass, not on every lint.
- **`ignorePatterns` lives in the config, not on script flags** — the CLI, CI, and the editor then apply the same set.

This config is the untyped rule set. The type-aware pass — `oxlint --type-aware --type-check`, which needs `oxlint-tsgolint` — adds `typescript/no-floating-promises` and the `no-unsafe-*` family: an `any` crossing a boundary and a promise nobody awaited both type-check cleanly, and untyped lint misses them too, so run the pass as a deliberate gate alongside the checks that stay outside `check`. `oxlint-tsgolint` discovers each file's `tsconfig.json` itself; do not pass `--tsconfig`, which type-aware linting ignores.

## Prettier

Prettier's defaults reflow large regions when an agent edits one line, which shifts surrounding line numbers and invalidates the exact-string matches the agent is holding for its next edit in the same session. These settings keep a one-line edit a one-line diff.

`.prettierrc.json`:

```json
{
  "printWidth": 120,
  "objectWrap": "preserve",
  "trailingComma": "all",
  "arrowParens": "always",
  "semi": true,
  "endOfLine": "lf"
}
```

- **`printWidth: 120`** — a one-line edit rarely triggers auto-wrap, so surrounding line counts stay stable.
- **`objectWrap: "preserve"`** — an object keeps the shape it was written in instead of collapsing or exploding when one property changes.
- **`trailingComma: "all"`** — appending an item is one added line, not a two-line diff that also rewrites the previous line.
- **`arrowParens: "always"`** — adding a parameter type annotation does not also force inserting parens.
- **`semi: true`** — ASI cannot silently change behavior when an agent prepends a line starting with `[`, `(`, `+`, `-`, or `/`.
- **`endOfLine: "lf"`** — no whole-file CRLF churn on mixed-OS checkouts.

`.prettierignore`:

```
node_modules
dist
build
coverage
*.min.js
package-lock.json
*.md
```

`*.md` is there because Prettier reflows paragraphs, list spacing, and table widths, which breaks the same exact-string matching.

## Scripts

Single project (§1–§3). Monorepos use the [§4](#4-monorepo-with-project-references) build script in place of `typecheck` and drop `check:decl`.

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit --pretty false",
    "typecheck:test": "tsc -p tsconfig.test.json --pretty false",
    "check:decl": "tsc -p tsconfig.declarations.json --pretty false",
    "lint": "oxlint --deny-warnings --format agent",
    "lint:fix": "oxlint --fix --deny-warnings --format agent",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "check": "npm run typecheck && npm run typecheck:test && npm run check:decl && npm run lint && npm run format:check && npm test"
  }
}
```

CI runs `npm run check` — the same command you run locally, so a green run on your machine is evidence about CI. The set of checks is identical everywhere; only the rendering may differ, so a CI step is free to re-render `lint` output as GitHub annotations by dropping `--format agent`.

`--format agent` is oxlint's one-line-per-diagnostic format. `--deny-warnings` on both lint scripts makes every finding something the agent has to resolve. `--fix` applies behavior-preserving fixes; `--fix-suggestions` and `--fix-dangerously` can change behavior, so run them deliberately and read the diff. `check:decl` exists wherever `tsconfig.declarations.json` does; pointing it at a §3 project is a TS5058 missing-file error. Drop `npm test` until a runner exists.

Outside `check` stay the [type-aware lint pass](#oxlint-the-only-linter) and two checks that need artifacts it does not build: [running the output](#run-the-output), and — for libraries — `attw --pack .` with `publint --strict` in `prepack`. `type-coverage --at-least 99 --strict` is optional, and puts a number in CI in place of a belief about leftover `any`.

## Run the output

`tsc` exiting 0 proves the types check; only running the emitted JS proves it loads. The gap opens wherever the compiler's resolver is more permissive than the runtime's: `bundler` resolution with Node-loaded output, extensionless relative imports, an `exports` map whose `types` condition points somewhere the runtime does not.

Every emitting route therefore ends in an execution, not a compile:

- §2 — `node dist/index.js` after a clean build.
- §3 — `npm pack`, install the tarball into a scratch consumer, import it under both `node` and a `nodenext` `tsc`.
- §4 — `tsc -b`, then run the package that declares the `references` edge, so its emitted import of the dependency resolves for real.
- CommonJS — run the emitted artifact under the package's real `type`.

## Path aliases

For anything Node loads, plain relative `./thing.js` imports are the default. An alias adds a second resolver that has to agree with the first, and nothing checks that it does.

When an alias is genuinely wanted, use `package.json` `imports` rather than tsconfig `paths`: Node resolves `imports` itself and `tsc` honors it under both `nodenext` and `bundler`, so there is one enforced source of truth.

```json
{ "type": "module", "imports": { "#src/*": "./dist/*" } }
```

Map to the **output**, not the source: a package that runs from `dist/` and maps to `./src/*` resolves at runtime into `src/`, where no `.js` files exist. For `noEmit` bundled apps the source mapping is fine, because the bundler resolves before anything runs. Use a segment prefix like `#src/*`; the bare `#/` prefix needs a recent Node and otherwise throws `ERR_INVALID_MODULE_SPECIFIER` while `tsc` exits 0.

## Adopting this in an existing project

Do not overwrite. Surface the diff against the matching route, flag standing-policy violations, let the user decide. Order by blast radius, and put anything that makes errors readable before the steps that produce many — each step a separate reviewable change with `npm run typecheck` green before the next. Propose the commits rather than making them unasked.

1. **Deprecated options** — remove every option removed in TypeScript 7; nothing else is evaluable until the config loads clean.
2. **`strict: true`**, explicit `types` and `rootDir`, an explicit `target` / `lib` (`esnext` unless a floor pins them), and the `module` / `moduleResolution` row that matches who resolves the output.
3. **`noErrorTruncation` + `--pretty false`** — before the noisy steps, so their errors arrive readable.
4. **Zero-behavior-change flags**, in one or more commits: `noUnusedLocals`, `noUnusedParameters`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `noImplicitOverride`, `allowUnreachableCode: false`, `allowUnusedLabels: false`, `noUncheckedSideEffectImports`.
5. **`noUncheckedIndexedAccess`** — the largest error count, and it adds branches.
6. `noPropertyAccessFromIndexSignature`, then `exactOptionalPropertyTypes`.
7. `verbatimModuleSyntax` + `isolatedModules` + `moduleDetection: force`.
8. **`erasableSyntaxOnly`** — mostly `enum` migration, so runtime shape changes here.
9. Split out the [§5 programs](#5-additional-programs), `tsconfig.test.json` included.
10. oxlint: adopt the config above and clear `correctness`; bring the type-aware pass in once untyped lint is green.
11. **`isolatedDeclarations`** last — the largest diff, but almost entirely additive annotations.

## When something "isn't working"

```bash
npx tsc --showConfig      # merged config after all `extends` — run this before guessing
npx tsc --explainFiles    # why each file is in the program
npx tsc --listFilesOnly   # what the program actually contains
npx oxlint --print-config path/to/file.ts   # the rules actually applied to one file
```

`--showConfig` prints merged explicit values, not defaults. On a solution root, `--listFilesOnly` returning nothing is the expected symptom of the §4 traversal problem, not an empty project.

Confirm the setup is live rather than assuming it:

- `const x: string = arr[0]` must error — `noUncheckedIndexedAccess`.
- A long **inline anonymous** union must print every member instead of `... N more ...` — `noErrorTruncation`. A named alias prints as its own name and never truncates, so it produces a false pass.
- A bare `somePromise()` statement must fail the type-aware pass (`oxlint --type-aware --type-check`) — the rules are running against a 7-clean config, not silently skipped.
- On a monorepo, a deliberate type error in one leaf must make `typecheck` exit non-zero.
