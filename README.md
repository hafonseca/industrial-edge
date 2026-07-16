# industrial-edge — Claude Code skill

A [Claude Code](https://claude.com/claude-code) skill for deploying and packaging apps on **Siemens Industrial Edge (IE)**.

IE apps are *almost* normal `docker-compose` stacks, but with hard constraints that silently break deploys (or destabilize the whole device) if ignored. This skill encodes those rules so Claude gets them right every time:

- Compose schema `'2.4'` (not v3.x)
- Named volumes only — no bind mounts, and names are **globally unique across all apps on the device** (volumes, `container_name`, host ports)
- Mandatory `mem_limit` / `mem_reservation` on every service, with the device-wide reset-to-reservation behavior explained
- Always the external `proxy-redirect` network; official `zzz_layer2_net1` pattern for Layer-2/fieldbus access (Profinet, GigE Vision, …)
- No `build:` — prebuilt, pinned images only
- Never share a volume between containers

It also carries an append-only **lessons-learned log** that grows as projects use it — the skill asks at the end of a session whether anything new about IE was learned and folds it back in.

## Install

Copy `SKILL.md` into your global Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/industrial-edge
curl -fsSL https://raw.githubusercontent.com/hafonseca/industrial-edge/main/SKILL.md \
  -o ~/.claude/skills/industrial-edge/SKILL.md
```

Or clone and symlink so `git pull` keeps it updated:

```bash
git clone git@github.com:hafonseca/industrial-edge.git
ln -s "$(pwd)/industrial-edge" ~/.claude/skills/industrial-edge
```

Claude Code picks it up automatically — it triggers whenever a task mentions Industrial Edge, IE apps, edge devices, or IE-specific compose constraints. You can also invoke it explicitly with `/industrial-edge`.

## Contributing lessons

Learned a new IE constraint, firmware gotcha, or name-collision trap? Open a PR appending to the **Lessons learned** log at the bottom of `SKILL.md` (format: `- [YYYY-MM] <lesson> — <project/context>`). Keep entries terse and normative; promote firm rules into the numbered rules section.
