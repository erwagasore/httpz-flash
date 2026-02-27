# AGENTS — httpz-flash

Operating rules for humans + AI.

## Workflow

- Never commit to `main`/`master`.
- Always start on a new branch.
- Only push after the user approves.
- Merge via PR.

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/).

- fix → patch
- feat → minor
- feat! / BREAKING CHANGE → major
- chore, docs, refactor, test, ci, style, perf → no version change

## Releases

- Semantic versioning.
- Versions derived from Conventional Commits.
- Release performed locally via `/create-release` (no CI required).
- Manifest (if present) is source of truth.
- Tags: vX.Y.Z

## Repo map

```
.gitignore        — Zig build artefact ignores
README.md         — Project overview and quickstart
AGENTS.md         — Operating rules for humans + AI
DESIGN.md         — Architecture and implementation design
LICENSE           — Licence terms
docs/             — Project documentation
  index.md        — Documentation index
```

## Merge strategy

- Prefer squash merge.
- PR title must be a valid Conventional Commit.

## Definition of done

- Works locally.
- Tests updated if behaviour changed.
- CHANGELOG updated when user-facing.
- No secrets committed.

## Orientation

- **Entry point**: `src/` (once created) — the middleware library source. See `DESIGN.md` for planned file layout.
- **Domain**: Cookie-based flash messages (one-time notifications between HTTP requests) for the httpz Zig HTTP server framework. Includes an `HX-Trigger` helper for HTMX partial swaps.
- **Stack**: Zig, httpz. No external dependencies beyond httpz.
