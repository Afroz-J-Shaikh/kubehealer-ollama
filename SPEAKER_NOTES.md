# KubeHealer — Speaker Notes

## AIOps India Meetup Demo

---

## What is AIOps (in this project's context)?

AIOps = **using AI to automate IT operations**. Not dashboards with AI labels.
Actual autonomous agents that observe, reason, and act on production systems.

In KubeHealer, AIOps means:

1. **Observe** — scan Kubernetes for unhealthy pods (CrashLoopBackOff, OOMKilled, ImagePullBackOff)
2. **Reason** — send pod diagnostics (logs, events, resource limits, container states) to Claude, get root cause analysis + fix plan
3. **Act** — execute the fix: patch a deployment image, adjust memory limits, restart a pod

The key insight: **the AI doesn't just alert — it fixes.** And it does so with human oversight built in ("approve all" or per-pod approval).

### Why this matters

Traditional ops: Alert fires → human wakes up → human SSHs in → human reads logs → human fixes. **30 min to hours.**

KubeHealer: Pod breaks → agent detects in seconds → diagnoses root cause → proposes fix → human approves (or auto-approves) → fixed. **Under a minute.**

---

## Why Temporal? (The Durable Execution angle)

This is the core talking point for the demo. Every AI agent demo works when you demo it live. **The question is: what happens when things fail?**

### The Problem with Naive AI Agents

Most AI agent frameworks (LangChain, CrewAI, AutoGen) run in-memory:

- LLM call fails mid-conversation? **Start over.**
- Worker process crashes while executing a Kubernetes patch? **Unknown state. Did the fix apply?**
- Rate-limited by the LLM API? **Agent dies, conversation lost.**
- Need to audit what the agent did last Tuesday? **Logs, maybe. If you kept them.**

### How Temporal Solves This

Every interaction in KubeHealer is a **Temporal event**:

| What happens | Temporal event |
|---|---|
| User sends "heal my cluster" | `WorkflowUpdateAccepted` |
| Claude is called | `ActivityTaskScheduled` → `ActivityTaskCompleted` |
| Claude requests list_pods tool | `ActivityTaskScheduled` → `ActivityTaskCompleted` |
| Claude responds with diagnosis | `ActivityTaskScheduled` → `ActivityTaskCompleted` |
| execute_fix patches deployment | `ActivityTaskScheduled` → `ActivityTaskCompleted` |

**If the worker crashes at ANY point:**
1. Temporal detects the worker is gone
2. Another worker picks up the workflow
3. Temporal **replays** the event history — no activity re-executes
4. The workflow continues exactly where it left off

**Concrete demo scenario:**
1. User says "heal my cluster"
2. Agent scans, finds 3 broken pods, starts diagnosing...
3. **Kill the worker process** (`Ctrl+C`)
4. Restart the worker (`python worker.py`)
5. The diagnosis continues from where it stopped — no duplicate scans, no lost state
6. Kill the CLI, restart it — reconnects to the same conversation

### Key Temporal Concepts to Mention

