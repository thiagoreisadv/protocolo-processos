# Agente de Peticionamento Eletronico — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local agent that controls the user's browser to fill court platform forms using Claude API for intelligent form interpretation, with human confirmation before filing.

**Architecture:** A FastAPI backend serves a single-page HTML frontend and orchestrates three modules: Browser Controller (Playwright CDP connection to an already-logged-in Chrome), AI Interpreter (Claude API to read form HTML and propose field values), and Flow Orchestrator (ties the pipeline together with SSE status updates and a human confirmation gate). All local, no external server.

**Tech Stack:** Python 3.11+, FastAPI, Playwright, Anthropic SDK, HTML/CSS/JS (no framework)

**Spec:** `docs/superpowers/specs/2026-10-27-agente-peticionamento-design.md`

## Global Constraints

- Python 3.11+ required
- All code runs locally on localhost — no external deployments
- No JS frameworks — vanilla HTML/CSS/JS only for the frontend
- Chrome must be started with `--remote-debugging-port=9222` by the user
- API key stored in `ANTHROPIC_API_KEY` environment variable — never hardcoded
- The agent never acts without human confirmation — every protocol action requires explicit approval

## Review Focus

1. **User cancels during browser execution** — after confirming but while fields are being filled, the user hits cancel; the agent must stop mid-execution and not submit a partial form. Test in Task 5.
2. **Form with required fields the AI marks low confidence** — when Claude returns a field with `confianca: "baixa"`, the confirmation UI must highlight it and the user must be able to edit the value before confirming. Test in Task 4.
3. **Folder path with no PDF files** — user provides a folder that exists but has no documents; the system must reject early with a clear message rather than navigating and failing at upload. Test in Task 5.
4. **Browser not reachable on CDP port** — Chrome not started with debug flag; the system must return a clear error instead of hanging or crashing. Test in Task 2.
5. **SSE connection drop mid-flow** — if the frontend loses the SSE connection, reconnecting must resume from the current state, not restart the flow. Test in Task 5.

---

## File Structure

```
protocolo-processos/
  src/
    __init__.py
    main.py                  # FastAPI app, endpoints, startup
    browser_controller.py    # Playwright CDP: connect, navigate, fill, upload
    ai_interpreter.py        # Claude API: analyze form HTML, return fill proposal
    flow_orchestrator.py     # Pipeline: navigate -> capture -> analyze -> confirm -> execute
    plataformas.py           # Platform registry (URLs, navigation patterns)
    models.py                # Pydantic models for API request/response and internal state
  static/
    index.html               # Single-page frontend
    style.css                # Styles
    app.js                   # Frontend logic (SSE, confirmation UI, history)
  tests/
    __init__.py
    conftest.py              # Shared fixtures (mock browser, mock Claude, test app client)
    test_models.py           # Model validation tests
    test_plataformas.py      # Platform registry tests
    test_browser_controller.py
    test_ai_interpreter.py
    test_flow_orchestrator.py
    test_api.py              # Endpoint integration tests
  requirements.txt
  README.md
```

---

### Task 1: Project Scaffolding, Models, and Platform Registry

**Files:**
- Create: `requirements.txt`
- Create: `src/__init__.py`
- Create: `src/models.py`
- Create: `src/plataformas.py`
- Create: `tests/__init__.py`
- Create: `tests/conftest.py`
- Create: `tests/test_models.py`
- Create: `tests/test_plataformas.py`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `ProtocolarRequest(numero_processo: str, pasta_documentos: str, plataforma: str)` — Pydantic model
  - `CampoSugerido(selector: str, tipo: str, label: str, valor_sugerido: str, confianca: str)` — Pydantic model
  - `UploadSugerido(selector: str, arquivos: list[str], ordem: list[str])` — Pydantic model
  - `PropostaPreenchimento(campos: list[CampoSugerido], uploads: list[UploadSugerido], observacoes: str)` — Pydantic model
  - `TaskStatus(task_id: str, status: str, etapa: str, proposta: PropostaPreenchimento | None, erro: str | None, protocolo: str | None)` — Pydantic model
  - `HistoricoItem(task_id: str, numero_processo: str, plataforma: str, status: str, protocolo: str | None, data: str)` — Pydantic model
  - `get_plataforma(key: str) -> dict` — returns platform config or raises `ValueError`
  - `listar_plataformas() -> list[dict]` — returns all platforms with key/nome
  - `montar_url_processo(key: str, numero_processo: str) -> str` — returns full URL with process number substituted

