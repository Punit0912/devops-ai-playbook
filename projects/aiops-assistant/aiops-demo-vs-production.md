# AIOps: Demo (Kira) vs. Production Plan (BMC Helix)

> **Purpose:** A presentation-ready reference for the PO/architect session.
> It maps the working **Kira** prototype in this repo to the **AIOps & Agentic Intelligence Layer for BMC Helix** initiative, and shows exactly where the production plan extends beyond the demo.

---

## 1. The one-liner for the demo

> "What you're about to see (**Kira**) is a **working prototype of Epic 1 + the read-only half of Epic 2**. The production plan takes this same proven pattern and scales it into a governed, multi-agent, write-capable platform for BMC Helix."

That single sentence de-risks the whole proposal — we're not pitching theory, we're pitching a **scaled version of something already running**.

---

## 2. Side-by-side: Demo repo vs. Production plan

| Capability | This repo (Kira demo) | Production plan (`bmc-helix-eks-cluster`) | Plan ref |
|-----------|----------------------|-------------------------------------------|----------|
| **Foundation model** | Bedrock (Claude), model access enabled | Same, + VPC endpoints, scoped IAM, budget alarm | Story 1.1 |
| **Reasoning engine** | Single Bedrock Agent ("Kira") | Multi-agent (orchestrator + specialists) | Epic 3+ |
| **Tools / connectors** | 3 Lambdas: logs, metrics, health | Reusable connectors: Prometheus, **Elasticsearch (helix-\*)**, CloudWatch | Story 1.3 |
| **Knowledge base (RAG)** | ❌ none | S3 Vectors over Terraform + MRs + incidents | Story 1.2 |
| **Data sources** | CloudWatch + Prometheus + EKS | + **EFK/Elasticsearch**, RDS, WAF, EventBridge, GuardDuty | §1a |
| **UI / entry point** | Streamlit web UI (human-invoked) | **Teams ChatOps** + scheduled digest + alert webhooks | Story 1.4, 2.4 |
| **Proactive reporting** | ❌ none | Daily Teams health digest | Story 1.4 |
| **Alert enrichment** | ❌ none | Alertmanager → Lambda → Bedrock → Teams | Story 2.1 |
| **RCA** | Ad-hoc, on-demand via prompt | Automated log/metric RCA + correlation | Story 2.2, 2.3 |
| **Write actions** | ❌ read-only | Terraform gen → **GitLab MR** (human+CI gated) | Epic 3 |
| **Governance** | None | Audit logging, tagging, FinOps, RBAC from day 1 | Story 1.6 |
| **Multi-agent** | ❌ single agent | ✅ orchestrator + specialist agents | Epic 3+ |
| **App-layer (Helix SQL/DevStudio)** | ❌ | Conditional draft-MR SQL (DBA-gated) | Epic 4 |
| **Value/ROI** | ❌ | Cost-vs-value scorecard, postmortems | Epic 5 |

---

## 3. What the demo *proves* (credibility anchors)

Each thing built in Kira maps to a production story — this is what makes architects trust the plan.

| What was demoed in Kira | Proves this production story is feasible |
|--------------------------|------------------------------------------|
| Bedrock agent invoking Claude in-account | **1.1** Bedrock enablement |
| Lambda action groups returning live EKS/Prometheus/CloudWatch data | **1.3** Live query connectors |
| Agent reasoning over tool output → root cause + remediation text | **2.2** Log RCA assistant |
| OpenAPI-schema'd action groups + IAM roles (agent + lambda) | **3.1** GitLab action-group tools (same mechanism, write instead of read) |
| Fluent Bit → CloudWatch pipeline | **§1a** infra signal wiring |
| End-to-end agentic loop (decide → call tool → synthesize) | The core **Agentic AI** thesis of the whole initiative |

> **Key talking point:** "The action-group + IAM + Lambda pattern used for read-only tools is *identical* to what Epic 3 uses for GitLab MR tools — the only difference is the tool does a write instead of a read, and a human approves it. So Epic 3 isn't new tech, it's the same pattern with a guardrail."

---

## 4. The critical *deltas* to call out honestly

Architects respect you more if you name the gaps yourself.

