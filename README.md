# agents-md

Project-agnostic `AGENTS.md` template used across the `~/pse/*` repos. Codifies the development workflow that has held up under sustained multi-PR refactor series:

- TDD (red → green → refactor)
- `backlog` CLI for progress tracking
- `jj` (Jujutsu) for version control
- A strict PR review loop on top of `roborev` + GitHub reviewer bots (`gemini-code-assist`, `coderabbitai`, `Cursor Bugbot`), including `close` and `compact` discipline

The template exists because the same workflow rules kept getting re-derived (and re-violated) per repo. Two real bugs shipped through 7 PRs in `morphogenesis` before the loop was written down; this template is the written-down version.

## What's project-agnostic vs project-specific

| Section | Scope |
|---------|-------|
| `## Development Workflow` (TDD, Progress Tracking, Version Control — `jj`, Pull Request review loop, Hard-won lessons) | **Shared** — copied verbatim |
| `## Project-specific` (Build & Test Commands, Project Structure, Project invariants) | **Per-repo** — each repo fills these in |

## Installation options

### Option 1: Symlink (recommended for `~/pse/*`)

Simplest. Single source of truth. Catches upstream updates with `git -C ~/pse/agents-md pull && jj git import` in the consuming repo.

```bash
cd ~/pse/<repo>
mv AGENTS.md AGENTS.md.bak           # if a local one exists
ln -sf ~/pse/agents-md/AGENTS.md AGENTS.md
# Then append the project-specific section:
# - either edit the symlink target (shared) for global changes
# - or remove the symlink and copy the file in (Option 2) for full local control
```

Caveat: a pure symlink gives every consuming repo the *same* project-specific
section, which is wrong. Use Option 2 if you need per-repo customization, or
hybrid (Option 3).

### Option 2: Copy

Full local control, no automatic updates. Re-sync by `diff` when needed.

```bash
cd ~/pse/<repo>
cp ~/pse/agents-md/AGENTS.md AGENTS.md
# Edit the bottom section to add build commands + project structure.
```

### Option 3: Hybrid (submodule + include-style)

Use a git submodule so the shared template is version-pinned per repo, then
compose it with a small per-repo preamble:

```bash
cd ~/pse/<repo>
git submodule add https://github.com/igor53627/agents-md.git .agents-md
# AGENTS.md (committed in the repo, NOT a symlink):
cat > AGENTS.md <<'EOF'
# <Project> — Agent Guidelines

This repo extends the [shared PSE agent template](.agents-md/AGENTS.md).
Read that file first for the workflow (TDD, backlog, jj, roborev PR loop).

## Project-specific
...build commands, layout, invariants...
EOF
```

This is more work but lets reviewers open `.agents-md/AGENTS.md` for the
shared rules while keeping per-repo AGENTS.md focused on project specifics.
Recommended for repos shared with contributors who don't have `~/pse/` set
up locally.

## Updating the template

1. Edit `AGENTS.md` in this repo.
2. `jj describe -m "docs(agents): ..."` and `jj git push`.
3. Open a PR; this repo's own `AGENTS.md` applies, so follow the review loop.
4. After merge, sync consuming repos: `git -C ~/pse/agents-md pull && jj git import` (Option 1) or re-`cp` (Option 2) or bump submodule (Option 3).

## License

MIT. See `LICENSE`.