- [ ] **Step 1: Create `requirements.txt`**

```
fastapi>=0.115.0
uvicorn>=0.30.0
playwright>=1.48.0
anthropic>=0.40.0
pydantic>=2.9.0
python-dotenv>=1.0.0
pytest>=8.3.0
pytest-asyncio>=0.24.0
httpx>=0.27.0
```

- [ ] **Step 2: Create empty `src/__init__.py` and `tests/__init__.py`**

- [ ] **Step 3: Write failing tests for Pydantic models in `tests/test_models.py`**

Test that `ProtocolarRequest` validates `numero_processo`, `pasta_documentos`, `plataforma` as required strings. Test that `TaskStatus` defaults `proposta`, `erro`, `protocolo` to `None`. Test that `CampoSugerido.confianca` accepts only `"alta"`, `"media"`, `"baixa"`.

- [ ] **Step 4: Run tests — expect FAIL (models not implemented)**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_models.py -v`

- [ ] **Step 5: Implement models in `src/models.py`**

All models as Pydantic `BaseModel`. `CampoSugerido.confianca` as `Literal["alta", "media", "baixa"]`. `TaskStatus.status` as `Literal["navegando", "analisando", "aguardando_confirmacao", "preenchendo", "concluido", "erro", "cancelado"]`.

- [ ] **Step 6: Run tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_models.py -v`

- [ ] **Step 7: Write failing tests for plataformas in `tests/test_plataformas.py`**

Test `get_plataforma("eproc_trf4")` returns dict with keys `nome`, `url_base`, `url_processo`. Test `get_plataforma("nonexistent")` raises `ValueError`. Test `montar_url_processo("eproc_trf4", "5001234-56.2026.4.04.7000")` returns URL with number substituted. Test `listar_plataformas()` returns list with at least one entry having `key` and `nome`.

- [ ] **Step 8: Run tests — expect FAIL**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_plataformas.py -v`

- [ ] **Step 9: Implement `src/plataformas.py`**

`PLATAFORMAS` dict with at least `eproc_trf4` entry per spec. Three functions: `get_plataforma`, `listar_plataformas`, `montar_url_processo`.

- [ ] **Step 10: Run tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/ -v`

- [ ] **Step 11: Create `tests/conftest.py`**

Empty for now — fixtures will be added in subsequent tasks.

- [ ] **Step 12: Commit**

```bash
git add requirements.txt src/ tests/
git commit -m "feat: add project scaffolding, Pydantic models, and platform registry"
```

---

### Task 2: Browser Controller

**Files:**
- Create: `src/browser_controller.py`
- Create: `tests/test_browser_controller.py`
- Modify: `tests/conftest.py` (add mock browser fixture)

**Interfaces:**
- Consumes: `montar_url_processo` from `src/plataformas.py`
- Produces:
  - `BrowserController` class with:
    - `async conectar(cdp_url: str = "http://localhost:9222") -> None` — connects via CDP, raises `ConnectionError` if unreachable
    - `async navegar_para_processo(plataforma: str, numero_processo: str) -> None` — navigates to process URL
    - `async capturar_formulario() -> str` — returns outer HTML of the largest `<form>` on page
    - `async preencher_campo(selector: str, valor: str, tipo: str) -> None` — fills input/select/textarea
    - `async fazer_upload(selector: str, caminhos_arquivos: list[str]) -> None` — sets input files
    - `async clicar_botao(selector: str) -> None` — clicks element
    - `async capturar_confirmacao() -> str` — waits for navigation/new content after submit, returns page text
    - `async desconectar() -> None` — closes browser context

- [ ] **Step 1: Add mock browser fixture to `tests/conftest.py`**

A `mock_page` fixture that returns an `AsyncMock` simulating a Playwright page: `goto`, `query_selector_all`, `inner_html`, `locator`, `set_input_files`, `click`, `wait_for_load_state`, `content`. A `mock_browser` fixture that patches `playwright.async_api.async_playwright` to return a mock whose `chromium.connect_over_cdp` returns a mock browser with one mock page.

- [ ] **Step 2: Write failing tests in `tests/test_browser_controller.py`**