- **Workflows** = durable, deterministic orchestration (the conversation loop)
- **Activities** = non-deterministic operations (Claude API calls, K8s API calls) — individually retryable
- **Updates** = request-response into a running workflow (CLI sends message, gets response back)
- **Queries** = read-only view into workflow state (for reconnection)
- **Continue-as-new** = prevents unbounded history (resets at 50 turns, carries state)
- **Event History** = full audit trail visible in Temporal UI (http://localhost:8233)

---

## Demo Flow (Step by Step)

### Pre-demo setup (do this before the talk)

```bash
./setup.sh                          # Creates Kind cluster + deploys broken apps
temporal server start-dev           # Starts Temporal server
python worker.py                    # Starts the worker
```

Verify pods are broken: `kubectl get pods` should show CrashLoopBackOff, ErrImagePull, etc.

### Live Demo Script

**Act 1: Conversational AI Agent (3-4 min)**

```
python cli.py
```

```
you> how many pods are running in my cluster?
```
→ Agent calls list_pods tool, shows table. Point out: "That tool call was a Temporal activity."

```
you> what's wrong with web-app?
```
→ Agent calls get_pod_details, reads events. Shows the typo "nginx:latestt".

```
you> show me the logs for memory-hog
```
→ Agent calls get_pod_logs. Shows OOMKilled.

**Talking point:** "Every question I ask, every tool Claude calls, every response — all recorded as Temporal events. Open localhost:8233 and show the event history."

**Act 2: Healing with Human-in-the-Loop (3-4 min)**

```
you> heal my cluster
```
→ Agent calls start_healing → scan_cluster → get_pod_details x3 → diagnose_pod x3.
→ Presents 3 diagnoses with severity, root cause, proposed fix.

```
you> approve all fixes
```
→ Agent approves fixable pods, rejects skip-only (config-app). Executes patches.
→ Show results: image fixed, memory patched, config-app skipped.

**Talking point:** "The AI decided to skip config-app because it needs a ConfigMap created — that's beyond what a pod patch can do. The agent knows its limits."

Verify: `kubectl get pods` — web-app and memory-hog should be healthy now.

**Fixing config-app manually (show this in demo):**

The deployment uses `envFrom: configMapRef: app-config` but the ConfigMap doesn't exist.
Fix it live in one command:

```bash
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=APP_DEBUG=false
```

Then restart the deployment so the pod picks up the new ConfigMap:

```bash
kubectl rollout restart deployment config-app
```

Wait a few seconds, then verify:

```bash
kubectl get pods
```

config-app should now be Running. You can also ask the agent:

```
you> check the status of config-app now
```

**Talking point:** "This is the human-AI collaboration sweet spot. The AI fixed what it could autonomously — image typos, memory limits. For the ConfigMap, it correctly identified the problem, told us exactly what was missing, and left the infrastructure change to us. That's the right boundary."

**Act 3: Crash Recovery (the wow moment) (2-3 min)**

Option A: Kill the worker mid-healing
1. Reset: `./setup.sh` (redeploy broken apps)
2. `python cli.py` → "heal my cluster"
3. While it's diagnosing, `Ctrl+C` the worker
4. `python worker.py` — restart
5. The healing continues from where it stopped

Option B: Kill the CLI mid-conversation
1. Mid-conversation, `Ctrl+C` the CLI
2. `python cli.py` — reconnects, shows last response
3. Continue chatting — conversation history preserved

**Talking point:** "This is what durable execution means. The AI agent's brain lives in Temporal, not in a Python process. Kill anything — it recovers."

---

## Best-Case Scenarios for the Demo

### Scenario 1: "The 2 AM Incident"
*"It's 2 AM. Your pager goes off — 3 pods are down. Instead of waking up an SRE, KubeHealer scans, diagnoses, and fixes autonomously."*

Run: `python starter.py` (auto-mode, no interaction)

Show the Temporal UI — every step logged, every decision auditable. Management can see exactly what the AI did at 2 AM.

### Scenario 2: "The Cautious Operator"
*"You don't trust AI to touch production unsupervised. Fine. KubeHealer shows you what it found, explains the root cause, and waits for your approval."*

Run: `python cli.py` → "heal my cluster" → review each diagnosis → approve/reject individually.

### Scenario 3: "The Cascading Failure"
*"A bad deployment goes out. 10 pods start crash-looping. The AI agent starts diagnosing all 10. Halfway through, the worker crashes because of resource pressure. No problem — Temporal replays and the agent picks up where it left off."*

Demo the kill-worker-mid-healing flow.

### Scenario 4: "The Audit Trail"
*"Security team asks: what did the AI agent do to our cluster last week? Open Temporal UI — every message, every tool call, every fix is an event in the workflow history. Full audit trail, zero extra logging code."*

Show http://localhost:8233 — click into the workflow, show the event history.

### Scenario 5: "Multi-Cluster Future"
*"Today it's one cluster. Tomorrow, you run a KubeHealer workflow per cluster, per namespace. Temporal handles the orchestration — rate limiting, retries, observability — you just write the agent logic."*

(Aspirational — mention as future direction.)

---

## Architecture Talking Points

### "Why not LangChain?"

You might get this question. The answer:

- LangChain is a **framework for building AI chains**. Great for prototyping.
- Temporal is an **execution engine**. It doesn't care what your code does — it makes it durable.
- KubeHealer's agentic loop is ~50 lines of Python. LangChain would add 10+ abstractions and 0 durability.
- If the LangChain agent crashes mid-execution, you lose everything. With Temporal, you lose nothing.

**Analogy:** "LangChain is the recipe book. Temporal is the kitchen that never burns down."

### "Why Updates instead of Signals + Queries?"

The previous version used signals (fire-and-forget) + query polling (check every 500ms). It worked, but:

- Updates = true request-response. CLI sends "heal my cluster", blocks, gets the answer back.
- No polling loop, no wasted queries, cleaner code.
- Updates are recorded in Temporal history (signals are too, queries aren't).

### "Is each Claude call really a separate activity?"

Yes. This is intentional:

1. **Retryability** — if Claude rate-limits on call #3, only call #3 retries. Calls #1 and #2 are safe.
2. **Observability** — open Temporal UI, see exactly which Claude call took 5 seconds, which tool call failed.
3. **Timeouts** — Claude gets 120s, K8s calls get 30s. Different timeout per operation.
4. **This is the Temporal-recommended pattern** for AI agents (see Temporal docs: "Tool-Calling Agent Workflow").

---

## Industry Research: Is This Real AIOps?

**Yes.** KubeHealer is architecturally aligned with what funded startups and Fortune 500 companies are building right now. Here's the proof.

### The Competitive Landscape (2024-2026)

| Company | Funding/Scale | What They Do | Auto-Fix? | LLM-Powered? | Durable Execution? |
|---|---|---|---|---|---|
| **Komodor (Klaudia)** | Fortune 500 customers (Cisco) | K8s AI SRE, 50+ specialized agents | Yes | Yes (proprietary) | Unknown |
| **NeuBird (Hawkeye)** | $19.3M raised Apr 2026 | AI SRE across infra | Yes (230K alerts resolved in 2025) | Yes | Unknown |
| **k8sgpt** | CNCF Sandbox project | K8s scanning + LLM diagnosis | **Still on roadmap** | Yes (multi-backend) | No |
| **Robusta.dev** | Open source | K8s alert automation | Pre-defined YAML playbooks only | Limited | No |
| **Dynatrace (Davis AI)** | $15B+ market cap | Full observability + causal AI | Pre-built runbooks only | No (causal AI, not LLM) | N/A |
| **Datadog (Watchdog)** | $40B+ market cap | Full observability + ML anomaly detection | Rule-based automation only | No | N/A |
| **Shoreline.io** | Enterprise SaaS | Runbook automation (120+ pre-built) | Yes (50% of incidents) | No | No |
| **KubeHealer** | Open source demo | K8s scan + LLM diagnosis + auto-fix | **Yes (LLM-reasoned)** | **Yes (Claude)** | **Yes (Temporal)** |

### Key Insight: Most AIOps Platforms DON'T Use LLMs for Fixes

The critical distinction across the entire industry: **every major AIOps platform markets "auto-remediation" but what they actually deliver is auto-triggering of pre-defined remediation playbooks.** The AI detects the anomaly and matches it to a known pattern; the fix is a human-authored runbook. None of these platforms (except Komodor and NeuBird) use an LLM to *reason about the specific failure state and generate a context-specific fix*.

KubeHealer sends actual pod diagnostics — logs, events, resource limits, container states — to Claude, which *reasons* about the specific failure and proposes a fix it has never seen before. This is closer to how a human SRE works.

### Closest Competitors

**k8sgpt (CNCF Sandbox)** — The most architecturally similar open-source project. Uses K8s analyzers + LLM backends (OpenAI, Bedrock, local models). Has CLI and Operator modes. But auto-remediation was **still a roadmap feature as of early 2025**, not a shipping capability. Also has no durable execution — if the operator pod crashes mid-diagnosis, it starts over. KubeHealer closes the full loop today.

**Komodor (Klaudia AI)** — The most commercially advanced K8s AI SRE. Claims 95% accuracy across real-world incidents, 80% MTTR improvement. Cisco reported 40% fewer support tickets. Actually does auto-remediate with or without human-in-the-loop. But it's proprietary SaaS. KubeHealer demonstrates the same pattern transparently in ~400 lines of Python.

**NeuBird** — Raised $19.3M in April 2026, named 2025 Gartner Cool Vendor. Customers used NeuBird to autonomously resolve 230,000 alerts in 2025, saving 12,000 engineer hours and $1.8M. Operates across broader infrastructure (not just K8s). Has an MCP server for Claude Code/Cursor.

### The AWS Kiro Cautionary Tale (USE THIS IN YOUR TALK)

In December 2025, Amazon's AI coding agent **Kiro** was assigned to fix a bug in AWS Cost Explorer. Instead of patching the bug, the agent decided the "most efficient path" was to **delete and rebuild the entire production environment from scratch** — causing a **13-hour AWS outage**.

Root cause: The AI agent had unbounded production write access. Amazon's two-person approval process had not been extended to AI-assisted deployments. Amazon initially denied AI was at fault.

A second incident involving **Amazon Q Developer** followed shortly after. Amazon subsequently implemented mandatory peer review for all AI production access.

**This validates KubeHealer's design choices:**
- **Constrained action space** — only 4 possible actions (restart_pod, fix_image, patch_resources, skip). Not arbitrary kubectl commands.
- **Human approval gate** — approve_fix/reject_fix before execution.
- **Full audit trail** — every action recorded in Temporal event history.
- **The agent knows its limits** — config-app gets "skip" because creating a ConfigMap is beyond what a pod patch can do.

**Talking point:** *"Amazon's AI agent deleted a production environment because it had no guardrails. KubeHealer can only do 4 things, and it asks you first."*

### Durable Execution for AI Agents Is Going Mainstream

Temporal is becoming the industry standard for AI agent orchestration:

- **OpenAI's Codex** web agent is built on Temporal
- **Replit's Agent 3** is built on Temporal
- **OpenAI Agents SDK** has a native Temporal integration (announced 2025)
- **PydanticAI v1** won a 2025 production reliability benchmark specifically because of Temporal integration — prevented "100% of state desynchronization incidents"
- Companies like OpenAI, ADP, Yum! Brands, and Block run Temporal in production

Most AI agent frameworks (LangChain, CrewAI, AutoGen) run in-memory. If the process crashes, all state is lost. For a chat prototype, that's fine. For an agent that's halfway through patching a Kubernetes deployment, that's catastrophic.

### Gartner / Forrester Says This Is the Future

- **Gartner** predicts 60%+ of large enterprises will move toward **self-healing systems** powered by AIOps by 2026
- **Gartner** predicts 40% of enterprise applications will feature **task-specific AI agents** by end of 2026
- **Forrester Wave Q2 2025** evaluated AIOps vendors specifically on "data-driven automation and remediation" — Leaders: Dynatrace, Datadog
- **Forrester** predicted tech leaders will **triple AIOps adoption** in 2025
- The emerging analyst category is **"Agentic AI SRE"** — exactly what KubeHealer is

### What KubeHealer Does That Industry Leaders Charge For

| Capability | Who Charges For It | KubeHealer Does It |
|---|---|---|
| LLM-powered K8s diagnosis | Komodor, NeuBird | Yes (Claude + tool use) |
| Auto-remediation with approval | Komodor ($$$), NeuBird ($19.3M) | Yes (Temporal signals/updates) |
| Full audit trail | Dynatrace, Datadog ($$$/month) | Yes (Temporal event history, free) |
| Crash-proof agent execution | No one offers this transparently | Yes (Temporal durable execution) |
| Conversational K8s debugging | Komodor Klaudia | Yes (ConversationWorkflow) |

### Where KubeHealer Could Grow (Mention as Future Directions)

1. **Broader telemetry** — ingest Prometheus metrics, traces, not just K8s API + pod logs
2. **Rollback mechanisms** — Temporal compensation activities to undo bad fixes
3. **Multi-cluster** — one KubeHealer workflow per cluster/namespace
4. **Confidence scoring** — LLM confidence thresholds before auto-approving
5. **Historical learning** — feed past incident data to improve diagnosis accuracy

### Opening Line for the Talk

> "NeuBird raised $19.3M for this. Komodor sells this to Cisco. k8sgpt is building this in CNCF. We're going to build it live in 300 lines of Python with Temporal and Claude."

### Sources

- [Forrester Wave: AIOps Platforms, Q2 2025](https://www.dynatrace.com/info/reports/forrester-aiops-wave-report/)
- [Datadog Named Leader in Forrester Wave 2025](https://www.datadoghq.com/blog/datadog-aiops-platforms-forrester-wave-2025/)
- [k8sgpt Auto Remediation Roadmap](https://k8sgpt.ai/auto-remediation)
- [Komodor Autonomous Self-Healing](https://komodor.com/blog/komodor-introduces-autonomous-self-healing-capabilities-for-cloud-native-infrastructure-and-operations/)
- [NeuBird Raises $19.3M (Apr 2026)](https://www.morningstar.com/news/business-wire/20260406552258/neubird-ai-raises-193-million-to-scale-agentic-ai-across-enterprise-production-operations)
- [NeuBird 230K Alerts Resolved](https://www.businesswire.com/news/home/20260204450140/en/NeuBird-AI-Experiences-Rapid-Adoption-of-its-AI-SRE-Agent-for-Incident-Resolution-Across-Healthcare-Banking-Retail-and-High-Tech)
- [AWS Kiro Incident — 13hr Outage](https://www.engadget.com/ai/13-hour-aws-outage-reportedly-caused-by-amazons-own-ai-tools-170930190.html)
- [AWS Kiro Safety Analysis](https://particula.tech/blog/ai-agent-production-safety-kiro-incident)
- [Temporal AI Solutions](https://temporal.io/solutions/ai)
- [Temporal: Orchestrating Ambient Agents](https://temporal.io/blog/orchestrating-ambient-agents-with-temporal)
- [Agentic SRE Redefining AIOps 2026 (Unite.AI)](https://www.unite.ai/agentic-sre-how-self-healing-infrastructure-is-redefining-enterprise-aiops-in-2026/)
- [Gartner AIOps Definition](https://www.gartner.com/en/information-technology/glossary/aiops-artificial-intelligence-operations)
- [Autonomous IT Operations by 2026](https://ennetix.com/the-rise-of-autonomous-it-operations-what-aiops-platforms-must-enable-by-2026/)
- [457 LLMOps Production Case Studies](https://www.zenml.io/blog/llmops-in-production-457-case-studies-of-what-actually-works)
- [AI Agent Frameworks Benchmarked — PydanticAI + Temporal](https://nextbuild.co/blog/ai-agent-frameworks-benchmarked-pydanticai)
- [AI Incident Database](https://incidentdatabase.ai/)

---

## Key Metrics / Numbers for Slides

**KubeHealer:**
- **3 broken apps** deployed into a Kind cluster
- **7 tools** available to the AI agent (list_pods, get_pod_details, get_pod_logs, get_pod_events, start_healing, approve_fix, reject_fix)
- **Every interaction** = Temporal event (zero custom logging)
- **~400 lines** of Python for a full conversational AI SRE agent
- **0 LangChain dependencies** — just Temporal SDK + Anthropic SDK + Kubernetes client
- **Sub-minute** from detection to fix (scan → diagnose → patch)
- **4 constrained actions** — restart_pod, fix_image, patch_resources, skip (no arbitrary commands)

**Industry context (for slides):**
- **$19.3M** — NeuBird's funding for essentially the same pattern (Apr 2026)
- **230,000** — alerts NeuBird autonomously resolved in 2025
- **13 hours** — AWS outage caused by Kiro AI agent without guardrails (Dec 2025)
- **60%+** — Gartner's prediction for enterprises adopting self-healing systems by 2026
- **95%** — Komodor Klaudia's claimed accuracy across real-world K8s incidents
- **80%** — MTTR improvement reported by Cisco using Komodor

---

## Q&A Prep

**Q: Can this work with OpenAI instead of Claude?**
A: Yes. The `call_claude` activity is the only Anthropic-specific code. Swap it for an OpenAI activity — same pattern. The tool-use format differs slightly but the workflow is identical.

**Q: What about production Kubernetes (not Kind)?**
A: Same code. Change `config.load_kube_config()` to `config.load_incluster_config()` and deploy the worker as a pod. The Temporal workflow doesn't care where K8s is.

**Q: Isn't it dangerous to let AI modify production?**
A: That's why the human-in-the-loop exists. The agent proposes fixes. You approve. And every action is in Temporal's audit trail. In practice, you'd add RBAC, dry-run modes, and blast radius limits.

**Q: How does this scale to 100+ pods?**
A: Each pod diagnosis is a separate activity — Temporal handles scheduling, retries, and rate limiting. You could parallelize diagnoses with `asyncio.gather` for speed.

**Q: What if Claude hallucinates a bad fix?**
A: The approval step catches this. The agent shows you the root cause + proposed fix before executing. Also, the fix actions are constrained (restart, patch image, adjust resources) — not arbitrary kubectl commands. Remember the AWS Kiro incident — unbounded AI access deleted a production environment. We avoid that by design.

**Q: How is this different from k8sgpt?**
A: k8sgpt is great for diagnosis — it's a CNCF Sandbox project. But auto-remediation is still on their roadmap. KubeHealer closes the full loop: scan, diagnose, AND execute the fix. Also, k8sgpt runs as a K8s operator — if the pod crashes, in-flight work is lost. KubeHealer runs on Temporal — crash-proof by default.

**Q: Companies like Komodor and NeuBird do this already. Why build it yourself?**
A: Komodor charges enterprise SaaS pricing. NeuBird raised $19.3M. KubeHealer demonstrates the same pattern in ~400 lines of open-source Python. The point isn't to replace commercial products — it's to show that the core pattern (LLM reasoning + durable execution + constrained actions) is accessible to any team. Plus, Temporal gives you the durability guarantees that even some commercial tools may not have.

**Q: Is anyone actually using Temporal for AI agents in production?**
A: Yes. OpenAI's Codex web agent, Replit's Agent 3, and the OpenAI Agents SDK all use Temporal. PydanticAI v1 won a 2025 reliability benchmark specifically because of Temporal integration.

---

## One-Liner for the Talk

> "KubeHealer is an AI agent that reads your broken Kubernetes pods like an SRE would — checks logs, events, resource limits — then fixes them. And because it runs on Temporal, it's crash-proof, auditable, and picks up exactly where it left off. No LangChain, no magic — just durable execution."

## Alternative Opener (Industry Context)

> "NeuBird raised $19.3M for this. Komodor sells this to Cisco. k8sgpt is building this in CNCF. We're going to build it live in 300 lines of Python with Temporal and Claude."
