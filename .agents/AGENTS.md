# Project-Scoped Rules & Memory for Hardcore Systems Engineering

This document serves as the permanent context and behavioral guardrails for AI agents working on the Hardcore Systems Engineering project in this workspace.

## 📂 Project Structure & Index

The workspace is organized into 11 classifications containing 57 projects. All compiled assets (PDFs, HTML maps, HTML sandboxes, Markdown cheatsheets) exist in:
* **Workspace Source**: `D:/材料/拆书/硬核技术系统工程/`
* **Publication Flat Root**: `D:/材料/拆书产出/硬核技术/`

### Published Projects (20 Active Sandboxes)
The following 20 projects are fully published with interactive offline HTML sandboxes and maps in the publication folder:
* `bpftrace`, `cilium`, `dae`, `daily_diagnostics`, `dapr`, `devops_firefighting`, `duckdb`, `maelstrom`, `production_survival`, `rocket_chip`, `ros2`, `rust_patterns`, `rustc`, `rustls`, `seastar`, `sled`, `tech_selection`, `tikv`, `tokio`, `wasmtime`.

---

## 🎨 Morandi Visual Design System

All LaTeX PDFs, HTML Maps, and Sandboxes must adhere to the low-saturation Morandi color tokens to maintain visual hierarchy:
* **M1 (Raft/Consensus - Slate Blue)**: `#7A8B99` / `RGB(122,139,153)`
* **M2 (Storage/Engine - Moss Green)**: `#7D8F7B` / `RGB(125,143,123)`
* **M3 (MVCC/Index - Plum Rose)**: `#9E828A` / `RGB(158,130,138)`
* **M4 (Lease/Locks - Terracotta)**: `#B58A7D` / `RGB(181,138,125)`
* **M5 (Watch/Async - Indigo)**: `#5F7582` / `RGB(95,117,130)`
* **M6 (Ops/Client - Antique Gold)**: `#BFA88F` / `RGB(191,168,143)`

---

## 💡 Lessons Learned & Troubleshooting Dictionaries

### 1. Python Multi-line String IndentationError
* **Problem**: When generating cheatsheet HTML generators or compilers, triple-single quotes `'''` nested within an outer triple-single quote python string will break parsing, leading to `IndentationError` or syntax errors.
* **Fix**: Use double-quotes `"""` for outer multiline blocks and single-quotes `'''` or standard quotes inside, or explicitly escape them. Never nest identical triple-quotes.

### 2. XeLaTeX Compilation Encoding
* **Problem**: Running `xelatex` via subprocess commands on Windows will trigger encoding exceptions if stdout/stderr contains non-UTF-8 characters.
* **Fix**: Always configure python command executions to use `sys.stdout.reconfigure(encoding='utf-8')` and capture subprocess outputs using `errors='replace'` or `encoding='utf-8'`.

---

## 🚀 Future Paradigms: Dual-State Sandboxes & FDE Integration

### 1. Dual-State Sandboxes (双态沙箱)
* **Explicit State (显式态)**: Muted-colored interactive SVG/HTML files designed for human readers. Keep them standalone, offline, and visually stunning.
* **Implicit State (隐式态)**: Headless, high-speed Python/BASH sandboxes running inside Docker/Firecracker without frontends, designed for training AI agents via Reinforcement Learning (RLAIF).

### 2. Forward Deployed Engineering (FDE)
When designing integration skills or agents, prioritize:
* **Agentic Guardrails**: Using `NeMo Guardrails` or prompt-validation checks.
* **Production Evals**: Using automated evaluation pipelines (`LangSmith` / `Phoenix`) to score output quality.
* **Container Orchestration**: Using `Docker` / `K8s` / `Terraform` for secure customer deployments.

---

## 📝 Technical Sandbox & Optimization Run Logs (技术沙箱与优化日志)

For historical context, debugging, or learning from prompt-backpropagation and tool self-healing verification runs, refer to:
* **Prompt Backpropagation Log**: [agent_sandbox_run_log.md](file:///D:/材料/拆书/硬核技术系统工程/03-discussions-and-iterations/agent_sandbox_run_log.md)
* **Skill Description Optimization Log**: [skill_opt_run_log.md](file:///D:/材料/拆书/硬核技术系统工程/03-discussions-and-iterations/skill_opt_run_log.md)
* **Execution Scratch Script**: [optimize_skill_description.py](file:///C:/Users/35893/.gemini/antigravity/brain/a056788c-0cd7-4802-9cb7-efde2abaf161/scratch/optimize_skill_description.py)

