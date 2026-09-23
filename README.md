# Prompt 6 Generate Shared TypeScript Types

This README explains, step by step, what was done in `packages/contract` to
turn the OpenAPI spec into TypeScript types shared by both `apps/backend`
and `apps/frontend`.

## Step 1 — Start from the spec

`packages/contract/openapi.yaml` is the single source of truth for the API.
It describes every route (`/api/work-orders`, `/api/assets`, ...) and every
schema (`WorkOrder`, `Asset`, `Technician`, ...) using standard OpenAPI 3.0
syntax. Nothing in the contract package is written by hand in TypeScript —
it is all derived from this one file.

## Step 2 — Scaffold the package

`packages/contract/package.json` was set up so that:

- The package is named `@equipment-hub/contract`, matching the naming
  pattern of the other workspaces (`apps/backend`, `apps/frontend`).
- `exports["."]` points **directly at `src/index.ts`** — no separate build
  step:

  ```json
  {
    "exports": {
      ".": "./src/index.ts"
    }
  }
  ```

  This works because `tsconfig.base.json` uses `"moduleResolution": "bundler"`,
  which lets consumers import `.ts` source files straight from a workspace
  dependency. There is no `tsc` compile, no `dist/` folder, and no watch
  process to keep in sync.

## Step 3 — Install the generator

[`openapi-typescript`](https://openapi-ts.dev/) was added as a
`devDependency` of `packages/contract` (it is only needed to *generate*
types, never at runtime):

```bash
npm install -D openapi-typescript --workspace=@equipment-hub/contract
```

## Step 4 — Add the `gen` script

A `gen` script was added to `packages/contract/package.json` that runs the
generator against the spec and writes the output to
`packages/contract/src/types.gen.ts`:

```json
"scripts": {
  "gen": "openapi-typescript openapi.yaml -o src/types.gen.ts"
}
```

To (re)generate the types, the exact command is:

```bash
npm run gen --workspace=@equipment-hub/contract
```

`packages/contract/src/types.gen.ts` is auto-generated and starts with a
header saying so. It contains a big set of nested `interface`/`type`
declarations that mirror the spec almost literally: a
`components["schemas"]` namespace holding every schema (`Asset`,
`WorkOrder`, ...), plus a `paths`/`operations` namespace describing every
route. It is correct, but not pleasant to import directly — nobody wants to
write `components["schemas"]["WorkOrder"]` all over the app.

## Step 5 — Add friendly type aliases

That is what `packages/contract/src/index.ts` is for. It is the package's
*public* entry point (the thing `exports["."]` points to), and all it does
is re-export short, readable names for the schemas the apps actually need:

```ts
import type { components } from "./types.gen.js";

export type WorkOrderState = components["schemas"]["WorkOrderState"];
export type WorkOrderAction = components["schemas"]["WorkOrderAction"];
export type Priority = components["schemas"]["Priority"];
export type Asset = components["schemas"]["Asset"];
export type Technician = components["schemas"]["Technician"];
export type WorkOrder = components["schemas"]["WorkOrder"];
export type NewWorkOrder = components["schemas"]["NewWorkOrder"];
export type TransitionCommand = components["schemas"]["TransitionCommand"];
export type AssignmentCommand = components["schemas"]["AssignmentCommand"];
export type DashboardSummary = components["schemas"]["DashboardSummary"];
export type ApiError = components["schemas"]["ApiError"];
```

Now `apps/backend` and `apps/frontend` can both do:

```ts
import type { WorkOrder, NewWorkOrder } from "@equipment-hub/contract";
```

Neither app has to hand-write its own copy of these types — they always
come from this one package, so backend and frontend can never silently
drift apart.

## Step 6 — Verify it compiles

A minimal `packages/contract/tsconfig.json` (extending the repo's shared
`tsconfig.base.json`, with `noEmit: true` since this package never builds
to JS) confirms `src/index.ts` type-checks cleanly:

```bash
npx tsc -p packages/contract/tsconfig.json --noEmit
```

## Why generated types should never be edited by hand

`packages/contract/src/types.gen.ts` is a **derived artifact**, not source
code — its only source of truth is `packages/contract/openapi.yaml`.
Editing it by hand causes real problems:

1. **It gets overwritten.** The next `npm run gen` regenerates the file from
   the spec and silently discards any manual edit. The "fix" disappears the
   next time someone runs the generator.
2. **It desyncs the type from reality.** A hand-edited type makes
   TypeScript believe the API returns or accepts a shape it does not
   actually use — which defeats the entire reason this package exists: to
   guarantee the backend and frontend agree on the contract.
3. **It hides real spec bugs.** If a generated type looks wrong, that means
   `openapi.yaml` is wrong (or misconfigured), not that the generated file
   is wrong. Patching the output treats the symptom and buries the real
   problem.
4. **It breaks the single-source-of-truth workflow.** The whole pipeline —
   `openapi.yaml` → `openapi-typescript` → `types.gen.ts` → `index.ts` →
   apps — exists so there is exactly one place to make a change. Hand-editing
   generated output creates a second, competing source of truth that will
   eventually contradict the first.

**Rule of thumb:** if a type needs to change, edit
`packages/contract/openapi.yaml`, then run
`npm run gen --workspace=@equipment-hub/contract`. Never edit
`packages/contract/src/types.gen.ts` directly.
