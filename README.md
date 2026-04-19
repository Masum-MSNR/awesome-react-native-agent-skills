# Awesome React Native Agent Skills

[![Agent Skills](https://img.shields.io/badge/Agent-Skills-blue?style=flat&logo=github)](https://agentskills.io)
[![React Native](https://img.shields.io/badge/React%20Native-0.75+-61dafb?style=flat&logo=react)](https://reactnative.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.4+-3178c6?style=flat&logo=typescript)](https://www.typescriptlang.org)
[![License: MIT](https://img.shields.io/badge/Code-MIT-green?style=flat)](LICENSE-CODE)
[![License: CC BY 4.0](https://img.shields.io/badge/Docs-CC%20BY%204.0-lightgrey?style=flat)](LICENSE-DOCS)

A collection of **20 agent skills** that give AI coding assistants (GitHub Copilot, Claude, Gemini, Cursor, etc.) expert-level knowledge of modern React Native and TypeScript development.

Drop the `.github/skills/` folder into your project and your agent automatically follows your architecture, uses current best practices, and ships code that passes review.

> Learn more about the Agent Skills standard at [agentskills.io](https://agentskills.io).

---

## Why use this

- **New Architecture first** -- skills assume Fabric + TurboModules enabled, with Hermes and bridgeless mode as the default.
- **Modern defaults** -- React Native 0.75+, TypeScript 5.4+ strict, React Navigation 7, TanStack Query 5, Reanimated 3, FlashList.
- **Opinionated but swappable** -- each skill explains the preferred tool *and* the alternative (Zustand vs Redux Toolkit, Maestro vs Detox, Unistyles vs NativeWind).
- **Actionable checklists** -- every skill ends with a verification checklist the agent can self-audit against.

---

## How it works

The `Agent.md` file is the foundation. Every AI agent reads it first to understand your project conventions. Individual skills layer on top for specific tasks.

```text
       ┌───────────────────────────────────────────┐
       │                  Agent.md                 │
       │     (read by all AI agents implicitly)    │
       └─────────────────────┬─────────────────────┘
                             │
     ┌───────────────┬───────┴───────┬───────────────┐
     ▼               ▼               ▼               ▼
┌──────────┐   ┌───────────┐   ┌────────────┐  ┌────────────┐
│ Architect│   │  UI & UX  │   │ Migration  │  │  Delivery  │
├──────────┤   ├───────────┤   ├────────────┤  ├────────────┤
│structure │   │components │   │class →     │  │testing     │
│state-mgmt│   │navigation │   │  hooks     │  │e2e         │
│data-layer│   │images     │   │old arch →  │  │flavors     │
│          │   │a11y       │   │  new arch  │  │ci-cd       │
│          │   │animations │   │            │  │            │
│          │   │i18n       │   │            │  │            │
└──────────┘   └───────────┘   └────────────┘  └────────────┘
```

---

## Available Skills

All skills live in `.github/skills/<category>/<skill-name>/SKILL.md`.

### Architecture

| Skill | What it covers |
|-------|----------------|
| [RN Architecture](.github/skills/architecture/rn-architecture/SKILL.md) | Feature-sliced layout, New Architecture enablement, module boundaries |
| [State Management](.github/skills/architecture/rn-state-management/SKILL.md) | Zustand vs Redux Toolkit vs Jotai, server state with TanStack Query |
| [Data Layer & Offline-First](.github/skills/architecture/rn-data-layer/SKILL.md) | MMKV / WatermelonDB / SQLite repository pattern |

### UI

| Skill | What it covers |
|-------|----------------|
| [Components](.github/skills/ui/rn-components/SKILL.md) | Composable components, StyleSheet vs Unistyles vs NativeWind, FlashList |
| [Navigation](.github/skills/ui/rn-navigation/SKILL.md) | React Navigation 7 native-stack, deep links, typed routes, tab+stack |
| [Images](.github/skills/ui/rn-images/SKILL.md) | `expo-image` / FastImage, caching, placeholders, priority, memory |
| [Accessibility](.github/skills/ui/rn-accessibility/SKILL.md) | `accessibilityRole`, labels, `importantForAccessibility`, RTL |
| [Animations](.github/skills/ui/rn-animations/SKILL.md) | Reanimated 3 worklets, Gesture Handler, Skia, Moti |
| [Localization & i18n](.github/skills/ui/rn-localization/SKILL.md) | i18next / Lingui, ICU plurals, dynamic locale, RTL mirroring |

### Migration

| Skill | What it covers |
|-------|----------------|
| [Class to Hooks](.github/skills/migration/class-to-hooks/SKILL.md) | Legacy class components to function components + hooks |
| [Old Arch to New Arch](.github/skills/migration/old-arch-to-new-arch/SKILL.md) | Fabric + TurboModules, codegen, interop module |

### Performance

| Skill | What it covers |
|-------|----------------|
| [Performance Audit](.github/skills/performance/rn-performance-audit/SKILL.md) | Hermes profiler, re-render detection, list perf |
| [Bundle Size](.github/skills/performance/rn-bundle-size/SKILL.md) | Hermes bytecode, Metro config, tree shaking, inline requires |

### Concurrency & Networking

| Skill | What it covers |
|-------|----------------|
| [Async Patterns](.github/skills/concurrency_and_networking/rn-async-patterns/SKILL.md) | Promises, async/await, cancellation with `AbortController` |
| [Data Fetching](.github/skills/concurrency_and_networking/rn-data-fetching/SKILL.md) | TanStack Query, optimistic updates, polling, retries |
| [Offline Sync](.github/skills/concurrency_and_networking/rn-offline-sync/SKILL.md) | Queueing mutations, conflict handling, background sync |

### Testing & Automation

| Skill | What it covers |
|-------|----------------|
| [Testing](.github/skills/testing_and_automation/rn-testing/SKILL.md) | Jest + React Native Testing Library, mocks, hook testing |
| [E2E Testing](.github/skills/testing_and_automation/rn-e2e-testing/SKILL.md) | Maestro vs Detox, CI setup, flake reduction |

### Build & Tooling

| Skill | What it covers |
|-------|----------------|
| [Flavors & Variants](.github/skills/build_and_tooling/rn-flavors/SKILL.md) | Android flavors, iOS schemes, runtime env selection |
| [CI/CD](.github/skills/build_and_tooling/rn-ci-cd/SKILL.md) | EAS Build, GitHub Actions, Fastlane, EAS Update, store submission |

---

## Quick Start

1. **Copy** the `.github/skills/` folder and `Agent.md` to your project root.
2. **Verify** the structure:
   ```text
   my-rn-project/
   ├── Agent.md
   ├── .github/
   │   └── skills/
   │       ├── architecture/
   │       │   ├── rn-architecture/
   │       │   │   └── SKILL.md
   │       │   └── ...
   │       └── ...
   ├── src/
   └── ...
   ```
3. **Reload** your editor (Copilot and similar extensions may need a window reload to index new skills).

Then just ask your agent naturally:

> "How should I structure the User Profile feature?"
> "Create a repository for fetching News with offline support."

The agent picks the right skill automatically.

---

## Other Environments

- **Claude**: copy or symlink `.github/skills` to `.claude/skills/`.
- **OpenCode**: supports `.opencode/skill/` and `.claude/skills/`. Global skills go in `~/.config/opencode/skill/`.

---

## Adding Custom Skills

Create a folder in `.github/skills/` with a `SKILL.md` containing the required frontmatter:

```markdown
---
name: my-custom-skill
description: What this skill does
---
# Instructions
...
```

---

## License

Dual-licensed:

- **Code** (snippets, configs, scripts) -- [MIT](LICENSE-CODE)
- **Documentation** (SKILL.md, README, Agent.md) -- [CC BY 4.0](LICENSE-DOCS)

See [LICENSE](LICENSE) for details.
