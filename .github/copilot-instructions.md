# ValueCell repository instructions

## Build, test, and lint commands

### Full stack

- Start the local app stack from the repo root: `bash start.sh`
- Windows entrypoint: `.\start.ps1`
- Backend only: `bash start.sh --no-frontend`
- Frontend only: `bash start.sh --no-backend`

### Backend (`python/`)

- Install dev dependencies: `cd python && uv sync --group dev`
- Run the API server: `cd python && uv run python -m valuecell.server.main`
- Run all tests: `cd python && uv run pytest`
- Run one test file: `cd python && uv run pytest valuecell/core/plan/tests/test_planner.py`
- Run one test case: `cd python && uv run pytest valuecell/core/plan/tests/test_planner.py::test_name`
- Root shortcuts: `make format`, `make lint`, `make test`
- Lint: `ruff check --config ./python/pyproject.toml ./python/`
- Format: `ruff format --config ./python/pyproject.toml ./python/`
- Import sort: `uv run --directory ./python isort .`

### Frontend (`frontend/`)

- Install dependencies: `cd frontend && bun install`
- Run the web app: `cd frontend && bun dev`
- Build: `cd frontend && bun run build`
- Type-check: `cd frontend && bun run typecheck`
- Lint: `cd frontend && bun run lint`
- Format/check formatting: `cd frontend && bun run format` or `cd frontend && bun run check`
- There is currently no frontend test script in `frontend/package.json`

## High-level architecture

- `frontend/` is a React Router v7 app. It uses TanStack Query for server state, persisted Zustand stores for local/session state, `i18next` for locales, and a custom fetch-based POST SSE client to consume streamed agent output.
- `python/valuecell/server/` is the FastAPI backend exposed under `/api/v1`. App startup creates/loads the system-level `.env`, initializes the SQLite database, and configures market data adapters before serving API routes.
- `python/valuecell/core/` is the orchestration runtime. `coordinate/orchestrator.py` is the main hub: it runs Super Agent triage, planning with human-in-the-loop checkpoints, task execution, and streaming/persistence of responses.
- Remote agents are A2A services. Agent implementations live under `python/valuecell/agents/`; `create_wrapped_agent(...)` turns an agent class into a served A2A endpoint, and `python/configs/agent_cards/*.json` declares how the runtime discovers and connects to each agent.
- Streaming output is a first-class contract. Task status events are routed into typed responses, buffered into stable conversation items, persisted, and then rendered by the frontend chat/task renderers.

## Key conventions

- Use **uv** for Python work and **bun** for frontend work. The root `Makefile` only covers Python format/lint/test tasks.
- Runtime config is **not** kept in the repo root `.env`. The backend and start scripts use the OS-specific ValueCell application directory (for `.env`, SQLite, LanceDB, and knowledge files), so changes to configuration/loading should preserve that behavior.
- New agents need both pieces of wiring: the Python agent module and a matching agent card JSON. The card `name` must match the agent class name used by `create_wrapped_agent(...)`.
- Preserve the typed streaming protocol across backend and frontend. Event names, `component_type`, `component_id`, tool-call metadata, and stable `item_id` behavior are coupled to frontend rendering and incremental conversation persistence.
- Frontend server calls should follow the existing pattern: hooks in `frontend/src/api/`, query keys centralized in `frontend/src/constants/api.ts`, and persisted app state in `frontend/src/store/`.
- Supported language keys are `en`, `zh_CN`, `zh_TW`, and `ja`. Settings state drives both `i18next` and the root HTML `lang`, so new locale work should update both surfaces consistently.
- Backend API handlers return the standard envelope shape (`code`, `data`, `msg`), and the frontend `apiClient` is built around that contract, including token refresh through the system store.

## Relevant MCP/tooling

- If a Playwright MCP server is available, use it for end-to-end checks on the React frontend, especially chat streaming flows, route navigation, settings screens, and backend connectivity from the browser.
- Prefer running the local stack first (`bash start.sh` or `cd frontend && bun dev` plus `cd python && uv run python -m valuecell.server.main`) before browser automation.

## Reference docs already in the repo

- `README.md`
- `.github/CONTRIBUTING.md`
- `docs/CORE_ARCHITECTURE.md`
- `docs/CONTRIBUTE_AN_AGENT.md`
- `AGENTS.md`
