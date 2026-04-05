# use-effect-killer

An agent skill that audits your React codebase for `useEffect` anti-patterns and proposes idiomatic fixes. Works with any coding agent that supports the [Agent Skills](https://github.com/vercel-labs/skills) specification — Claude Code, Codex, Cursor, GitHub Copilot, Gemini CLI, and more.

Based on React's official guide: [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect).

## What it detects

| # | Anti-Pattern | Recommended Fix |
|---|-------------|-----------------|
| 1 | Derived state via effect | Compute inline (`const x = a + b`) |
| 2 | Expensive computation in effect | `useMemo` |
| 3 | Reset all state on prop change | `key={prop}` on child component |
| 4 | Adjust some state on prop change | Derive during render |
| 5 | Event logic in effect | Move to event handler |
| 6 | POST/mutation in effect | Move to event handler |
| 7 | Effect chains | Consolidate into event handler + derived state |
| 8 | App init in effect | Module-level code or `didInit` guard |
| 9 | Notify parent via effect | Call callback in event handler |
| 10 | Pass data to parent via effect | Lift data fetching to parent |
| 11 | External store subscription | `useSyncExternalStore` |
| 12 | Initialize state from props via effect | Pass prop directly to `useState` |
| 13 | Fetch without cleanup | Add ignore flag / abort controller, or use a data-fetching library |

## Install

### One-command install (recommended)

```bash
npx skills add victor36max/use-effect-killer
```

### Manual install (Claude Code)

```bash
mkdir -p .claude/skills/use-effect-killer
curl -sL https://raw.githubusercontent.com/victor36max/use-effect-killer/main/SKILL.md \
  -o .claude/skills/use-effect-killer/SKILL.md
```

### Manual install (Codex / other agents)

```bash
mkdir -p .agents/skills/use-effect-killer
curl -sL https://raw.githubusercontent.com/victor36max/use-effect-killer/main/SKILL.md \
  -o .agents/skills/use-effect-killer/SKILL.md
```

## Usage

In Claude Code, run:

```
/use-effect-killer
```

Or scope to a specific directory:

```
/use-effect-killer src/components
```

The skill will scan for all `useEffect` usages, classify each one against the 13 anti-patterns, and produce a structured report with file paths, line numbers, and concrete fix suggestions.

## License

MIT
