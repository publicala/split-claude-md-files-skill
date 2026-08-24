# split-claude-md-files

> [!IMPORTANT]
> **This repo has moved.** The skill now lives as `/bonsai:split` in the [bonsai plugin](https://github.com/publicala/claude-plugins/tree/main/plugins/bonsai) inside [publicala/claude-plugins](https://github.com/publicala/claude-plugins). Install with `/plugin marketplace add publicala/claude-plugins` then `/plugin install bonsai@publicala`. This repo is archived and kept for history.

Claude Code skill - moves CLAUDE.md rules to the load scope of the sessions they govern: demotes always-resident rules into nested CLAUDE.md files or path-scoped rules, promotes scoped rules that outgrew their file, and flags orphaned scoped files.

The CLAUDE.md quartet: [feed-claude-md-files](https://github.com/publicala/feed-claude-md-files-skill) adds rules from observed patterns, [bake-claude-md-files](https://github.com/publicala/bake-claude-md-files-skill) converts crystallized rules into tooling, [audit-claude-md-files](https://github.com/publicala/audit-claude-md-files-skill) prunes and verifies what remains, and [split-claude-md-files](https://github.com/publicala/split-claude-md-files-skill) moves what remains to the scope that reads it. Install all four from [publicala/claude-plugins](https://github.com/publicala/claude-plugins).

## How it works

1. Inventories every CLAUDE.md and `.claude/rules/` file with its load class (always resident, nested, path-scoped) and est. token cost
2. Greps the first-party paths each resident rule binds and lets the result decide the shape: one subtree, one glob family, or path-free
3. Checks the load path: scoped files load on reads only, so a rule that binds new-file creation or shell actions gets a probe (a fresh agent runs the task and its tool calls show whether the scoped file would have loaded in time)
4. Runs the same evidence in reverse: promotes scoped rules whose governed paths outgrew their file, flags orphaned scoped files
5. Presents the evidence-backed placement map, then applies approved moves verbatim as a PR

Safety prohibitions and routing clauses never demote: the action they guard is not gated behind reading one subtree, so the scoped copy could be absent at the moment it matters.

## Install

### Via Plugin Marketplace

```
/plugin marketplace add publicala/claude-plugins
/plugin install split-claude-md-files@publicala
```

### Via skills.sh

```bash
npx skills add publicala/split-claude-md-files-skill
```

### Manual

Copy `skills/split-claude-md-files/SKILL.md` into your skills directory:

```bash
# Global (all projects)
mkdir -p ~/.claude/skills/split-claude-md-files
cp skills/split-claude-md-files/SKILL.md ~/.claude/skills/split-claude-md-files/

# Project-level
mkdir -p .claude/skills/split-claude-md-files
cp skills/split-claude-md-files/SKILL.md .claude/skills/split-claude-md-files/
```

## Usage

Run it in the project you want reorganized. The command depends on how you installed it:

- **skills.sh / manual**: `/split-claude-md-files`
- **Plugin marketplace**: `/split-claude-md-files:split-claude-md-files` (plugin skills are namespaced as `/<plugin>:<skill>`)

Run it after an `audit-claude-md-files` pass, when a resident CLAUDE.md carries rules that only govern one part of the project.

## Resources

- [feed-claude-md-files](https://github.com/publicala/feed-claude-md-files-skill) - Surfaces patterns into new CLAUDE.md rules
- [bake-claude-md-files](https://github.com/publicala/bake-claude-md-files-skill) - Converts CLAUDE.md rules into automated checks
- [audit-claude-md-files](https://github.com/publicala/audit-claude-md-files-skill) - Prunes CLAUDE.md files with evidence-backed cuts
- [CLAUDE.md Guide](https://github.com/publicala/claude-md-guide) - Presentation slides about CLAUDE.md files
- [CLAUDE.md docs](https://docs.anthropic.com/en/docs/claude-code/memory) - Official documentation

## License

MIT
