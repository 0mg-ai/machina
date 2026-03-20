# AI Agent Platform — Final Architecture for ASUS NUC 15 Pro

## Executive Summary

A single ASUS NUC 15 Pro (32GB RAM, 1TB HDD) hosts an autonomous multi-agent platform: Slack-triggered, Kubernetes-orchestrated, gVisor-sandboxed, with durable workflow execution. Infrastructure footprint: ~5GB. Remaining ~25GB supports 10–20 concurrent agent sandboxes at 1–2GB each.

This document reconciles two independent architecture proposals (MY_ARCHITECTURE.md and RESEARCH.md), resolving their disagreements with justified choices.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ASUS NUC 15 Pro (32GB RAM, 1TB HDD)              │
│                         MicroK8s Single-Node Cluster                │
│                                                                      │
│  ┌─────────────┐                          ┌────────────────────┐    │
│  │  Slack Bolt  │─────────────────────────▶│  Temporal.io       │    │
│  │  (Socket     │                          │  (Workflow Engine)  │    │
│  │   Mode)      │◀─────────────────────────│                    │    │
│  │  ~256MB      │   progress + approvals   │  ~1.5GB w/ PG     │    │
│  └─────────────┘                          └────────┬───────────┘    │
│                                                     │                │
│                                           spawns K8s Jobs            │
│                                                     │                │
│  ┌──────────────────────────────────────────────────▼───────────┐    │
│  │                    gVisor-Sandboxed Agent Pods                │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │    │
│  │  │ Research  │  │ Coder    │  │ Analysis │  │ Writer   │    │    │
│  │  │ Agent    │  │ Agent    │  │ Agent    │  │ Agent    │    │    │
│  │  │ Claude   │  │ Claude   │  │ Claude   │  │ Claude   │    │    │
│  │  │ Agent SDK│  │ Code CLI │  │ Agent SDK│  │ Agent SDK│    │    │
│  │  │ ~512MB   │  │ ~1-2GB   │  │ ~512MB   │  │ ~512MB   │    │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │    │
│  │       └──────────────┴──────────────┴──────────────┘         │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                              │                                       │
│                     ┌────────▼─────────┐                             │
│                     │   PostgreSQL     │  ← Temporal + metadata      │
│                     │   ~512MB         │                              │
│                     └──────────────────┘                              │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  ConfigMaps: AGENT.md, SKILL.md, channel-mapping.yaml         │  │
│  │  Secrets: ANTHROPIC_API_KEY, SLACK_BOT_TOKEN                  │  │
│  │  PVCs: Agent workspaces, JSONL transcripts, workflow artifacts│  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                              │
                     Anthropic Claude API
                     (cloud inference)
