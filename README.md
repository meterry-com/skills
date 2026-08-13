# Meterry Skills Distribution

This repository is the distribution source for the `meterry` Codex skill.

## What lives here

- `plugins/meterry/` is the Codex plugin container.
- `plugins/meterry/skills/meterry/` is the actual skill.
- `.agents/plugins/marketplace.json` defines the marketplace entry for Codex.
- `.well-known/skills/index.json` is the public skill index entry.

## Installing in Codex

Install the skill with `npx skills add`:

```bash
npx skills add meterry-com/skills --skill meterry
```

If you are working from a local checkout, point `npx skills add` at the checked-out repository instead of the public GitHub source.

## Public skill index

Serve `.well-known/skills/index.json` from your domain so `/skills/meterry` can point to the
Meterry skill entry and its install source. That page should link to the actual skill content and
the Git repository rather than trying to act as the skill payload itself.


## Repository layout

```text
skills/
  README.md
  .agents/plugins/marketplace.json
  plugins/meterry/.codex-plugin/plugin.json
  plugins/meterry/skills/meterry/SKILL.md
  plugins/meterry/skills/meterry/agents/openai.yaml
```
