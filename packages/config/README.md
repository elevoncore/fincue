# `packages/config`

Shared tooling configuration so every app/package in the monorepo lints and
compiles the same way — not yet populated.

## Intended contents

```text
config/
├── eslint/
│   └── base.js          # Shared ESLint config, extended by apps/web/.eslintrc
├── typescript/
│   ├── base.json         # Shared tsconfig compilerOptions
│   ├── nextjs.json        # Next.js-specific overrides, extends base.json
│   └── react-native.json  # Expo-specific overrides, extends base.json (Phase 10+)
└── tailwind/
    └── base.js           # Shared design tokens (colors, spacing) — see docs/gamification-strategy.md for the color-nudge palette
```

## Why this exists

Without a shared config package, `apps/web` and (later) `apps/mobile` tend
to drift into subtly different linting/formatting rules, which shows up as
noisy diffs and "why does this pass CI on their machine but not mine"
confusion. Centralizing it here means a rule change happens once.
