# Agentic Analyser — Project Details

## 1. Project summary

**Agentic Analyser** is a local Python web application for exploring CSV and Excel datasets with an AI-assisted analysis workflow. It also includes an interactive data-preparation workspace. The browser UI is served by Flask, data work is performed with pandas and Python, and language-model requests are sent to OpenRouter.

The application has two main pages:

- **Analyse** (`/`): upload one or more datasets, choose a question, receive AI-generated Python analysis, see progress and outputs, review a structured report and charts, ask follow-up questions, and export the report.
- **Preprocess** (`/preprocess`): profile one dataset, configure cleaning and feature transformations, preview the resulting data, and download or pass it back to Analyse.

The app is designed to run on the user's machine at `http://localhost:5050`. It is not a hosted multi-user service.

## 2. Technology

| Area | Technology |
| --- | --- |
| Web server and HTTP API | Python, Flask, Flask-CORS, Waitress (when installed) |
| Data loading and transformations | pandas, NumPy, SciPy |
| AI requests | OpenRouter Chat Completions API through `requests` |
| Frontend | Plain HTML, CSS, and JavaScript; no React build step |
| Charts | Matplotlib and Seaborn |
| Local persistence | SQLite (`.data/agentic.db`) |
| Report exports | ReportLab for PDF; openpyxl for Excel |

Dependencies are listed in `requirements.txt`. Versions are currently unpinned.

## 3. Repository map

| File or folder | Purpose |
| --- | --- |
| `server.py` | Flask application, page serving, API routes, upload handling, streaming, exports, and session coordination. |
| `agent.py` | OpenRouter prompts and requests, analysis loop, follow-up flow, result handling, and chart collection. |
| `config.py` | Loads `.env`, reads the API key and model settings, and defines session helpers. |
| `utils.py` | Dataset loading, dataset summaries, code-block parsing, Server-Sent Event formatting, and plot-file helpers. |
| `executor.py` | Writes model-generated Python to a temporary script, runs it in a subprocess, and returns its output. |
| `preprocessor.py` | Dataset profiling, transformation functions, pipeline ordering, and reproducible script generation. |
| `database.py` | SQLite schema and access helpers for analysis history, reports, plots, and preprocessing logs. |
| `pdf.py` | Builds an analysis PDF with report text and charts. |
| `excel_export.py` | Builds a formatted analysis workbook with report and chart content. |
| `static/index.html` | Analyse page markup, styles, and browser-side upload, streaming, history, and export logic. |
| `static/preprocess.html` | Preprocess page markup, styles, profiling display, transformation controls, preview, and downloads. |
| `static/favicon.png` | Browser icon. |
| `requirements.txt` | Python dependencies. |
| `env.example` | Example environment file; replace its placeholder API key before use. |
| `SETUP_GUIDE.txt` | Original setup instructions. |
| `start.sh` | Unix-style convenience script that installs dependencies and starts the server. |
| `.data/` | Runtime SQLite database; created automatically and ignored by Git. |
| `.env` | Local secrets and model overrides; create it locally and keep it private. |

## 4. Setup and run

### Requirements

- Python 3.10 or newer.
- An OpenRouter API key for AI suggestions and analysis.
- Internet access for OpenRouter requests.

### Windows PowerShell

From the project folder:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Create `.env` in the project root. It can be based on `env.example`; set the key and model values:

```dotenv
OPENROUTER_API_KEY=your-openrouter-key
OPENROUTER_MODEL=nvidia/nemotron-3-ultra-550b-a55b:free
OPENROUTER_FALLBACK_MODEL=nvidia/nemotron-3.5-lightning:free
```

Start the app:

```powershell
python server.py
```

