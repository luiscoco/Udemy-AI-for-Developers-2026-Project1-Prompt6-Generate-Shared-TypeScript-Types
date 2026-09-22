# Generate Shared TypeScript Types

This package turns `openapi.yaml` — the single source of truth for the API —
into TypeScript types that both `apps/backend` and `apps/frontend` import.
Neither app hand-writes its own copies of `WorkOrder`, `Asset`, etc.; they all
come from here, so backend and frontend can never silently drift apart.

This README walks through exactly how the package was built, step by step,
so you can reproduce (or extend) the same setup yourself.

## Running the app (Windows terminal)

At this stage of the course, `apps/backend` and `apps/frontend` are still
empty scaffolds (just a `package.json` + `tsconfig.json`, no server or UI
code yet), and the root `package.json`'s `dev` script is a placeholder:

```json
"dev": "echo \"no dev server configured yet\" && exit 0"
```

So there is no Fastify server or Vite dev server to start yet — running
`npm run dev` right now will just print that placeholder message. The one
command that's actually functional at this point in the course is
regenerating the shared types from `packages/contract`. From the repo root,
in PowerShell or Command Prompt:

```powershell
npm run gen --workspace=@equipment-hub/contract
```

Once the backend and frontend are implemented in a later step, this section
will be updated with the real `npm run dev` command to launch both.

## Step 1 — Start from the spec

`openapi.yaml` already existed before this package did. It describes every
route (`/api/work-orders`, `/api/assets`, ...) and every schema
(`WorkOrder`, `Asset`, `Technician`, ...) using standard OpenAPI 3.0 syntax.
Nothing here was written by hand in TypeScript — it's all derived from this
one file.

## Step 2 — Scaffold the package

A workspace package needs a `package.json` that:

- Names the package `@equipment-hub/contract`, matching the pattern the
  other workspaces already use (`@equipment-hub/backend`,
  `@equipment-hub/frontend`).
- Points `exports["."]` **directly at `src/index.ts`**, with no build step:

  ```json
  {
    "exports": {
      ".": "./src/index.ts"
    }
  }
  ```

  This works because `tsconfig.base.json` uses `"moduleResolution": "bundler"`,
  which lets consumers import `.ts` source files straight from a workspace
  dependency. There's no `tsc` compile, no `dist/` folder, no watch process —
  one less moving part to keep in sync.

- Declares a `gen` script that regenerates the types from the spec.

## Step 3 — Install the generator

We used [`openapi-typescript`](https://openapi-ts.dev/), added as a
`devDependency` (it's only needed to *generate* types, never at runtime):

```bash
npm install -D openapi-typescript --workspace=@equipment-hub/contract
```

## Step 4 — Generate the types

The `gen` script runs the generator against the spec and writes the output
to `src/types.gen.ts`:

```json
"scripts": {
  "gen": "openapi-typescript openapi.yaml -o src/types.gen.ts"
}
```

To (re)generate the types, run:

```bash
npm run gen --workspace=@equipment-hub/contract
```

`src/types.gen.ts` is auto-generated and starts with a header saying so.
**Never edit it by hand** — see [Why generated types are off-limits](#why-generated-types-are-off-limits)
below.

The output is a big file of nested `interface`/`type` declarations that
mirror the spec almost literally: a `components["schemas"]` namespace holding
every schema (`Asset`, `WorkOrder`, ...), and a `paths`/`operations`
namespace describing every route, its parameters, and its response shapes.
It's correct, but it's not pleasant to import directly — nobody wants to
write `components["schemas"]["WorkOrder"]` all over the app.

## Step 5 — Add friendly aliases

That's what `src/index.ts` is for. It's the package's *public* entry point
(the thing `exports["."]` points to), and all it does is re-export short,
readable names for the schemas apps actually need:

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

Now `apps/backend` and `apps/frontend` just do:

```ts
import type { WorkOrder, NewWorkOrder } from "@equipment-hub/contract";
```

## Step 6 — Verify it compiles

A minimal `tsconfig.json` (extending the repo's shared
`tsconfig.base.json`, with `noEmit: true` since this package never builds
to JS) confirms `src/index.ts` type-checks cleanly:

```bash
npx tsc -p packages/contract/tsconfig.json --noEmit
```

## Why generated types are off-limits

`src/types.gen.ts` is a **derived artifact**, not source code — its only
source of truth is `openapi.yaml`. Editing it by hand causes real problems:

1. **It gets overwritten.** The next `npm run gen` regenerates the file from
   the spec and throws away any manual edit without warning. The bug you
   "fixed" comes right back.
2. **It desyncs the type from reality.** A hand-edited type makes
   TypeScript believe the API returns or accepts a shape it doesn't
   actually use — which defeats the entire reason this package exists: to
   guarantee the backend and frontend agree on the contract.
3. **It hides real spec bugs.** If a generated type looks wrong, that means
   `openapi.yaml` is wrong (or misconfigured), not that the generated file
   is wrong. Patching the output treats the symptom and buries the actual
   problem.
4. **It breaks the single-source-of-truth workflow.** The whole point of
   this pipeline — `openapi.yaml` → `openapi-typescript` → `types.gen.ts` →
   `index.ts` → apps — is that there's exactly one place to make a change.
   Hand-editing generated output creates a second, competing source of
   truth that will eventually contradict the first.

**The rule of thumb:** if you need a type to change, edit `openapi.yaml`,
then run `npm run gen`. Never edit `src/types.gen.ts` directly.
