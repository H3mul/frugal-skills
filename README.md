# frugal-skills

Claude Code plugin: skills that push Claude to be analytical and frugal with tokens.

## Skills

| Command | What it does |
|---|---|
| `/frugal-skills:implement` | Runs a prepared plan cheaply: Haiku subagents for large plans, sequential edits for small ones, tests batched at the end of each phase. |

## Install

```
/plugin marketplace add H3mul/frugal-skills
/plugin install frugal-skills@frugal-skills
```

## Try locally

```
claude --plugin-dir /path/to/frugal-skills
```

## Adding a skill

Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter. It becomes `/frugal-skills:<name>`.
