# Copilot instructions for get-started-with-ai-chat

## Project shape

This repo is a Python + React sample app for an Azure-hosted AI chat experience.

- Backend: `src/` contains the FastAPI app, AI model integration, and the RAG/search logic.
- Frontend: `src/frontend/` contains the Vite/React UI, which is built into static assets and served by the backend.
- Deployment: `azure.yaml` and `infra/` define the `azd` deployment to Azure Container Apps and the supporting Azure resources.
- Runtime data: `src/api/data/embeddings.csv` is the knowledge base used by the Azure AI Search vector index.

The key runtime path is:

- `src/api/main.py` creates the FastAPI app and initializes Azure clients.
- `src/api/routes.py` defines the `/` and `/chat` endpoints and the SSE response stream.
- `src/api/search_index_manager.py` owns the Azure AI Search index lifecycle and vector queries.
- `src/frontend/src/components/agents/AgentPreview.tsx` is the main UI integration point for chat requests and streaming responses.

## Build, test, and lint commands

Use the repo-root commands below when working locally.

### Environment setup

```bash
cd src
python -m pip install -r requirements.txt
```

For linting/dev tools:

```bash
python -m pip install -r ../requirements-dev.txt
```

### Frontend setup and build

```bash
cd src/frontend
pnpm install
pnpm build
```

The repo also has a convenience script:

```bash
cd src/frontend
pnpm run setup
```

This installs pnpm, installs dependencies, and runs the frontend build.

### Local app run

```bash
cd src
python -m uvicorn "api.main:create_app" --factory --reload
```

If you are changing `gunicorn.conf.py` or the search-index startup behavior, validate the configuration first:

```bash
cd src
python gunicorn.conf.py
```

### Azure deployment commands

```bash
azd up
azd down
```

### Test commands

The project has a focused unit test for the search index manager:

```bash
python -m pytest tests/test_search_index_manager.py -q
```

To run a single test method in that file:

```bash
python -m pytest tests/test_search_index_manager.py -q -k test_index_exist_mock
```

### Lint commands

```bash
ruff check src tests
ruff format --check src tests
```

If `ruff` is not on PATH in your environment, use:

```bash
python -m ruff check src tests
python -m ruff format --check src tests
```

## High-level architecture

### Backend architecture

The backend is a FastAPI app that boots Azure AI clients during startup. In `src/api/main.py`, the app reads environment variables, chooses Azure authentication flow (`AzureDeveloperCliCredential` in local dev vs `ManagedIdentityCredential` in production), creates an `AIProjectClient`, and configures the chat and embeddings clients.

Then it conditionally creates the Azure AI Search index for RAG when the search-related environment variables are present. This means the app can run either as a plain chat app or as a RAG-enabled chat app depending on the deployment configuration.

`src/api/routes.py` handles the streaming responses from the model. It calls the search index first when enabled, puts the retrieved context into the prompt, and then streams content back to the browser with Server-Sent Events.

### Frontend architecture

The frontend is a Vite React app under `src/frontend/src`. The app is intentionally minimal: it mounts a single `AgentPreview` component and wraps it in the ThemeProvider. The main interaction logic lives in the agent components under `src/frontend/src/components/agents/`.

The build output is emitted to `src/api/static/react`, and the backend serves the compiled frontend from that directory via `app.mount("/static", StaticFiles(...))`.

### Deployment and Azure infrastructure

`azure.yaml` defines the app service container deployment, while `infra/` contains the Bicep modules. The repo is designed around Azure AI Foundry / Azure AI Services, Azure AI Search, Application Insights, and Container Apps. The deployment is not a plain local-only app; most configuration is environment-driven and the app expects Azure service endpoints and model deployment names to be set.

## Key conventions and repo-specific patterns

- Keep server-side AI configuration in environment variables; the app resolves model and search settings from `os.environ` rather than hard-coded values.
- RAG is startup-driven: the search index is created when the app starts if the required Azure AI Search environment variables are configured. Do not assume the knowledge base is file-backed in a normal directory scan; the app relies on the precomputed embeddings CSV and search index creation.
- The repo prefers `azd` as the canonical deployment flow. For local development, the app is run with `uvicorn` using the app factory pattern, not with a custom Flask/Django entrypoint.
- The frontend is built before serving in production; treat `src/frontend` changes as a build artifact concern unless the backend explicitly needs the static assets to be regenerated.
- The app uses streaming chat responses (`text/event-stream`), so changes to request/response handling should preserve the SSE format used by the frontend and backend together.
- Be careful with search/index configuration changes; they are tied to both the startup logic in `main.py` and the vector index logic in `search_index_manager.py`.
- The repository preserves Azure-specific version constraints in `src/requirements.txt` and the Docker build. Do not casually upgrade dependency versions without checking compatibility implications.

## When making changes

- Prefer editing the Python backend and React frontend in-place rather than introducing a second app structure.
- If you change the frontend, run `pnpm build` so the generated assets remain consistent with the backend’s static serving setup.
- If you change Azure resource shape or deployment behavior, check `azure.yaml` and the Bicep templates under `infra/` together; they are part of the same deployment story.
- Update or validate the relevant tests when changing search/index or chat behavior; the focused test file is `tests/test_search_index_manager.py`.