Test `conectar` succeeds with mock CDP. Test `conectar` raises `ConnectionError` when CDP connect raises. Test `navegar_para_processo` calls `page.goto` with the URL from `montar_url_processo`. Test `capturar_formulario` returns the HTML of the form element. Test `preencher_campo` with `tipo="text"` calls `fill`, with `tipo="select"` calls `select_option`. Test `fazer_upload` calls `set_input_files` with the file paths. Test `desconectar` closes context. **Review Focus #4:** Test that `conectar` with unreachable CDP raises `ConnectionError` with message mentioning port 9222.

- [ ] **Step 3: Run tests — expect FAIL**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_browser_controller.py -v`

- [ ] **Step 4: Implement `src/browser_controller.py`**

`BrowserController` class. `conectar` wraps `async_playwright().start()` then `chromium.connect_over_cdp(cdp_url)` in try/except, raising `ConnectionError` with user-friendly message on failure. `capturar_formulario` finds all `<form>` elements and returns the `outer_html()` of the longest one. `preencher_campo` dispatches on `tipo`: `"select"` uses `select_option`, `"checkbox"` uses `set_checked`, everything else uses `fill`. `fazer_upload` uses `set_input_files`.

- [ ] **Step 5: Run tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_browser_controller.py -v`

- [ ] **Step 6: Commit**

```bash
git add src/browser_controller.py tests/test_browser_controller.py tests/conftest.py
git commit -m "feat: add browser controller with Playwright CDP connection"
```

---

### Task 3: AI Interpreter

**Files:**
- Create: `src/ai_interpreter.py`
- Create: `tests/test_ai_interpreter.py`
- Modify: `tests/conftest.py` (add mock Anthropic client fixture)

**Interfaces:**
- Consumes: `PropostaPreenchimento`, `CampoSugerido`, `UploadSugerido` from `src/models.py`
- Produces:
  - `async analisar_formulario(html_formulario: str, nomes_documentos: list[str]) -> PropostaPreenchimento` — sends form HTML and document names to Claude, returns structured proposal

- [ ] **Step 1: Add mock Anthropic fixture to `tests/conftest.py`**

A `mock_anthropic` fixture that patches `anthropic.AsyncAnthropic` and makes `messages.create` return a mock response whose `content[0].text` is a JSON string matching the `PropostaPreenchimento` schema.

- [ ] **Step 2: Write failing tests in `tests/test_ai_interpreter.py`**

Test `analisar_formulario` with a simple HTML form string and a list `["peticao.pdf", "procuracao.pdf"]` returns a `PropostaPreenchimento` with at least one `CampoSugerido` and one `UploadSugerido`. Test that the prompt sent to Claude contains the HTML and the document names. Test that malformed JSON from Claude raises `ValueError` with a descriptive message. **Review Focus #2:** Test that when Claude returns a field with `confianca: "baixa"`, the field appears in the result with that confidence level preserved.

- [ ] **Step 3: Run tests — expect FAIL**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_ai_interpreter.py -v`

- [ ] **Step 4: Implement `src/ai_interpreter.py`**

`analisar_formulario` builds a prompt instructing Claude to: examine the form HTML, identify all fillable fields (inputs, selects, textareas, file uploads), and return a JSON matching `PropostaPreenchimento`. The prompt includes the document file names so Claude can map them to upload fields and suggest ordering. Parse Claude's response as JSON, validate with Pydantic, return the model. On parse failure, raise `ValueError`.

- [ ] **Step 5: Run tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_ai_interpreter.py -v`

- [ ] **Step 6: Commit**

```bash
git add src/ai_interpreter.py tests/test_ai_interpreter.py tests/conftest.py
git commit -m "feat: add AI interpreter for form analysis via Claude API"
```

---

### Task 4: Frontend (HTML/CSS/JS)

**Files:**
- Create: `static/index.html`
- Create: `static/style.css`
- Create: `static/app.js`

**Interfaces:**
- Consumes:
  - `POST /api/protocolar` — `{numero_processo, pasta_documentos, plataforma}` returns `{task_id}`
  - `GET /api/status/{task_id}` — SSE stream of `TaskStatus` JSON events
  - `POST /api/confirmar/{task_id}` — `{campos_editados?: CampoSugerido[]}` returns `{ok: true}`
  - `POST /api/cancelar/{task_id}` — returns `{ok: true}`
  - `GET /api/historico` — returns `HistoricoItem[]`
  - `GET /api/plataformas` — returns `{key, nome}[]`
- Produces: User-facing UI (no programmatic interface consumed by other tasks)

- [ ] **Step 1: Create `static/index.html`**

