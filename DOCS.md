# CLI Agent (AI Terminal Assistant) — Technical Documentation

> **Version:** 1.1.1  
> **Repository:** [Ayushsingh-02082004/ai-terminal-assistant](https://github.com/Ayushsingh-02082004/ai-terminal-assistant)  
> **Web Portal:** [cli-agent-website.vercel.app](https://cli-agent-website.vercel.app/)

---

## 1. Executive Summary

**CLI Agent** is an intelligent, autonomous Command Line Interface (CLI) assistant built using **CrewAI**, **LiteLLM / Ollama Cloud / Gemini / OpenAI**, and **Python**. It enables users to perform complex system administration, file management, code analysis, and Git operations using plain natural language prompts.

The system features a **two-agent sequential pipeline** (Router & Executor), deterministic **Fast-Path execution** for common low-latency shell operations, multi-turn conversation memory, robust safety guardrails against destructive OS actions, a full-screen **Textual TUI** / **Rich CLI** interface, and zero-dependency standalone binary distribution via **PyInstaller**.

---

## 2. System Architecture & Workflow

```
               ┌───────────────────────────────────────────────┐
               │              User Prompt / Query              │
               └───────────────────────┬───────────────────────┘
                                       │
                                       ▼
                   ┌───────────────────────────────────────┐
                   │    Fast-Path Execution Check Engine   │
                   │    (Deterministic Local Bypassing)    │
                   └───────┬───────────────────────┬───────┘
                           │                       │
           [Match Found]   │                       │   [No Fast-Path Match]
                           ▼                       ▼
            ┌─────────────────────┐     ┌─────────────────────────────────────┐
            │ Direct Tool/System  │     │       CrewAI Sequential Crew        │
            │ Execution Result    │     └──────────────────┬──────────────────┘
            └─────────────────────┘                        │
                                                           ▼
                                                ┌─────────────────────┐
                                                │    Router Agent     │
                                                │ (Intent & Planning) │
                                                └──────────┬──────────┘
                                                           │
                                                           ▼
                                                ┌─────────────────────┐
                                                │   Executor Agent    │
                                                │ (Tool Orchestration)│
                                                └──────────┬──────────┘
                                                           │
                                       ┌───────────────────┴───────────────────┐
                                       │                                       │
                                       ▼                                       ▼
                         ┌───────────────────────────┐           ┌───────────────────────────┐
                         │  Custom Services / Tools  │           │   Session Memory / State  │
                         │ (Shell, File, Code, Git)  │           │    (Context & History)    │
                         └─────────────┬─────────────┘           └─────────────┬─────────────┘
                                       │                                       │
                                       └───────────────────┬───────────────────┘
                                                           │
                                                           ▼
                                                ┌─────────────────────┐
                                                │ Output Formatter UI │
                                                │  (Rich / Textual)   │
                                                └─────────────────────┘
```

### Architecture Highlights:
1. **Fast-Path Engine:** Intercepts explicit deterministic commands (e.g. `clear`, `pwd`, `ls`, `git status`) to execute locally without incurring LLM API latency.
2. **Router Agent (`Command Router & Intent Classifier`):** Analyzes the prompt alongside system environment variables and conversation history. Identifies category (`Shell`, `File`, `Code`, `Git`), verifies security constraints, and builds execution instructions.
3. **Executor Agent (`Task Executor`):** Executes planned actions using custom sandboxed Python tools (`shell_tool`, `file_tool`, `code_tool`, `git_tool`) and returns exact runtime outputs.
4. **Rich / Textual UI Layer:** Renders styled terminal responses with syntax highlighting, panel cards, and colored status indicators.

---

## 3. Prerequisites & Environment Configuration

### 3.1 Prerequisites
* **Python:** `3.10` to `3.13`
* **Git:** Required for repository setup and Git tool operations
* **OS Support:** Windows 10/11 (PowerShell/CMD), Linux (Bash/Zsh), macOS (Zsh/Bash)

### 3.2 Environment Variables (`.env`)
The application reads configuration from local `backend/.env` or global user configuration `~/.cli-agent/.env`.

| Variable Name | Type | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `OLLAMA_API_KEY` | String | *Required* | API Key for Ollama Cloud or LiteLLM backend |
| `OLLAMA_MODEL_NAME` | String | `gemma4:31b-cloud` | Model name/tag for inference |
| `OLLAMA_API_BASE` | String | `https://ollama.com/v1` | Base URL endpoint for API requests |
| `MAX_ITER` | Integer | `15` | Maximum execution iterations for the Executor Agent |
| `MAX_ROUTER_ITER` | Integer | `10` | Maximum planning iterations for the Router Agent |

---

## 4. Installation & Setup Guide

### Method 1: Automated Script Installation (Recommended)

* **Windows (PowerShell):**
  ```powershell
  .\install.ps1
  ```
* **Linux / macOS (Bash):**
  ```bash
  chmod +x install.sh
  ./install.sh
  ```

### Method 2: Manual Installation from Source

1. **Clone the Repository:**
   ```bash
   git clone https://github.com/Ayushsingh-02082004/ai-terminal-assistant.git
   cd ai-terminal-assistant
   ```

2. **Create and Activate Virtual Environment:**
   ```bash
   # Windows PowerShell
   python -m venv venv
   .\venv\Scripts\Activate.ps1

   # Linux / macOS
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Install Dependencies:**
   ```bash
   pip install -r backend/requirements.txt
   ```

4. **Configure Credentials:**
   ```bash
   cp backend/.env.example backend/.env
   ```
   *Edit `backend/.env` with your API Key and Model preferences.*

---

## 5. Usage & Features

### 5.1 Launching the Agent
* **Launch Modern Textual TUI (Default):**
  ```bash
  cd backend
  python run.py
  ```
* **Launch Classic CLI Loop:**
  ```bash
  cd backend
  python run.py --classic
  ```

### 5.2 Interactive Command Examples
* **File Operations:**
  * `"list all files in the current folder"`
  * `"find files in backend smaller than 20 MB"`
  * `"create a file named hello.txt with content Hello World"`
* **Code & Syntax Analysis:**
  * `"verify the python syntax of backend/run.py"`
  * `"check code style and look for potential bugs in backend/src/cli_agent/crew.py"`
* **Git Workflows:**
  * `"check the git status of the repository"`
  * `"show git log summary of the last 3 commits"`
* **System Commands:**
  * `"check system environment and list installed dependencies"`

---

## 6. Internal Service Toolsets Specification

Located under `backend/src/cli_agent/services/`:

| Module | Core Functionality | Safety & Constraints |
| :--- | :--- | :--- |
| `fast_path.py` | Low-latency local command bypassing for clear/ls/pwd queries | Executes direct deterministic actions without calling LLM |
| `shell_tool.py` | Cross-platform command execution via subprocess | Blocks hard drive formatting, system shutdowns, root deletions |
| `file_tool.py` | Reading, creating, searching, and deleting local workspace files | Restricts file access to files < 20MB; ignores dependency dirs |
| `code_tool.py` | Python code validation, AST parsing, and refactoring aid | Analyzes code structure safely without executing untrusted code |
| `git_tool.py` | Automated Git repository status, diff inspection, and logging | Prevents unconfirmed `--force` actions or credential leaks |
| `env_detector.py` | Detects host OS, platform architecture, shell type, and path | Provides platform awareness to LLM context |
| `memory_manager.py` | Multi-turn chat session memory manager | Retains conversational context across query steps |
| `history_manager.py` | Persists command history logs | Saves user execution logs for session recovery |

---

## 7. Declarative Agent & Task Configuration

Defined in `backend/src/cli_agent/config/`:

* **`agents.yaml`**: Configures backstories, roles, and safety rules for `router_agent` and `executor_agent`. Enforces cross-platform rejection of dangerous OS actions (`rm -rf /`, `format c:`, reading SSH keys or `/etc/shadow`).
* **`tasks.yaml`**: Defines parameters for `routing_task` and `execution_task`, specifying input variables (`system_env_context`, `conversation_history`, `user_request`) and expected structured output.

---

## 8. Standalone Binary Build & Packaging

The application includes automated cross-platform bundling via **PyInstaller** (`build_installer.py` / `cli-agent.spec`).

### Key Packaging Optimizations:
* **Payload Size Reduction:** Excludes non-essential heavy machine learning frameworks (`torch`, `tensorflow`, `scipy`, `pandas`, `lxml`, `PIL`) to keep binary distribution lightweight.
* **Included Bundles:** Auto-collects runtime dependencies for `crewai`, `crewai_tools`, `rich`, `textual`, `litellm`, and `cryptography`.
* **Building Executable:**
  ```bash
  python build_installer.py
  ```
  Generates standalone executable in `dist/cli-agent` (or `dist/cli-agent.exe`).

---

## 9. Web Portal Landing Page (`FrontEnd/`)

The repository includes a modern, Vercel-ready product website located in the `FrontEnd/` directory:
* **Tech Stack:** HTML5, Vanilla CSS3 (Custom design system), JavaScript, Vercel Configuration (`vercel.json`).
* **Features:** Responsive layout, live terminal UI mockups, interactive feature highlights, installation guides, and dark-themed aesthetics.

---

## 10. Security Guardrails & Safety Constraints

1. **Destructive Command Guard:** Rejects instructions that attempt system disk formatting (`format c:`), hard drive wipes (`mkfs`, `dd`), system shutdowns, or unrestricted root deletions (`rm -rf /`).
2. **Credential Protection:** Blocks reading or exposing private SSH keys (`~/.ssh/id_rsa`), AWS/cloud credentials (`~/.aws/credentials`), or operating system password files (`/etc/shadow`, `SAM`).
3. **Workspace Isolation:** Rejects operations attempting to mutate system configuration directories outside the project root unless explicitly permitted.
4. **Dependency Skipping:** Ignores dependency folders (`.venv`, `node_modules`, `__pycache__`, `.git`) during file scans to preserve CPU & memory efficiency.

---

## 11. Troubleshooting & FAQ

#### Q1: `OLLAMA_API_KEY was not found` prompt appears on startup.
* **Solution:** Provide your key during the interactive first-time setup prompt, or create `~/.cli-agent/.env` with your API credentials.

#### Q2: Script execution error on Windows PowerShell.
* **Solution:** Run PowerShell as Administrator and execute:
  ```powershell
  Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
  ```

#### Q3: How to clear session memory during a session?
* **Solution:** Type `clear`, `/clear`, or `reset` in the interactive console loop to reset conversation state.

---

*Documentation maintained by Ayush Singh. For issues or feature requests, visit the [GitHub Repository](https://github.com/Ayushsingh-02082004/ai-terminal-assistant).*
