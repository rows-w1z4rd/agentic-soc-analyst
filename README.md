# SOC Multi-Agent AI System

An automated security operations pipeline that processes raw logs through four specialized agents to detect, correlate, and remediate threats end-to-end. Built with LangGraph, Qwen 2.5 7B (local), ChromaDB RAG, and Docker.

---

## What it does

Raw logs enter a LangGraph DAG where four agents work sequentially:

1. **Extractor** — Parses raw logs, filters noise, classifies events into structured JSON via SLM
2. **Analyzer** — Correlates events into incidents, maps to MITRE ATT&CK via RAG, scores confidence  
3. **Reporter** — Retrieves remediation playbooks via RAG, generates narrative reports with timestamps
4. **Executor** — Dispatches remediation actions (block IP, isolate host, notify) with audit trail

Each agent reads from shared LangGraph state—no direct calls between them.

---

## Tech stack

- **Orchestration:** LangGraph 0.2 DAG (not chains)
- **Model:** Qwen 2.5 7B via Ollama — local inference
- **Vector DB:** ChromaDB 0.5.5 with MITRE ATT&CK techniques + remediation playbooks + analyst feedback
- **Infrastructure:** Docker Compose (fully containerized)
- **API/Dashboard:** FastAPI + vanilla JS for operations feed
- **GPU:** NVIDIA RTX 4060 (×5–6 speedup vs CPU)

---

## Key concepts

**Agent orchestration via shared state**  
Four agents communicate through a TypedDict-based `PipelineState`, not by calling each other. The state grows—each agent enriches it and passes it forward. Makes debugging, testing, and reordering agents straightforward.

**RAG for threat intelligence mapping**  
MITRE ATT&CK techniques and remediation playbooks stored as vector embeddings in ChromaDB. The analyzer queries by incident description and retrieves matching techniques. Playbooks retrieved the same way. The retrieval directly shapes what the SLM sees, avoiding hard-coded detection rules.

**Local inference with optimization**  
Qwen 2.5 7B runs locally via Ollama on an RTX 4060. Handles GPU memory constraints by limiting batch size. Prompt engineered for structured JSON output from a smaller model. Temperature tuned per agent (0.1 for extraction precision, 0.3 for report narrative).

**Adaptive learning**  
A third ChromaDB collection (`analyst_feedback`) stores analyst verdicts on past incidents. During analysis, the system queries this collection alongside MITRE techniques. Feedback grows from dashboard submissions and gets re-used on subsequent runs—incidents similar to past false positives are handled differently.

**Structured log parsing at scale**  
The extractor pre-filters ~20 keywords before calling the SLM (no cost on benign lines). Handles messy, multi-format logs (nginx, SSH, syslog) by normalizing to JSON structure. Falls back to regex extraction for timestamps and IPs when needed.

**Deterministic remediation dispatch**  
Agent 4 uses keyword matching (block, isolate, notify, rate-limit, forensic) to map recommendations to handlers. No SLM guess—this keeps remediation auditable and fast. Critical incidents trigger a supplementary SLM call for additional containment steps.

**Live sandbox with real remediation**  
Two containers—`soc-victim` (Ubuntu 24.04 with nginx, SSH, rsyslog) and `soc-attacker` (generates SSH brute force, SQLi, port scans)—run real attacks. The executor can run live mode (`EXECUTOR_LIVE_MODE=true`) via Docker SDK, executing `iptables DROP` rules inside the victim container and confirming them with `iptables -L`. Validated end-to-end: attack → pipeline → real block rule → re-run attack blocked.

**ELK Stack integration**  
Elasticsearch + Kibana for log indexing and visualization. Logs bulk-ingested or streamed real-time from victim container. The extractor has a third resolution path: `ELK_MODE=true` queries Elasticsearch for logs instead of reading files.

---

## Quick run

```bash
docker compose up -d
docker exec soc-ollama ollama pull qwen2.5:7b
docker exec soc-app python src/main.py --run
# Dashboard at http://localhost:8080
```

---

## Operating modes

**Baseline** — static log dataset  
```bash
docker exec soc-app python src/main.py --run
```

**ELK Stack** — logs indexed in Elasticsearch + Kibana  
```bash
docker compose --profile elk up -d
docker exec soc-app python src/rag/elk_ingest.py
docker exec -e ELK_MODE=true soc-app python src/main.py --run
# Kibana at http://localhost:5601
```

**Live sandbox** — real attacker/victim containers with live iptables blocking  
```bash
docker compose --profile sandbox up -d
docker exec soc-attacker python3 -c "import generate_traffic as g; g.ssh_brute_force()"
docker exec -e LIVE_MODE=true -e EXECUTOR_LIVE_MODE=true -e VICTIM_CONTAINER=soc-victim \
  soc-app python src/main.py --run
docker exec soc-victim iptables -L INPUT -n  # confirm DROP rule applied
```

---

## Project structure

```
src/
├── agents/
│   ├── extractor.py       # Agent 1
│   ├── analyzer.py        # Agent 2
│   ├── reporter.py        # Agent 3
│   └── executor.py        # Agent 4
├── rag/
│   ├── mitre_loader.py    # MITRE ATT&CK → ChromaDB
│   ├── playbook_loader.py # Remediation playbooks → ChromaDB
│   ├── feedback_loader.py # Analyst feedback → ChromaDB (adaptive learning)
│   ├── elk_ingest.py      # Bulk insert logs into Elasticsearch
│   └── elk_live_shipper.py# Tail victim logs → Elasticsearch real-time
├── sandbox/
│   ├── victim/            # Ubuntu 24.04 target (nginx, SSH, rsyslog)
│   └── attacker/          # Attack traffic generator (brute force, SQLi, port scan)
├── dashboard/
│   ├── main.py            # FastAPI — /api/summary, /api/audit, /api/report, /api/feedback
│   └── index.html         # Operations + Incident Report tabs
├── shared/
│   └── memory.py          # Thread-safe shared state bus
├── data/
│   ├── sample_logs.log    # 36-line SOC simulation dataset (7 attack types)
│   └── incoming/          # Drop a log file here to use it instead of the sample
├── output/                # Timestamped run artifacts (gitignored)
└── main.py                # LangGraph StateGraph entry point
```

---

## Results

| Metric | Value |
|--------|-------|
| Line-level classification accuracy | 96.6% (28/29) |
| MITRE ATT&CK mapping accuracy | 100% (6/6 confirmed) |
| False positive rate | 0% |
| Full pipeline — GPU (RTX 4060) | ~100–130 s |
| Full pipeline — CPU | ~10–12 min |
| Total SLM calls per run | ~47 |

---

## Branches

- **`main`** — submission-stable, all results in this README are from this branch
- **`pawn-test`** — adaptive learning (validated), ELK Stack (validated), live sandbox with real iptables remediation (validated)

---

*rowd4ddy 2025/2026*
