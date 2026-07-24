# sb-skills

My agent skills collection.

## Skills

### `implement-loop`

Process a batch of work items: implement each via a TDD subagent, review via a reviewer subagent, fix until clean, then create a PR. Issue-tracker-agnostic.

```
/skill:implement-loop
```

### `tightrope`

Gate code changes against the tightrope between ponytail minimalism and
engineering soundness — runs `/ponytail-review` and the `review` skill,
then interprets both reports to resolve or escalate. Pipeline: tension
check → test → lint → PR gate.

```
/tightrope [--lean ponytail|engineering] [--skip <stage>] [--intent "..."]
/tightrope <task description>
```

### `system-architecture` ⚠️ WIP

> **Work in progress.** Prima versione, non ancora battle-tested.
>
> Progetta l'architettura di sistema per un PRD: produce Mermaid sequence/state
> diagrams, endpoint contracts e data model, e li appende al PRD sul tracker.
> Progettato per essere usato tra `/to-prd` e `/to-issues`.

```
/system-architecture <prd-number>
```

### `program-design` ⚠️ WIP

> **Work in progress.** Prima versione, non ancora battle-tested.
>
> Produce program design artifacts (call-stack tree, file-tree diff, type
> signatures) per un ticket e li appende al ticket sul tracker.
> Progettato per essere usato tra `/to-issues` e `implement-loop`.

```
/program-design <ticket-number>
```

## Install

### Pi

```bash
pi install git:github.com/salvatorebottiglieri/pi-skills
```

Or symlink:

```bash
ln -s ~/path/to/pi-skills/* ~/.pi/agent/skills/
```

### Claude Code

Symlink the skills you want into `.claude/skills/`:

```bash
# In your project or globally
mkdir -p ~/.claude/skills
ln -s ~/path/to/pi-skills/* ~/.claude/skills/

# Or just copy
cp -r ~/path/to/pi-skills/* ~/.claude/skills/
```

Or reference them in `CLAUDE.md` / `CLAUDE_GLOBAL.md`:

```markdown
See skills in ~/pi-skills/ for reusable workflows.
```