```

---

## Key Disagreements Resolved

### 1. Workflow Engine: Temporal.io (not Argo Workflows)

RESEARCH recommended Argo for lower RAM (~300–500MB vs claimed 6–10GB for Temporal). However:

- **The 6–10GB Temporal figure is inflated.** Temporal's single-process dev server with PostgreSQL backend runs at ~1.5GB. The production multi-service deployment is heavy, but the dev-server mode is designed exactly for single-node setups.
- **Temporal is fundamentally better for interactive agents.** Your core requirement — "give tasks, observe progress, intercept" — maps directly to Temporal's Signal/Query primitives. With Argo, human-in-the-loop requires building a custom Suspend→external-webhook→Resume pipeline from scratch.
- **Durable execution matters.** Temporal's event-sourced history survives node reboots, pod evictions, and deploys. Argo stores workflow state in etcd CRDs, which is resilient but loses in-step progress if a pod dies mid-execution.
- **Workflow-as-code beats YAML DAGs** for agent orchestration. "If the research finds X, spawn 3 sub-agents; if Y, ask the human" is trivial in TypeScript. In Argo YAML, it requires complex template parametrization.

**Cost:** ~1GB extra RAM vs Argo. **Payoff:** Native human-in-the-loop, durable state, dynamic branching — all critical for your use case.

### 2. Agent Framework: Claude Agent SDK + Claude Code CLI (not LangChain DeepAgent)

RESEARCH recommended DeepAgent for lower per-container RAM (~512MB vs claimed 4–8GB for Claude SDK). However:

- **Claude Agent SDK is a lightweight TypeScript library**, not the full Claude Code runtime. It runs at ~256–512MB per container — comparable to DeepAgent.
- **Claude Code CLI in headless mode** is heavier (~1–2GB) but provides battle-tested coding tools (file I/O, Bash, git, Grep, Glob) with zero implementation effort. Use it specifically for coding agents.
- **You're primarily calling the Anthropic API.** Adding LangChain as an intermediary layer adds complexity, dependency surface, and abstraction leaks without meaningful benefit when Claude is your primary model.
- **The OpenClaw patterns you want to adopt** (markdown-as-config, skills, AGENT.md) map directly to Claude Code's native system. DeepAgent would require reimplementing these patterns.

**Recommendation:** Claude Agent SDK for general-purpose agents (research, analysis, writing). Claude Code CLI in headless mode for coding-specific agents. This gives you 15–25 concurrent agents depending on mix.

### 3. AI Gateway: Skip Initially, Add Portkey Later (not LiteLLM)

Both proposals agree LiteLLM has issues. RESEARCH's phased approach is correct:

- **Start without a gateway.** The Anthropic SDK handles retries, streaming, and errors natively. Rate limiting across agents can be a shared counter in PostgreSQL (already deployed for Temporal).
- **When multi-provider routing or cost analytics are needed**, deploy Portkey OSS (~128MB) rather than LiteLLM (~4GB with known memory leaks). Portkey drops in as a base URL change — no architectural modifications.

### 4. Message Queue: No Redis Needed

RESEARCH introduced Redis as a message queue between Slack and Argo. With Temporal, this is unnecessary — the Slack bot calls Temporal's workflow client directly. One fewer moving part.

---

## Decision Summary

| Decision | Choice | Why | Alternative rejected |
|---|---|---|---|
| Orchestration | **MicroK8s** (single-node) | Right-sized, single command setup | — |
| Workflow engine | **Temporal.io** (dev-server mode) | Durable execution, native human-in-loop, workflow-as-code | Argo (no native HITL, YAML-only) |
| Container isolation | **gVisor** (runsc) | ~64MB overhead/pod, no HW virtualization needed | Kata (200MB/pod), OpenShell (not K8s-native) |
| Agent lifecycle | **Agent Sandbox** (k8s SIG CRD) | K8s-native, WarmPools, NetworkPolicy integration | OpenShell (bundles own K3s) |
| Agent runtime | **Claude Agent SDK** (general) + **Claude Code CLI** (coding) | Native Anthropic integration, markdown-as-config, zero tool setup for coding | DeepAgent (unnecessary abstraction layer) |
| AI gateway | **None initially** → Portkey OSS later | Avoid premature complexity; Portkey is lightweight drop-in | LiteLLM (memory leaks, 4GB) |
| Slack | **Bolt SDK** (TypeScript, Socket Mode) | No public URL needed, direct Temporal client integration | — |
| Database | **PostgreSQL** (single instance) | Temporal backend + agent metadata | — |
| Message queue | **None** | Temporal eliminates need for Redis | Redis (unnecessary with Temporal) |

---

## RAM Budget

| Component | RAM |
|---|---|
| MicroK8s control plane + addons | ~1.0 GB |
| Temporal dev-server + PostgreSQL | ~2.0 GB |
| Slack Bolt service | ~256 MB |
| Agent Sandbox controller | ~256 MB |
| OS + headroom | ~2.0 GB |
| **Infrastructure total** | **~5.5 GB** |
| **Available for agents** | **~26.5 GB** |

Agent capacity depends on mix:
- Claude Agent SDK agents: ~512MB each → up to ~50 lightweight agents
- Claude Code CLI agents: ~1.5GB each → up to ~17 coding agents
- Realistic mixed workload: **15–25 concurrent agents**

Anthropic API rate limits will likely be the binding constraint before RAM.

---

## Component Details

### 1. MicroK8s Setup

```bash
microk8s enable dns storage rbac metrics-server
```

### 2. gVisor + Agent Sandbox

Install `runsc` binary, configure containerd, create `gvisor` RuntimeClass. Agent Sandbox provides the CRD layer on top:

```yaml
apiVersion: agentsandbox.sigs.k8s.io/v1alpha1
kind: AgentSandbox
metadata:
  name: task-abc123