Single HTML page with: header "Protocolo de Processos", form section (numero_processo text input, pasta_documentos text input, plataforma dropdown populated from `/api/plataformas` on load, "Iniciar" button), status panel (hidden by default, shows step-by-step progress), confirmation panel (hidden, shows table of proposed fields with editable value column, file list, "Confirmar"/"Cancelar" buttons), history section (table of past protocols). Fields with `confianca: "baixa"` get a yellow highlight in the confirmation table.

- [ ] **Step 2: Create `static/style.css`**

Clean, minimal styling. Status panel with colored step indicators (gray=pending, blue=active, green=done, red=error). Confirmation table with editable cells. Low-confidence fields highlighted yellow. Responsive layout.

- [ ] **Step 3: Create `static/app.js`**

Functions: `carregarPlataformas()` fetches `/api/plataformas` and fills dropdown. `iniciarProtocolo()` posts to `/api/protocolar`, then opens SSE to `/api/status/{task_id}`. SSE `onmessage` parses `TaskStatus` JSON: updates status panel steps; when status is `aguardando_confirmacao`, shows confirmation panel populated from `proposta`. `confirmar()` collects any edited values from the table and posts to `/api/confirmar/{task_id}`. `cancelar()` posts to `/api/cancelar/{task_id}`. `carregarHistorico()` fetches and renders history table. **Review Focus #5:** On SSE error, reconnect with `EventSource` after 2s, and on reconnect the status endpoint replays the current state.

- [ ] **Step 4: Verify by opening `static/index.html` in a browser**

Confirm all elements render, dropdown placeholder works, panels toggle visibility with JS. API calls will fail (backend not running) — visual check only.

- [ ] **Step 5: Commit**

```bash
git add static/
git commit -m "feat: add frontend with form, status panel, confirmation UI, and history"
```

---

### Task 5: Flow Orchestrator and API Endpoints

**Files:**
- Create: `src/flow_orchestrator.py`
- Create: `src/main.py`
- Create: `tests/test_flow_orchestrator.py`
- Create: `tests/test_api.py`

**Interfaces:**
- Consumes:
  - `BrowserController` from `src/browser_controller.py` — `conectar`, `navegar_para_processo`, `capturar_formulario`, `preencher_campo`, `fazer_upload`, `clicar_botao`, `capturar_confirmacao`, `desconectar`
  - `analisar_formulario` from `src/ai_interpreter.py`
  - All models from `src/models.py`
  - `get_plataforma`, `listar_plataformas`, `montar_url_processo` from `src/plataformas.py`
- Produces:
  - `FlowOrchestrator` class with:
    - `__init__(browser: BrowserController, interpreter_fn)` — dependency injection
    - `async iniciar(request: ProtocolarRequest) -> str` — starts flow, returns task_id
    - `async get_status(task_id: str) -> TaskStatus` — returns current status
    - `async confirmar(task_id: str, campos_editados: list[CampoSugerido] | None) -> None` — resumes after confirmation
    - `async cancelar(task_id: str) -> None` — cancels flow
    - `get_historico() -> list[HistoricoItem]` — returns history
  - FastAPI app at `src/main.py` with endpoints per spec

- [ ] **Step 1: Write failing tests for `FlowOrchestrator` in `tests/test_flow_orchestrator.py`**

Use mock `BrowserController` and mock `analisar_formulario`. Test `iniciar` returns a task_id string and sets status to `"navegando"`. Test that after `iniciar` completes navigation+analysis, status becomes `"aguardando_confirmacao"` with a proposta. Test `confirmar` triggers browser fill and sets status to `"preenchendo"` then `"concluido"`. Test `cancelar` sets status to `"cancelado"`. **Review Focus #1:** Test that calling `cancelar` while status is `"preenchendo"` stops execution and does not call `clicar_botao` (the submit button). **Review Focus #3:** Test that `iniciar` with a `pasta_documentos` path containing no files raises `ValueError` with message about empty folder.

