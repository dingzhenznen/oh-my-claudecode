# Repository Guidelines

## Project Structure & Module Organization
- `src/` contains the TypeScript source, organized by domain: `cli/`, `agents/`, `features/`, `hooks/`, `mcp/`, `team/`, `hud/`, and `notifications/`.
- `src/__tests__/` and feature-local `__tests__/` folders hold unit and integration tests. Prefer colocated tests when extending a module.
- `bridge/` contains published CLI bridge entrypoints, `scripts/` contains build and release helpers, and `dist/` is generated output.
- `docs/`, `templates/`, `skills/`, and `agents/` contain user-facing docs and runtime resources that may need updates when behavior changes.

## Build, Test, and Development Commands
- `npm run build` — compiles TypeScript and rebuilds bridge, CLI, MCP, and runtime artifacts.
- `npm run dev` — runs `tsc --watch` for source development.
- `npm run dev:full` — watches TypeScript plus bridge/runtime build scripts.
- `npm run test:run` — runs the full Vitest suite once.
- `npm run lint` — checks `src/` with ESLint.
- `npm run format` — formats `src/**/*.ts` with Prettier.

## Coding Style & Naming Conventions
- Use TypeScript with 2-space indentation, semicolons, and ESM imports consistent with existing files.
- Match existing naming: `kebab-case` for many filenames (`background-tasks.ts`), `camelCase` for functions, `PascalCase` for types/classes.
- Keep changes focused and minimal; extend existing modules before adding new top-level areas.
- Run `npm run lint` and `npm run format` when touching non-trivial code.

## Testing Guidelines
- Tests use `vitest`; name files `*.test.ts` and mirror the source area, e.g. `src/team/__tests__/runtime.test.ts`.
- Start with targeted tests for the module you changed, then broaden only as needed.
- Add tests for new logic or bug fixes when adjacent tests already exist; avoid introducing a new test framework.

## Commit & Pull Request Guidelines
- Follow the repository’s history: Conventional Commit style such as `feat(team): ...`, `fix(keyword-detector): ...`, or `chore(release): ...`.
- Keep commits scoped to one logical change.
- PRs should include a clear summary, affected areas, test evidence (`npm run test:run`, targeted Vitest command, etc.), and linked issues when applicable.
- Include screenshots or terminal output only for user-visible CLI/HUD changes.

## Configuration & Release Notes
- Treat `src/` as the source of truth; do not hand-edit `dist/` unless the workflow explicitly requires regenerated artifacts.
- When changing commands, agents, hooks, or docs-visible behavior, update `README.md` or relevant files in `docs/`.
