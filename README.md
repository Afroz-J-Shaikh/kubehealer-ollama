# KubeHealer

AI agent that finds broken Kubernetes pods, diagnoses them with Claude,
and fixes them automatically — orchestrated by Temporal for durable execution.

## How It Works

1. **Scan** — finds unhealthy pods (CrashLoopBackOff, OOMKilled, ImagePullBackOff...)
2. **Diagnose** — sends pod details to Claude, gets root cause + fix plan
3. **Fix** — executes remediation (restart, patch image, adjust resources)

Everything runs inside Temporal workflows = crash-proof, retryable, fully observable.

## Two Modes

### Interactive (conversational AI agent)

```
python cli.py
```

Chat naturally with your cluster:

```
you> how many pods are running?
you> what's wrong with web-app?
you> show me the logs for memory-hog
you> heal my cluster
you> approve all fixes
```

Every message, every AI response, every tool call is stored in Temporal's event history.
Kill the CLI or worker mid-conversation — restart and it picks up exactly where it left off.

### Auto-heal (one-shot)

```
python starter.py
```

Scans, diagnoses, and fixes everything automatically. No interaction needed.

## Quick Start

Prerequisites: Python 3.11+, Docker, Kind, Temporal CLI, Anthropic API key

```
Terminal 1: ./setup.sh
Terminal 2: temporal server start-dev
Terminal 3: pip install -r requirements.txt && cp .env.example .env  # paste your API key in .env
Terminal 3: python worker.py
Terminal 4: python cli.py
```

Open http://localhost:8233 to see the workflow trace in Temporal UI.

## What Gets Fixed

| Broken App | Problem | AI Diagnosis | Auto-Fix |
|---|---|---|---|
| web-app | Image "nginx:latestt" (typo) | Detects typo | Patches to nginx:latest |
| memory-hog | 10Mi limit + stress 100M | OOMKilled | Patches to 256Mi |
| config-app | Missing ConfigMap | Can't auto-fix | Skips with explanation |

## Architecture

```
CLI (thin terminal)                    Temporal Worker
  |                                         |
  |-- update(send_message, "how many") --->|
  |                                    ConversationWorkflow
  |                                    ├─ activity: call_claude
  |                                    ├─ activity: list_pods       (tool call)
  |                                    ├─ activity: call_claude     (with tool result)
  |                                    └─ returns response via update
  |<-- "I see 5 pods running..." ----------|
  |                                         |
  |-- update(send_message, "heal it") ---->|
  |                                    ├─ activity: call_claude → tool: start_healing
  |                                    ├─ activity: scan_cluster
  |                                    ├─ activity: get_pod_details (x3)
  |                                    ├─ activity: diagnose_pod (x3)
  |                                    ├─ activity: call_claude → "Found 3 issues..."
  |<-- response with diagnoses ------------|
```

### Key Design Decisions

- **Temporal Updates** (not signal+query) — CLI sends a message and blocks until the full agentic loop completes. No polling. The response comes back directly.
- **Each Claude call and tool call = separate activity** — individually retryable, visible in Temporal UI, different timeouts per tool.
- **Continue-as-new at 50 turns** — prevents unbounded event history.
- **Fixed workflow ID** — CLI reconnects to the same conversation after crash.

## Tech Stack

- **Temporal** — durable workflow orchestration
- **Anthropic Claude** — LLM for diagnosis + conversational agent
- **Kubernetes** — target environment
- **Kind** — local K8s cluster
- **Python** — everything glued together

## Built For

AIOps India meetup — demonstrating AIOps, DevOps, AI Agents,
and Durable Execution in practice.
