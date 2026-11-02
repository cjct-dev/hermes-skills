# Hermes Skills

Shared [Hermes Agent](https://hermes-agent.nousresearch.com/) skills. Each skill lives in `skills/<name>/` following the [agentskills.io](https://agentskills.io/specification) standard.

## Install

Install a single skill:

```bash
hermes skills install cjct-dev/hermes-skills/linux-media-player-control
```

Or add the whole repo as a tap:

```bash
hermes skills tap add cjct-dev/hermes-skills
```

## Skills

| Skill | What it does |
|---|---|
| [linux-media-player-control](skills/linux-media-player-control) | Control MPRIS media players (Elisa, VLC, Cantata, ...) on Linux via the user DBus session bus with `busctl` — no plugin needed. |

## Contributing

Skills are plain markdown with a `SKILL.md` (YAML frontmatter: `name`, `description`, `version`). Submit a PR with the skill under `skills/<name>/`.