1. **Single-agent → Multi-agent.** Kira is one agent with 3 tools. Production needs specialist agents (observability, K8s, ITSM, remediation) + an orchestrator. *Justified because each agent gets tighter IAM scope and a smaller blast radius.*

2. **No RAG yet.** Kira answers from live tool calls only. Production adds a **knowledge base** (Story 1.2) so answers match your Terraform conventions — essential before any code generation (Epic 3).

3. **Elasticsearch is the big new connector.** The demo used CloudWatch for logs. **Helix app logs are deliberately *excluded* from CloudWatch** and live in Elasticsearch (`helix-*`, `logstash-*`). This is the single most important architectural correction between demo and prod — worth a dedicated slide.

4. **Streamlit → Teams.** Demo UI won't ship; production entry point is Teams ChatOps + webhooks + schedules (reusing the `lambda_ecr_vulnerability_digest.tf` scaffold the team already trusts).

5. **Read-only → Human-gated writes.** Kira never writes. Production's "read before write" principle means writes only appear in Epic 3, always as MRs, never auto-applied.

---

## 5. Suggested demo narrative (5 slides)

1. **"Here's a working AIOps agent"** — live Kira demo: ask "why are we seeing 503s?" → watch it call tools → RCA answer.
2. **"This proves the core pattern"** — the §3 table (demo → production-story mapping).
3. **"Here's the production reality"** — the signal/connector map (Elasticsearch vs CloudWatch for Helix logs).
4. **"Here's the safe rollout"** — the epic timeline; emphasize **value at Week 8**, read-before-write, human-in-the-loop.
5. **"Here's why it's low-risk"** — reuse existing scaffold, governance/FinOps from day 1, Epic 4 optional.

---

## 6. Anticipated PO/architect questions

| They'll ask | The answer |
|-------------|------------|
| "Isn't this risky on prod?" | Read-before-write; no auto-apply; every change is a human+CI-gated MR. |
| "What's the cost?" | Budget alarm in 1.1, FinOps thread throughout, ROI scorecard in 5.1. S3 Vectors chosen over OpenSearch Serverless for cost. |
| "Why not just use Copilot?" | Copilot = authoring/interactive (IDE). Bedrock = always-on, event-driven, in-VPC autonomous runtime. Complementary, not competing. |
| "Why multi-agent, not one big agent?" | Tighter per-agent IAM scope, smaller blast radius, higher accuracy per specialist. |
| "When do we see value?" | Week 8 — daily digest + onboarding bot, both read-only and low-risk. |
| "What if Helix SQL isn't in a repo?" | Epic 4 is explicitly conditional; descope to recommend-only, Epics 1–3 + 5 still deliver full value. |

---

## 7. Concepts covered by the demo

- **AIOps** — AI applied to IT operations (incident diagnosis, RCA). ✅
- **Agentic AI** — autonomous reasoning + tool-calling agent. ✅
- **LLMOps** — operating the LLM app (system prompt, agent lifecycle, tool orchestration). ✅ (managed style)
- **MLOps** — training/fine-tuning your own model. ❌ (not covered; only needed if a custom model is trained)
- **Multi-agent systems** — orchestrator + specialists. ❌ in demo, ✅ in production plan.

---

## 8. Mapping to the production initiative (quick index)

| Production Epic | Weeks | Demo coverage |
|-----------------|-------|---------------|
| **Epic 1 — Foundation** | 1–8 | ✅ Prototyped (Bedrock, connectors, agentic loop) |
| **Epic 2 — Passive Intelligence** | 9–16 | ⚙️ Partially (on-demand RCA; no alert enrichment/correlation yet) |
| **Epic 3 — Agentic Actions** | 17–24 | 🔓 Pattern proven (action groups), write path not built |
| **Epic 4 — App-Layer (conditional)** | 25–32 | ❌ Not covered (needs Elasticsearch + app corpus) |
| **Epic 5 — Production Readiness & Value** | 33–36 | ❌ Not covered (ROI, postmortems, enablement) |

---

*Draft v1 · Prepared for the PO/architect demo · Source prototype: `projects/aiops-assistant` (Kira).*
