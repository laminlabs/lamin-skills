# Lamin Skills

These skills teach AI coding agents how to correctly use LaminDB, curate datasets, and interact with the Lamin ecosystem. They follow the open [Agent Skills standard](https://agentskills.io).

## Installation

**Note:** If you have installed `lamindb` (e.g., via `pip install lamindb`), you **already have these skills**! They are bundled directly with the package.

You only need to install them manually if you want to use the absolute latest skills that haven't been included in a `lamindb` release yet, or if you are working in an environment without the `lamindb` package installed.

If you need to install them manually, you can use the following methods:

### 1. Via `npx skills`

You can install the skills globally or locally using the standard Agent Skills CLI:

```bash
npx skills add laminlabs/lamin-skills
```

### 2. Via GitHub CLI

If you use the GitHub CLI (`gh`), you can install the skills with the `gh skill` extension:

```bash
gh skill install laminlabs/lamin-skills
```

## Compatibility with `library-skills`

Lamin Skills are bundled directly with the `lamindb` Python package in the `.agents/skills/` directory. This makes them fully consistent with [`library-skills`](https://github.com/tiangolo/library-skills), allowing you to use the official Lamin skills seamlessly alongside other agent skills you might have in your projects.
