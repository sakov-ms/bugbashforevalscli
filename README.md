# Bug Bash — PR #360 (Retrieval Evaluators) on Zava Insurance Claims Agent

End-to-end guide to set up and run the bug bash for **[microsoft/M365-Copilot-Agent-Evals#360](https://github.com/microsoft/M365-Copilot-Agent-Evals/pull/360)** — which adds the new `RetrievalQuery` and `RetrievalResult` evaluators — against the **Zava Insurance Claims** declarative agent.

This README walks you from zero (clone the agent, install the eval CLI) to running an evaluation and viewing an HTML report.

---

## 1. Folder layout (after setup)

You will end up with two sibling folders under one working directory (e.g. `C:\Users\<you>\bootcamp\`):

```
bootcamp/
├── zava-insurance-claims/         ← the agent project (provision to your tenant)
└── evals cli testing/             ← the eval CLI workspace (where you run tests)
    ├── package/                   ← unpacked CLI source
    │   └── src/clients/cli/main.py
    ├── env/
    │   └── .env.local             ← your secrets / config (NOT committed)
    ├── evals/                     ← the 10 dataset JSON files (this repo's datasets/)
    └── .evals/                    ← HTML / JSON / CSV reports go here
```

The 10 prioritized datasets in this `bugbash/datasets/` folder are what you copy into the CLI's `evals/` folder.

---

## 2. Get the Zava Insurance Claims agent

Clone (or download as zip) the agent project. It contains the declarative agent manifest, MCP plugin manifest, and SharePoint knowledge source configuration.

```powershell
cd C:\Users\<you>\bootcamp
git clone <repo-url-for-zava-insurance-claims> zava-insurance-claims
# or copy the folder from this repo: bugbash/zava-insurance-claims/
```

### Provision the agent to your tenant

From the agent folder, sign in to M365 and provision a local-mode build:

```powershell
cd zava-insurance-claims
npx -y --package @microsoft/m365agentstoolkit-cli atk auth login m365
npx -y --package @microsoft/m365agentstoolkit-cli atk provision --env local
```

> ⚠️ The agent's `declarativeAgent.json` points to a SharePoint site at `https://microsoft.sharepoint-df.com/teams/ZavaClaims/Shared Documents`. If your account does not have access, edit the `oneDriveAndSharePoint` capability in `appPackage/declarativeAgent.json` to point at your own SharePoint site (containing the Zava claims policy guidebook) and re-provision.

---

## 3. Get the eval CLI (PR #360 build)

The CLI is published as an npm-style tarball (`.tgz`) containing the Python CLI source plus a Node wrapper.

1. Download the tarball from the PR (or your team's distribution channel). Example file name: `microsoft-m365-copilot-eval-1.8.0-preview.1.tgz`.
2. Create a workspace folder and put the tarball inside:

   ```powershell
   cd C:\Users\<you>\bootcamp
   mkdir "evals cli testing"
   cd "evals cli testing"
   # Drop the .tgz file here, then:
   ```
3. Extract and install:

   ```powershell
   # 3a. Initialize an npm workspace and install the tarball
   npm init -y
   npm install .\microsoft-m365-copilot-eval-1.8.0-preview.1.tgz

   # 3b. The Python CLI source lives at: package\src\clients\cli\main.py
   #     Install the Python dependencies it needs:
   python -m pip install --upgrade pip
   python -m pip install python-dotenv azure-ai-evaluation requests msal markdown jsonschema
   ```

   > If you have a `requirements.txt` inside `package/`, prefer: `python -m pip install -r .\package\requirements.txt`

---

## 4. Configure environment variables (`env/.env.local`)

Create `env\.env.local` in your `evals cli testing` folder with the following keys. **All are required** — the CLI fails fast if any are missing.

```dotenv
# --- Azure OpenAI (judge model used by Relevance / Coherence / Groundedness / Similarity) ---
AZURE_AI_OPENAI_ENDPOINT=https://<your-aoai-resource>.openai.azure.com/
AZURE_AI_API_KEY=<your-azure-openai-api-key>
AZURE_AI_API_VERSION=2024-08-01-preview
AZURE_AI_MODEL_NAME=<your-deployment-name>          # e.g. gpt-4o, gpt-4o-mini

# --- WorkIQ A2A endpoint (the bridge that calls your M365 agent and returns retrieval artifacts) ---
WORK_IQ_A2A_ENDPOINT=https://<workiq-a2a-host>/    # provided by the WorkIQ A2A team
WORK_IQ_A2A_CLIENT_ID=<entra-app-client-id>        # Entra app used to acquire a token for the A2A endpoint
WORK_IQ_A2A_SCOPES=api://<workiq-a2a-app-id>/.default
TENANT_ID=<your-entra-tenant-id-guid>              # tenant where the agent is provisioned
```

### What each variable does

| Variable | Used for | Where to get it |
|---|---|---|
| `AZURE_AI_OPENAI_ENDPOINT` | Base URL of your Azure OpenAI resource — the LLM-judge evaluators (`Relevance`, `Coherence`, `Groundedness`, `Similarity`) call this. | Azure Portal → Azure OpenAI resource → *Keys and Endpoint*. |
| `AZURE_AI_API_KEY` | Auth for the Azure OpenAI judge calls. | Same place as endpoint. |
| `AZURE_AI_API_VERSION` | Azure OpenAI REST API version (e.g. `2024-08-01-preview`). | Azure OpenAI docs / preview tracker. |
| `AZURE_AI_MODEL_NAME` | The **deployment name** of the chat model the evaluators will use (NOT the underlying model id). | Azure OpenAI Studio → *Deployments*. |
| `WORK_IQ_A2A_ENDPOINT` | Where the CLI sends the agent prompt; the WorkIQ A2A service relays to M365 Copilot and returns the response **plus** the retrieval artifact (`application/vnd.ms-workiq-internal.retrieval`) that the new `RetrievalQuery` / `RetrievalResult` evaluators consume. | Provided by the WorkIQ A2A team for your tenant. |
| `WORK_IQ_A2A_CLIENT_ID` | Entra app client id used to acquire a bearer token to call the A2A endpoint. | Entra Portal → *App registrations*. |
| `WORK_IQ_A2A_SCOPES` | OAuth scope string for the A2A app (typically `api://<a2a-app-id>/.default`). | The A2A team / Entra app *Expose an API* tab. |
| `TENANT_ID` | Your Entra tenant id (a GUID). Used for both auth and routing. | Entra Portal → *Overview* → *Tenant ID*. |

> 🔒 Keep `.env.local` out of source control (`.gitignore` it). Treat the API key, client id, and scopes as secrets.

---

## 5. Copy the datasets into the CLI workspace

This `bugbash/datasets/` folder contains the 10 prioritized test JSON files (8 RAG + 2 MCP). Copy them next to the CLI:

```powershell
$src  = "C:\Users\<you>\bootcamp\bugbash\datasets"
$dest = "C:\Users\<you>\bootcamp\evals cli testing\evals"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Get-ChildItem $src -Recurse -Filter *.json | Copy-Item -Destination $dest -Force
```

You should now see 10 files under `evals cli testing\evals\` (file names like `rag-01-mortgage-joint-check-threshold.json`, `mcp-04-show-claim-detail-by-number.json`, etc.).

Also create the report output folder:

```powershell
New-Item -ItemType Directory -Force -Path "C:\Users\<you>\bootcamp\evals cli testing\.evals" | Out-Null
```

---

## 6. Run an evaluation

From `C:\Users\<you>\bootcamp\evals cli testing\` run:

```powershell
python -m dotenv -f .\env\.env.local run -- python .\package\src\clients\cli\main.py --prompts-file ".\evals\rag-01-mortgage-joint-check-threshold.json" --output ".\.evals\rag-01-report.html"
```

### What every part of this command does

| Token | What it means |
|---|---|
| `python -m dotenv` | Invokes the [`python-dotenv`](https://pypi.org/project/python-dotenv/) CLI as a module. We use the module form because the `dotenv` script may not be on Windows `PATH`. |
| `-f .\env\.env.local` | Tells `dotenv` to load environment variables from `env\.env.local` (not the default `.env` in cwd). |
| `run --` | Tells `dotenv` to execute the command that follows **with the loaded env vars injected** into the subprocess. The `--` terminates `dotenv`'s flag parsing so everything after it is the actual command. |
| `python .\package\src\clients\cli\main.py` | The eval CLI entry point (the unpacked tarball's Python source). |
| `--prompts-file ".\evals\rag-01-mortgage-joint-check-threshold.json"` | Path to a v1.5.0 eval document containing one or more prompts plus their evaluator configurations. You can use any file from `evals\`. |
| `--output ".\.evals\rag-01-report.html"` | Where to write the result. **File extension drives the format**: `.html` → styled HTML report (summary banner + aggregates + per-prompt cards), `.json` → raw JSON, `.csv` → flat CSV. If `--output` is omitted, results print to console. |

Open the generated `.evals\rag-01-report.html` in your browser to see results.

### Run a different test

Swap the `--prompts-file` and `--output` filenames:

```powershell
python -m dotenv -f .\env\.env.local run -- python .\package\src\clients\cli\main.py --prompts-file ".\evals\rag-07-failure-no-matching-query.json" --output ".\.evals\rag-07-report.html"
```

### Run all 10 datasets in one batch (each produces its own HTML report)

```powershell
$timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
Get-ChildItem .\evals\rag-*.json, .\evals\mcp-*.json | ForEach-Object {
    $out = ".\.evals\$timestamp-$([IO.Path]::GetFileNameWithoutExtension($_.Name)).html"
    Write-Host "`n=== Running $($_.Name) -> $out ===" -ForegroundColor Cyan
    python -m dotenv -f .\env\.env.local run -- python .\package\src\clients\cli\main.py --prompts-file $_.FullName --output $out
}
```

---

## 7. The 10 prioritized datasets

PR #360 is about retrieval evaluators, so 8 of 10 slots target RAG. The 2 MCP slots cover tool-call smoke + scope guardrail.

### RAG (`evals/rag-*.json`) — 8 tests

| Priority | File | What it covers | Expected outcome |
|---|---|---|---|
| **P1** | `rag-01-mortgage-joint-check-threshold.json` | Happy `RetrievalQuery` + `RetrievalResult` baseline — mortgage joint-check threshold ($10K) from §8.4 | **Pass** |
| **P1** | `rag-07-failure-no-matching-query.json` | Failure code `no_matching_query` | **Fail** (intended) |
| **P1** | `rag-08-failure-required-terms-missing.json` | Failure code `required_terms_missing`; `includes_missing=['unicorn']` | **Fail** (intended) |
| **P1** | `rag-09-failure-excluded-terms-found.json` | Failure code `excluded_terms_found`; `excludes_found=['claim']` | **Fail** (intended) |
| **P1** | `rag-10-failure-not-retrieved.json` | Failure code `not_retrieved` (or `error` if capability had no hits) | **Fail** (intended) |
| **P2** | `rag-03-deductibles-min-count.json` | `min_expected_count` count-only path (§2 Definitions + §6.1 HomeShield) | **Pass** |
| **P2** | `rag-11-max-rank-boundary-strict.json` | `max_rank: 1` boundary — Claims Timeline (§8.3) | **Fail** (likely) |
| **P2** | `rag-12-max-rank-boundary-relaxed.json` | Same prompt as rag-11, `max_rank: 20` — proves `max_rank` is honored | **Pass** |

### MCP (`evals/mcp-*.json`) — 2 tests

| Priority | File | What it covers |
|---|---|---|
| **P3** | `mcp-04-show-claim-detail-by-number.json` | `show-claim-detail` smoke test — verifies MCP tool invocation + groundedness on claim `CN202504990` |
| **P3** | `mcp-13-scope-decline-hr.json` | Negative — agent must **decline** an out-of-scope HR question and NOT invoke any tool |

> Extract phrases in `retrievalExtract_contains` are pre-filled with verbatim text from the actual SharePoint doc `2026-03-25-zava-claims-insurance-policy-guidebook.docx`. If you swap to your own SharePoint site, copy equivalent verbatim phrases from your version of the doc and update the JSON.

---

## 8. What to report when filing bugs

For each finding include:
- The dataset file (e.g. `rag-08-failure-required-terms-missing.json`)
- The full evaluator output JSON (`matched_items`, `missing_items`, `extract_failures`, `matched_queries`, `includes_missing`, `excludes_found`)
- The CLI tarball version (e.g. `microsoft-m365-copilot-eval-1.8.0-preview.1`)
- A screenshot of the HTML report card (`.evals\*.html`) for the failing prompt

---

## 9. Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `ERROR Missing required environment variables: …` | `.env.local` not loaded. Make sure you used `python -m dotenv -f .\env\.env.local run --` and the file exists at that exact path. |
| `Schema validation error: Document validation failed` | The dataset has a field the schema rejects (e.g. `id` at item root, or an unknown evaluator like `ToolCallAccuracy`). Move custom keys under `extensions`; only use evaluators in `EvaluatorMap` (Relevance, Coherence, Groundedness, Similarity, Citations, ExactMatch, PartialMatch, RetrievalQuery, RetrievalResult). |
| `The term 'dotenv' is not recognized` | The `dotenv` script isn't on PATH. Use the module form: `python -m dotenv …` (or `pip install "python-dotenv[cli]"`). |
| HTML report is empty / no aggregates | Agent didn't respond. Check the WorkIQ A2A endpoint reachability, token acquisition, and that the agent is provisioned in the same tenant as `TENANT_ID`. |

---

## 10. Cleanup

```powershell
Remove-Item -Recurse -Force .\.evals
```
