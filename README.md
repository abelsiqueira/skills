# skills

A collection of agent skills in the [Agent Skills](https://agentskills.io) format. Each skill is a folder under `skills/` with a `SKILL.md`.

| Skill | Use it when |
| --- | --- |
| [`marimo-library`](skills/marimo-library/SKILL.md) | A project has marimo notebooks next to a local Python library, and you need to decide where code lives. |
| [`marimo-playwright`](skills/marimo-playwright/SKILL.md) | A marimo notebook is served as an app and needs browser tests. |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add abelsiqueira/skills
```

It asks which skills to install and for which agents. To install one skill for Claude Code, for every project:

```bash
npx skills add abelsiqueira/skills --skill marimo-library -g -a claude-code
```

Or by hand, with a clone and a link:

```bash
git clone https://github.com/abelsiqueira/skills
ln -s "$PWD/skills/skills/marimo-library" ~/.claude/skills/marimo-library
```

`marimo-library` relies on [marimo-pair](https://github.com/marimo-team/marimo-pair) for work on a running notebook. Install it too.

## Why marimo-library

- Writing everything in a notebook and moving it to a library "at the end" turns the move into a rewrite. With a library from day one, code moves in small steps, when a rule says so.
- Cells are hard to lint, test and document. Library code is not.
- An agent that reads a whole notebook to change one cell wastes its context. Code that has moved to the library no longer needs reading.

[Vox Paginae](https://github.com/nlesc-rags2riches/vox-paginae) is a project built this way: two notebooks served as apps, one shared library, and browser tests.

## License

[Apache-2.0](LICENSE)