Open [http://localhost:5050](http://localhost:5050). Stop the server with **Ctrl+C** in its terminal. After frontend edits, refresh the browser with **Ctrl+F5**.

### macOS or Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python server.py
```

The `start.sh` script is an alternative convenience entry point on Unix-like systems. It runs `pip install -r requirements.txt` and then `python server.py`; a virtual environment is still recommended.

### Server behavior

The server uses Waitress on `0.0.0.0:5050` if Waitress is installed. Otherwise it falls back to Flask's threaded development server. The startup routine attempts a small OpenRouter model check when an API key is configured.

## 5. Model configuration and response timing

`config.py` loads variables from `.env` using `python-dotenv`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `OPENROUTER_API_KEY` | Empty | Required for model-backed features. |
| `OPENROUTER_MODEL` | `nvidia/nemotron-3-ultra-550b-a55b:free` | Primary model. An environment value overrides the default. |
| `OPENROUTER_FALLBACK_MODEL` | `nvidia/nemotron-3.5-lightning:free` | Fallback model after a primary request error or timeout. |

Standard model requests use a timeout of 18 seconds for the primary and 25 seconds for the fallback, with one attempt per model. Suggestion requests use 15 and 20 seconds. The Analyse page offers Quick (2 steps), Balanced (3 steps), and Detailed (5 steps); the server clamps requested analysis depth to 2–5 steps. A step can include an OpenRouter call and local Python execution, so total duration depends on the model service and dataset.

## 6. Analyse workflow

1. The browser uploads one or more CSV or Excel files. The application reads them into pandas DataFrames. Excel support uses `openpyxl` and `xlrd` where appropriate.
2. The `/columns` endpoint returns column names and inferred data types for the first file. The `/suggest` endpoint asks the configured model for suggested questions. If the model fails or returns unusable output, the server constructs questions from the column names and types.
3. When the user starts an analysis, `/analyse` loads the file(s), creates temporary CSV copies, builds a summary, creates an in-memory session, and starts the agent in a background thread.
4. The model receives the question, data shape, column names and types, null counts, numeric summary, and a sample of the first three rows. It can request Python code to inspect the local temporary CSV and calculate requested results.
5. Model responses and tool outputs are streamed to the browser using Server-Sent Events. The live board displays elapsed time, current step, tool-run count, chart count, and workflow stage. Iteration cards show the model's analysis and Python output.
6. The agent returns a structured Markdown report with key findings, trends, and recommendations. The UI can display charts, accept follow-up questions, and export a PDF or Excel report.

The analysis prompt asks the model to use observed values and follow a report structure. The output still depends on model behavior and should be reviewed.

## 7. Preprocess workflow

The Preprocess page accepts one CSV or Excel file. It creates a column profile, shows a sample preview, and offers smart recommendations. Profiling samples up to 5,000 rows for some numeric statistics on larger datasets; row count, null counts, and unique counts are calculated from the full dataset.

Available pipeline operations include:

| Operation | Options / examples |
| --- | --- |
| Missing values | Drop affected rows; fill with mean, median, mode, forward-fill, backward-fill, or a constant. |
| Type conversion | Numeric, datetime, string, boolean, or category. |
| Outliers | IQR or Z-score bounds; remove, cap, or keep flagged values. |
| Encoding | Label, one-hot, target encoding, or drop a selected encoded column. |
| Scaling | Standard, min-max, robust, or no scaling. |
| Feature engineering | Datetime components, log/square-root transforms, arithmetic interaction features, duplicate removal, column removal/rename, low-variance removal, and high-null-column removal. |

Pipeline order is: selected early feature rules (drop/rename/duplicates/null or variance filtering), missing-value handling, type conversion, outlier handling, derived feature creation, encoding, then scaling. Applying the pipeline updates the working CSV, returns an updated profile and preview, and records a transformation log plus generated Python code.

The cleaned data can be sent to Analyse or downloaded as CSV, Excel, JSON, or HTML. The code-generation endpoint is available for a reproducible preprocessing script. JSON and HTML conversion currently parse CSV text in browser-side code; complex quoted CSV fields containing commas or embedded newlines may not round-trip correctly in those two formats. CSV and Excel are the safer exports for such data.

## 8. HTTP API reference

All endpoints are served by the local Flask application. JSON endpoints return a JSON object; uploads use `multipart/form-data`; analysis and follow-up responses use `text/event-stream`.

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/` | Serve the Analyse page. |
| `GET` | `/preprocess` | Serve the Preprocess page. |
| `GET` | `/model` | Return the configured primary model name. |
| `GET` | `/static/<path>` | Serve static assets. |
| `GET` | `/favicon.ico` | Serve the favicon. |
| `POST` | `/columns` | Read an uploaded file and return columns and dtypes. |
| `POST` | `/suggest` | Return AI-generated or column-derived analysis questions. |
| `POST` | `/analyse` | Upload data and stream an analysis run. Form fields include files, question, and `max_iterations`. |
| `POST` | `/followup` | Stream a follow-up answer for an in-memory analysis session. |
| `POST` | `/export_pdf` | Return a PDF report as base64 JSON. |
| `POST` | `/export_excel` | Return an analysis workbook as base64 JSON. |
| `POST` | `/prep/profile` | Upload and profile a dataset for preprocessing. |
| `POST` | `/prep/apply` | Apply a transformation pipeline and return updated profile/preview. |
| `POST` | `/prep/download_csv` | Return the current cleaned CSV as base64 JSON. |
| `POST` | `/prep/export_xlsx` | Convert an uploaded CSV to a formatted Excel workbook. |
| `POST` | `/prep/download_code` | Return generated preprocessing code as base64 JSON. |
| `GET` | `/prep/pending` | Return the latest processed in-memory preprocessing session. |
| `GET` | `/prep/get_csv/<session_id>` | Download a cleaned CSV for an in-memory preprocessing session. |
| `POST` | `/prep/full_log` | Return the transformation log for an in-memory preprocessing session. |
| `GET` | `/analyse/get_csv/<session_id>` | Download the original dataset associated with an analysis session when its temporary file is available. |
| `GET` | `/sessions/list` | List up to ten recent saved analysis reports. |
| `GET` | `/sessions/load/<session_id>` | Load a saved report and charts from SQLite. |
| `DELETE` | `/sessions/delete/<session_id>` | Delete a saved analysis session from SQLite. |

## 9. Persistence and session lifetime

- SQLite is stored at `.data/agentic.db`; the directory and schema are initialized automatically.
- Completed analysis history stores report text, dataset name/path metadata, and charts. The browser history panel lists up to ten recent reports.
- Uploaded analysis and preprocessing files are kept as temporary CSV copies. Original/cleaned CSV handoff and preprocessing API calls rely on the current process's in-memory session maps and temporary files.
- Consequently, saved analysis reports can be restored from SQLite after restart, but in-memory follow-up context and some dataset/preprocessing handoff operations may not survive a server restart or temporary-file cleanup.
- The server removes database sessions older than seven days during startup cleanup.

## 10. Security, privacy, and operational limits

- The app has no login or user authorization. It binds to `0.0.0.0` when started, which may make it reachable by other devices on the local network. Use it only on a trusted machine/network and do not expose port 5050 publicly without adding authentication and request protections.
- The API key belongs in `.env`, which is ignored by Git. Do not paste it into source code, project documentation, or commits.
- Dataset summaries, the requested question, and code-execution outputs are sent to OpenRouter for AI analysis. Avoid sending data that must remain entirely local.
- Analysis code returned by the model is executed by `executor.py` in a regular local Python subprocess with a 30-second timeout. It is **not an operating-system sandbox** and is not restricted to the uploaded CSV. Only run this application with data and prompts you trust; do not expose the analysis endpoint to untrusted users.
- The application is a local prototype, not a hardened multi-user or production service. It has no authentication, quotas, or configured upload-size limits.

## 11. Troubleshooting

| Symptom | Check |
| --- | --- |
| `ModuleNotFoundError` | Activate `.venv` and run `pip install -r requirements.txt`. |
| Python command not found | Install Python 3.10+ and add it to `PATH`; reopen the terminal. |
| OpenRouter reports authentication/model errors | Check `.env`, ensure the key is valid, and confirm the primary/fallback model IDs are enabled for the account. Restart the server after editing `.env`. |
| Suggestions are blank | Confirm the API key and server terminal logs; column-derived suggestions are used when the model response is unavailable. |
| Port 5050 is occupied | Stop the other server process or change the port in the server startup code. |
| Old interface remains after editing HTML | Restart `python server.py`, then hard-refresh the browser with `Ctrl+F5`. |
| Excel import fails | Confirm `openpyxl` for `.xlsx` and `xlrd` for `.xls` are installed. |

## 12. Current project status

The current frontend includes a redesigned dark dashboard, live analysis status board, selectable analysis depth, suggested questions, dataset handoff between tabs, and animated liquid-metal-style buttons implemented in CSS. The project remains a single-process Flask application with in-memory active sessions and SQLite-backed report history.