spec:
  runtimeClassName: gvisor
  image: ghcr.io/0mg-ai/agent-runtime:latest
  resources:
    limits:
      memory: "2Gi"
      cpu: "2"
  networkPolicy:
    egress:
      - to: [api.anthropic.com]
      - to: [github.com]
    ingress: []
  volumes:
    - name: workspace
      persistentVolumeClaim:
        claimName: task-abc123-workspace
    - name: agent-config
      configMap:
        name: researcher-agent-config
```

Complement with Kubernetes-native security: NetworkPolicies (deny-all default, whitelist per agent), PodSecurityStandards set to `Restricted`, SecurityContexts (`runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, drop all capabilities).

### 3. Temporal.io (Dev-Server Mode)

Deploy via Helm with PostgreSQL backend (not Cassandra — too heavy for NUC):

```typescript
// Example: multi-agent workflow with human-in-the-loop
import { proxyActivities, condition, defineSignal, setHandler } from '@temporalio/workflow';

const { spawnAgent, reportToSlack } = proxyActivities<Activities>({
  startToCloseTimeout: '2 hours',
  heartbeatTimeout: '30 seconds',
});

export const humanApproval = defineSignal<[boolean]>('humanApproval');

export async function researchAndReport(task: string, slackChannel: string) {
  // Step 1: Research agent
  const research = await spawnAgent({
    type: 'claude-agent-sdk',
    config: 'agents/researcher/AGENT.md',
    task: `Research: ${task}`,
  });

  // Report to Slack, wait for human approval
  await reportToSlack(slackChannel, `Research complete. Review:\n${research.summary}`);
  let approved = false;
  setHandler(humanApproval, (value) => { approved = value; });
  await condition(() => approved, '24 hours');

  // Step 2: Writer agent (only after approval)
  const report = await spawnAgent({
    type: 'claude-agent-sdk',
    config: 'agents/writer/AGENT.md',
    task: `Write report based on: ${research.output}`,
  });

  await reportToSlack(slackChannel, `Final report:\n${report.output}`);
}
```

### 4. Agent Runtimes

**Claude Agent SDK** for general-purpose agents:

```typescript
import { Agent } from 'claude-agent-sdk';

const agent = new Agent({
  model: 'claude-sonnet-4-6',
  tools: [fileTool, webSearchTool, bashTool],
  systemPrompt: await readFile('/config/AGENT.md', 'utf-8'),
});

const result = await agent.run(taskDescription, {
  onProgress: (event) => {
    Context.current().heartbeat(event); // Temporal heartbeat
  },
});
```

**Claude Code CLI** for coding agents (headless mode):

```bash
claude --message "$TASK" \
  --system-prompt /config/AGENT.md \
  --allowedTools "Edit,Read,Write,Bash,Grep,Glob" \
  --output-format stream-json
```

### 5. Slack Integration (Bolt SDK, TypeScript)

