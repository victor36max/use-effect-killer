# use-effect-killer

A [Claude Code](https://claude.ai/code) skill that audits your React codebase for `useEffect` anti-patterns and proposes idiomatic fixes.

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
| 12 | Fetch without cleanup | Add ignore flag / abort controller, or use a data-fetching library |

## Install

### Project-level (shared with team)

```bash
# From your project root
mkdir -p .claude/skills
cp -r use-effect-killer .claude/skills/
```

### Personal (available across all your projects)

```bash
mkdir -p ~/.claude/skills
cp -r use-effect-killer ~/.claude/skills/
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

The skill will scan for all `useEffect` usages, classify each one against the 12 anti-patterns, and produce a structured report with file paths, line numbers, and concrete fix suggestions.

## License

MIT
