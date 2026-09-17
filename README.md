<div align="center">

# DiagramForge AI

**An agentic AI assistant that turns natural-language prompts into editable technical diagrams on a live Excalidraw canvas.**

*"Draw the OAuth authorization-code flow"* → a labeled, connected, overlap-free sequence diagram you can keep editing by hand or by chat.


`TypeScript` · `React 19` · `Vercel AI SDK` · `OpenAI GPT-5.4` · `Cloudflare Workers + Durable Objects` · `Excalidraw` · `Upstash Vector` · `Tavily` · `Braintrust` · `Zod`

</div>

---

## What it does

DiagramForge AI is not a one-shot "prompt → picture" generator. It is a **multi-step, tool-using agent** that treats the canvas as state it can read, modify and extend:

- **Create** architecture, sequence, flowchart, state-machine and ER diagrams from plain English.
- **Modify** existing diagrams conversationally — *"make the login box red"*, *"add a cache between the API and the database"* — without redrawing everything.
- **Research before drawing** with a private RAG knowledge base and live web search, so diagrams of real systems are accurate rather than hallucinated.
- **Self-correct layouts**: every canvas mutation returns overlap feedback, and the agent is required to fix collisions in its next step.
- **Stay editable**: output is native Excalidraw elements with real arrow bindings, so you can drag shapes and the arrows follow.

## Architecture

```
┌───────────────────────────────┐          WebSocket           ┌──────────────────────────────────┐
│  Browser (React + Excalidraw) │ ◄──────────────────────────► │  Cloudflare Worker               │
│                               │   streamed tokens + tool     │  DesignAgent (Durable Object)    │
│  useAgentChat / onToolCall    │   calls (Vercel AI SDK)      │  streamText + stepCountIs(8)     │
│                               │                              │                                  │
│  queryCanvas    ─► read scene │                              │  server-side tools:              │
│  addElements    ─► mutate     │ ── tool results ───────────► │   searchWeb      ─► Tavily       │
│  updateElements ─► mutate     │                              │   searchKnowledge─► Upstash Vector│
│  removeElements ─► mutate     │                              │                                  │
└───────────────────────────────┘                              └──────────────────────────────────┘
```

Key design decisions:

| Concern | Approach |
|---|---|
| **Canvas as source of truth** | The four canvas tools are *client-side* tools (no `execute` on the server). The Worker streams the call, the browser applies it to the live Excalidraw scene and returns the result via `addToolOutput`, and the agent loop resumes. |
| **Stateful sessions** | The agent runs as a Cloudflare **Durable Object** (`AIChatAgent`) with SQLite-backed message persistence per session. |
| **Shared agent core** | `src/agent-core.ts` holds the system prompt, tool wiring and step limit. The production Worker (`streamText`) and the eval harness (`generateText`) both call it, so they cannot drift. |
| **Layout feedback loop** | `addElements` returns an `overlaps` array from bounding-box collision detection (`src/context/overlaps.ts`). The system prompt makes fixing them mandatory. |
| **Cross-call arrow binding** | Excalidraw's `convertToExcalidrawElements` only resolves arrow `start`/`end` ids within a single batch. `src/context/cross-call-bindings.ts` patches bindings to shapes from earlier calls or the existing canvas, so multi-call diagrams don't end up with floating arrows. |
| **Compact canvas context** | `serializeCanvasState` gives the model a human-readable summary (id, type, position, label, connections) instead of raw Excalidraw JSON. |

## Agent tools

| Tool | Runs on | Purpose |
|---|---|---|
| `queryCanvas` | Browser | Read a summary of every element currently on the canvas. Called before any modification. |
| `addElements` | Browser | Add shapes, text and bound arrows using Excalidraw's skeleton format. Returns overlap detection. |
| `updateElements` | Browser | Recolor, move, relabel or resize existing elements by id. |
| `removeElements` | Browser | Delete elements by id. |
| `searchWeb` | Worker | Tavily search for current information about unfamiliar systems. |
| `searchKnowledge` | Worker | Top-K semantic retrieval over a private corpus in Upstash Vector (RAG). |

All tool inputs are validated with **Zod** schemas (`src/tools/element-schema.ts`).

## Evaluation

The agent is measured, not vibe-checked. `evals/` contains a **Braintrust** eval harness:

- **Golden dataset** (`evals/datasets/`) — 30+ cases across `create` / `modify` / `domain` / `edge` categories, tagged by difficulty. Modify cases replay a synthetic prior conversation so the agent can't tell it apart from a real session.
- **Headless canvas simulator** — `runAgent` in `agent-core.ts` seeds an in-memory canvas and answers `queryCanvas` inline; `applySkeleton.ts` mirrors what Excalidraw would actually render so scorers grade *rendered results*, not raw model claims.
- **Custom scorers** (`evals/scorers/`):
  `schema` · `structure` · `toolChoice` · `labelKeyword` · `boundArrows` · `boundLabels` · `connectivity` · `noOverlaps`
- Every run is auto-tagged with git branch, commit and dirty flag, so experiments are comparable in the Braintrust dashboard.

```bash
npm run eval
```

## RAG knowledge base

`data/corpus/` holds short reference docs the agent can consult before drawing:

- OAuth 2.0 authorization-code flow
- Kubernetes pod networking
- PostgreSQL write path
- Cloudflare Workers request lifecycle
- Startup org structure
- General diagramming best practices

Embeddings are generated server-side by Upstash's hosted model. The embed script resets the index and re-upserts every file, so re-running it is idempotent:

```bash
npm run embed
```

## Getting started

### Prerequisites

- Node.js 20+
- API keys: [OpenAI](https://platform.openai.com/), [Upstash Vector](https://upstash.com/) (create an index with a hosted embedding model), [Tavily](https://tavily.com/) (free tier), [Braintrust](https://www.braintrust.dev/) (optional, for evals)

### Setup

```bash
git clone <this-repo>
cd diagramforge-ai
npm install

cp .dev.vars.example .dev.vars
# fill in OPENAI_API_KEY, TAVILY_API_KEY, BRAINTRUST_API_KEY,
# UPSTASH_VECTOR_REST_URL, UPSTASH_VECTOR_REST_TOKEN

npm run embed   # populate the knowledge base
npm run dev     # http://localhost:5173
```

### Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Vite dev server with the Cloudflare Worker running locally |
| `npm run build` | Production build |
| `npm run agent "draw a flowchart"` | Hit the agent over WebSocket without the UI (dev server must be running) |
| `npm run eval` | Run the Braintrust eval suite |
| `npm run embed` | Rebuild the Upstash Vector index from `data/corpus/` |

### Deploy

```bash
npx wrangler deploy
```

Set the same secrets in Cloudflare with `npx wrangler secret put <NAME>`.

## Project structure

```
src/
├── agent.ts              # DesignAgent Durable Object (AIChatAgent)
├── agent-core.ts         # System prompt, streamAgent (prod) + runAgent (eval)
├── tools.ts              # Tool aggregator
├── tools/                # One file per tool + shared Zod element schema
├── context/
│   ├── canvas-state.ts   # Canvas → compact summary for the model
│   ├── overlaps.ts       # Bounding-box collision detection
│   ├── cross-call-bindings.ts
│   └── applySkeleton.ts  # Node-side simulator of Excalidraw's skeleton helper
├── rag/                  # Upstash Vector client + embed script
├── components/           # Canvas, chat panel, streaming + tool-status UI
└── worker.ts             # Worker entry
evals/
├── diagram.eval.ts       # Braintrust eval definition
├── datasets/             # Golden test cases
└── scorers/              # Custom scorers
data/corpus/              # RAG source documents
```

