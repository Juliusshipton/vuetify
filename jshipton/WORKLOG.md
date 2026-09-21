# Manual Entry For Work

## Step 1 Environment setup

Prerequisites: Node.js (LTS), pnpm, Git >= 2.20.

Fork `vuetifyjs/vuetify`, clone the fork, and branch from `upstream/dev`
(per the contributing guide: bug fixes target `master`, features target `dev`).

From the repository root:

```bash
pnpm i               # install workspace dependencies
pnpm build vuetify   # build the component library
pnpm build api       # generate API metadata used by the docs
pnpm dev             # start the dev server on http://localhost:8090
```

`pnpm dev` serves `packages/vuetify/dev/Playground.vue`, which renders against
the local source — the sandbox used throughout this exercise to reproduce the
issue and verify changes.

## Step 2 Repository Structure Map, Recreate Issue (Concurrent Prompts)

One prompt on (Fable 5.1) to create the jshipton/REPO_ANALYSIS.md for a broad description of repo structure and .  

One prompt on (Opus 5) to create the specific file jshipton/Playground.vue to recreate the issue.  

These prompts were executed concurrently as they are separate concerns and also don't need the same model if token usage is a concern. 

## Step 3 Implementation Plan & Failing Tests

Conversation with (Fable 5.1) to establish the jshipton/IMPLEMENTATION_PLAN.md

Failing tests written first in `packages/vuetify/src/labs/VHeatmap/__tests__/heatmap.spec.ts`
(new `useHeatmap firstDayOfWeek` block). Expect 7 failed / 17 passed before implementation.

The `test` script lives in `packages/vuetify/package.json`, so run from there
(from `packages` or the root pnpm recurses into every package and fails):

```bash
cd packages/vuetify
pnpm test src/labs/VHeatmap --project unit --run   # run once, jsdom only
pnpm test src/labs/VHeatmap                        # watch mode, re-runs on save
pnpm test src/labs/VHeatmap -t firstDayOfWeek      # only the new block
```

## Step 4 Implementation

Signed off on implementation and (Fable 5.1) completed successfully. 

Verified manually with pre approved test cases and visual verification with playground environment. 

Feature Request Completed. 