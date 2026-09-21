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