```typescript
import { App } from '@slack/bolt';
import { Client as TemporalClient } from '@temporalio/client';

const app = new App({ token: SLACK_BOT_TOKEN, signingSecret: SLACK_SIGNING_SECRET });
const temporal = new TemporalClient();

// Route @mentions directly to Temporal — no Redis queue needed
app.event('app_mention', async ({ event, say }) => {
  const agentType = parseAgentType(event.text);
  const task = parseTask(event.text);

  const handle = await temporal.workflow.start('singleAgentTask', {
    args: [{ task, agentType, slackChannel: event.channel, threadTs: event.ts }],
    taskQueue: 'agent-tasks',
    workflowId: `task-${event.ts}`,
  });

  await say({ text: `Started ${agentType} agent. Workflow: ${handle.workflowId}`, thread_ts: event.ts });
});

// Human-in-the-loop: Slack button → Temporal signal
app.action('approve_task', async ({ body, ack }) => {
  await ack();
  const workflowId = body.actions[0].value;
  await temporal.workflow.getHandle(workflowId).signal(humanApproval, true);
});
```

Socket Mode eliminates the need for public URLs, SSL certs, or ingress — perfect for a NUC behind NAT.

---

## Agent Configuration Structure

Adopted from OpenClaw's markdown-as-config pattern:

```
agents/
├── researcher/
│   ├── AGENT.md              # System prompt, identity, behavior rules
│   ├── tools.md              # Available tools and their constraints
│   └── skills/
│       └── web-search/
│           └── SKILL.md      # Composable capability definition
├── coder/
│   ├── AGENT.md
│   ├── CLAUDE.md             # Claude Code-specific config
│   └── skills/
│       ├── test-runner/SKILL.md
│       └── pr-creator/SKILL.md
├── writer/
│   └── AGENT.md
└── workflows/
    ├── research-and-report.ts
    ├── code-review-pipeline.ts
    └── daily-standup.ts
```

Each agent directory becomes a ConfigMap mounted into the sandbox pod. All files are plain-text, version-controlled in Git, diffable, and reviewable.

---

## Patterns Adopted from OpenClaw

1. **Gateway-as-control-plane** — Slack bot normalizes input, applies routing, enforces policy before spawning agents
2. **Markdown-as-config identity** — AGENT.md + SKILL.md separate agent behavior from infrastructure
3. **Thread resolution with caching** — efficient Slack thread management for long-running tasks
4. **Channel allowlist/policy** — per-channel enforcement of which agent types respond
5. **Block-based interactive replies** — Slack modals for approval buttons tied to Temporal signals
6. **JSONL transcripts** — agent execution logs stored in PVCs alongside Temporal workflow state
7. **Announce queues** — deferred child-agent completion announcements with throttled Slack updates (~1 update/sec)
8. **Hierarchical session keys** — `workflow:slack:channel-id:thread-ts` for full routing path encoding
9. **Plugin boundaries** — each agent type is self-contained with own deps and a well-defined interface
10. **Fail-safe defaults** — narrow scope by default, explicit opt-in for broader access

---

## Implementation Order

1. **MicroK8s** setup + gVisor RuntimeClass
2. **PostgreSQL** deployment (single instance)
3. **Temporal** dev-server (Helm, PostgreSQL backend)
4. **Agent container image** (Claude Agent SDK + Claude Code CLI pre-installed)
5. **Agent Sandbox** CRD controller
6. **Slack Bolt** bot with Temporal client integration
7. **First workflow:** single-agent-task (one agent, one Slack thread)
8. **Human-in-the-loop:** approval buttons, Temporal signals
9. **Multi-agent workflows** as needed
10. **Portkey OSS gateway** when multi-provider routing is needed

---

## Hardware Note

**Swap the 1TB HDD for an NVMe SSD if possible.** The HDD is the real bottleneck — PostgreSQL, etcd, and agent file I/O all suffer on spinning disk. If the NUC has an NVMe slot, even a 256GB SSD for database volumes and etcd would meaningfully improve performance. Use the HDD for agent workspace volumes and long-term artifact storage where sequential I/O is acceptable.
