# Project guidance

## Scope and architecture

- This repository is the Modular RAG MCP Server, a Python 3.10+ project. Keep changes focused on the requested task and preserve the existing modular, provider-driven design.
- Runtime settings belong in `config/settings.yaml` and are loaded through the existing settings/configuration modules. Do not hardcode values that are already configurable.
- Keep MCP protocol handling in `src/mcp_server/`, ingestion in `src/ingestion/`, shared types and settings in `src/core/`, and provider implementations under `src/libs/`.
- Keep tests under the matching `tests/unit/`, `tests/integration/`, or `tests/e2e/` directory and follow nearby patterns.

## Working rules

- Read the relevant sections of `DEV_SPEC.md` and existing implementation before changing behavior; treat the specification and established code patterns as the project reference.
- Do not modify unrelated business code or generated test fixtures when handling documentation or agent-configuration tasks.
- Keep API keys and other credentials out of source, documentation, and committed configuration. Use placeholders in examples.
- `.agents/skills/` contains project-specific reusable workflows and reference material. Preserve those files unless a change explicitly concerns them; `.claude/` is not the canonical location for project rules or skills.
- Do not create commits unless the user explicitly requests one.

## Commands

- Install project dependencies using the project's documented/environment setup; development dependencies are declared in `pyproject.toml`.
- Run the focused pytest file(s) relevant to a code change. The pytest configuration and markers are defined in `pyproject.toml`.
- Use `ruff` for linting when requested or when validating Python changes.