- [ ] **Step 2: Run tests — expect FAIL**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_flow_orchestrator.py -v`

- [ ] **Step 3: Implement `src/flow_orchestrator.py`**

`FlowOrchestrator` manages a dict of `task_id -> TaskStatus` and a dict of `task_id -> asyncio.Event` for the confirmation gate. `iniciar` creates task, spawns an `asyncio.Task` running `_executar_fluxo`. `_executar_fluxo`: set status `navegando` -> call `browser.navegar_para_processo` -> set status `analisando` -> call `browser.capturar_formulario` -> call `interpreter_fn` -> set status `aguardando_confirmacao` with proposta -> await confirmation Event -> if confirmed: set status `preenchendo`, iterate `proposta.campos` calling `browser.preencher_campo` (checking cancelled flag before each), call `browser.fazer_upload`, call `browser.clicar_botao` for submit, call `browser.capturar_confirmacao`, set status `concluido` with protocolo. On any exception: set status `erro`. `confirmar` sets the Event, optionally replacing proposta campos with edited values. `cancelar` sets a cancelled flag and sets the Event so the flow unblocks and exits. Before `iniciar` navigates, validate that `pasta_documentos` exists and contains at least one file — raise `ValueError` if not. History stored as a list, appended on `concluido`, `erro`, or `cancelado`.

- [ ] **Step 4: Run tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_flow_orchestrator.py -v`

- [ ] **Step 5: Write failing tests for API endpoints in `tests/test_api.py`**

Use `httpx.AsyncClient` with FastAPI's `TestClient`. Test `POST /api/protocolar` returns `{task_id}`. Test `GET /api/status/{task_id}` returns SSE events. Test `POST /api/confirmar/{task_id}` returns `{ok: true}`. Test `POST /api/cancelar/{task_id}` returns `{ok: true}`. Test `GET /api/historico` returns list. Test `GET /api/plataformas` returns list with `key` and `nome`. Test `POST /api/protocolar` with invalid plataforma returns 400. **Review Focus #4 (endpoint level):** Test that when browser connection fails, `POST /api/protocolar` returns 503 with message about Chrome CDP.

- [ ] **Step 6: Run tests — expect FAIL**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/test_api.py -v`

- [ ] **Step 7: Implement `src/main.py`**

FastAPI app. Mount `static/` as static files. On startup, create `BrowserController` and `FlowOrchestrator`. Endpoints: `POST /api/protocolar` validates request, calls `orchestrator.iniciar`, returns task_id. `GET /api/status/{task_id}` returns SSE (`EventSourceResponse` from `sse-starlette` — add to requirements.txt) that polls `orchestrator.get_status` every 500ms and yields JSON. `POST /api/confirmar/{task_id}` calls `orchestrator.confirmar`. `POST /api/cancelar/{task_id}` calls `orchestrator.cancelar`. `GET /api/historico` returns `orchestrator.get_historico()`. `GET /api/plataformas` returns `listar_plataformas()`. Root `/` serves `static/index.html`.

- [ ] **Step 8: Run all tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/ -v`

- [ ] **Step 9: Commit**

```bash
git add src/flow_orchestrator.py src/main.py tests/test_flow_orchestrator.py tests/test_api.py requirements.txt
git commit -m "feat: add flow orchestrator and FastAPI endpoints with SSE status"
```

---

### Task 6: Integration Test and README

**Files:**
- Create: `README.md`
- Modify: `tests/test_api.py` (add end-to-end flow test)

**Interfaces:**
- Consumes: all modules
- Produces: documentation and an E2E test proving the full flow

- [ ] **Step 1: Write end-to-end test in `tests/test_api.py`**

A test `test_fluxo_completo` that: posts to `/api/protocolar` with a mock folder path, reads SSE events until `aguardando_confirmacao`, verifies the proposta has campos and uploads, posts to `/api/confirmar/{task_id}`, reads SSE events until `concluido`, then hits `/api/historico` and verifies the entry exists. All with mocked browser and Claude.

- [ ] **Step 2: Run tests — expect PASS**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/ -v`

- [ ] **Step 3: Write `README.md`**

Sections: what this does (3 sentences), prerequisites (Python 3.11+, Chrome, Anthropic API key), install (`pip install -r requirements.txt && playwright install chromium`), usage (start Chrome with `--remote-debugging-port=9222`, set `ANTHROPIC_API_KEY`, run `python -m uvicorn src.main:app`, open `http://localhost:8000`), how it works (4-step summary: inform process → AI proposes → you confirm → it files).

- [ ] **Step 4: Run full test suite one last time**

Run: `cd /home/user/protocolo-processos && python -m pytest tests/ -v --tb=short`

- [ ] **Step 5: Commit**

```bash
git add README.md tests/test_api.py
git commit -m "feat: add end-to-end test and README with usage instructions"
```
