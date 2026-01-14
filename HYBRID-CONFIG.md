# Hybrid OpenCode Config (ML + Systems)

This repo is a hybrid OpenCode configuration focused on:

- ML engineering + Python development
- Systems administration + networking
- DevOps workflows

It is designed to be used via symlink at `~/.config/opencode`.

## Local Setup

1. Clone this repo:

```bash
git clone https://github.com/reggles10/opencode-config.git ~/repos/personal/opencode-config
```

2. Backup any existing config and symlink:

```bash
mv ~/.config/opencode ~/.config/opencode.backup-$(date +%Y%m%d-%H%M%S)
ln -s ~/repos/personal/opencode-config ~/.config/opencode
```

3. Install dependencies:

```bash
bun install
npm install -g opencode-swarm-plugin
brew install ollama
ollama serve &
ollama pull nomic-embed-text
```

4. Verify:

```bash
swarm doctor
```

## Agents

This config includes 23 agents.

### ML/Python/Data

- `ai-engineer.md`
- `ml-engineer.md`
- `data-scientist.md`
- `data-engineer.md`
- `data-analyst.md`
- `python-pro.md`
- `rust-engineer.md`
- `git-workflow-manager.md`
- `quarto-expert.md`
- `marimo-expert.md`
- `manim-expert.md`
- `code-reviewer.md`

### Systems

- `sysadmin-engineer.md`
- `network-specialist.md`
- `devops-engineer.md`

### Prompting

- `prompt-engineer.md`

### Review

- `code-reviewer-pro.md` (replaces legacy `reviewer.md`)

## Knowledge Files

- `knowledge/ml-patterns.md`
- `knowledge/python-async-patterns.md`
- `knowledge/systems-admin-patterns.md`

## Notes

- `swarm doctor` reports `CASS` missing as optional. If you want it, install from:
  `https://github.com/Dicklesworthstone/coding_agent_session_search`
