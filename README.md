# open-ISCR
Capisco il problema. L'app mobile non gestisce i link di download sandbox. Ecco il contenuto completo della blueprint, così puoi copiarlo e incollarlo direttamente dove preferisci.

---

OpenISCR

Open In-Session Contextual Reclamation Protocol

> A privacy-first, provider-agnostic protocol and SDK for capturing structured developer feedback inside AI-powered IDEs — without interrupting flow.

---

1. Vision

Current feedback loops are broken:
- Users must leave the IDE, reconstruct context, and open tickets.
- Product teams receive decontextualized "I'm unhappy" reports.
- High-value developers (heavy MCP, agent, codebase users) churn silently because their friction is invisible to analytics optimized for casual chat usage.

OpenISCR is an open protocol that lets any IDE or AI coding assistant detect session friction, extract anonymized metadata (never content), and transmit actionable signals to product teams — all without the user leaving their workspace.

Core principles:
1. Privacy by Design: No conversation content, no code, no PII. Only metrics and signals.
2. Provider Agnostic: Works with OpenAI, Anthropic, Google, Kimi, local LLMs, or any API-compliant backend.
3. Zero-Friction Opt-In: One-click contextual consent, not a terms-of-service wall.
4. Self-Hostable: Anyone can run the collector and dashboard. No vendor lock-in.

---

2. Architecture (Abstract)

```
┌─────────────────────────────────────────────────────────────┐
│                     IDE / Editor                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   Session    │  │  Frustration │  │  Context         │ │
│  │   Monitor    │  │  Detector    │  │  Extractor       │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘ │
│         │                 │                    │           │
│         └─────────────────┴────────────────────┘           │
│                           │                                │
│                    ┌──────┴──────┐                         │
│                    │   Privacy   │                         │
│                    │   Layer     │                         │
│                    └──────┬──────┘                         │
│                           │                                │
│                    ┌──────┴──────┐                         │
│                    │ Transmitter │                         │
│                    │  (Batch)    │                         │
│                    └──────┬──────┘                         │
└─────────────────────────────┼───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    OpenISCR Collector                         │
│  (Self-hosted / Managed) — Receives anonymized payloads    │
│  - Aggregation                                               │
│  - Differential privacy enforcement                          │
│  - Alert routing to Product / Quality teams                  │
└─────────────────────────────────────────────────────────────┘
```

2.1 Components

Component	Responsibility	Host	
Session Monitor	Tracks token usage, latency, feature flags (MCP, tools, agent mode), session duration.	IDE (local or web)	
Frustration Detector	Triggers on linguistic patterns + statistical anomalies (e.g., token consumption > 3σ from user baseline).	IDE (local or web)	
Context Extractor	Builds a structured metadata payload. Never reads prompt/response content.	IDE (local or web)	
Privacy Layer	Applies ε-differential privacy to numerical metrics; strips identifiable device fingerprints.	IDE + Collector	
Transmitter	Buffers, batches, and sends payloads asynchronously (beacon, fetch, or background sync).	IDE (local or web)	
Collector	Receives, validates schema, aggregates, and exposes an API for dashboards.	Self-hosted server	
Dashboard	Real-time view of aggregated signals, segment health, and anomaly trends.	Self-hosted server	

---

3. The Protocol (Payload Schema v1.0)

```json
{
  "$schema": "https://open-iscr.org/schema/v1.0.json",
  "type": "object",
  "required": ["schema_version", "session_id", "timestamp", "environment", "metrics", "anomaly_score"],
  "properties": {
    "schema_version": { "type": "string", "enum": ["1.0"] },
    "session_id": { "type": "string", "format": "uuid" },
    "timestamp": { "type": "string", "format": "date-time" },
    "environment": {
      "type": "object",
      "properties": {
        "ide": { "type": "string", "examples": ["vscode", "cursor", "windsurf", "theia", "web"] },
        "ide_version": { "type": "string" },
        "llm_provider": { "type": "string", "examples": ["openai", "anthropic", "kimi", "google", "local"] },
        "model": { "type": "string", "examples": ["gpt-4o", "claude-3-7", "k2.7-code", "llama3.1"] },
        "features": {
          "type": "array",
          "items": { "type": "string", "examples": ["mcp", "agent", "skills", "inline-chat", "composer"] }
        }
      }
    },
    "metrics": {
      "type": "object",
      "properties": {
        "tokens_prompt": { "type": "integer", "minimum": 0 },
        "tokens_completion": { "type": "integer", "minimum": 0 },
        "tokens_total": { "type": "integer", "minimum": 0 },
        "mcp_calls": { "type": "integer", "minimum": 0 },
        "tools_used": { "type": "array", "items": { "type": "string" } },
        "session_duration_ms": { "type": "integer", "minimum": 0 },
        "request_count": { "type": "integer", "minimum": 0 }
      }
    },
    "anomaly_score": { "type": "number", "minimum": 0, "maximum": 1, "description": "Computed deviation from user baseline" },
    "frustration_signals": {
      "type": "array",
      "items": { "type": "string", "examples": ["cost_complaint", "inefficiency", "abandonment_intent"] }
    },
    "user_segment": {
      "type": "string",
      "examples": ["anonymous", "free", "pro", "enterprise"]
    },
    "event_context": {
      "type": "object",
      "description": "Optional: active promotions, experiments, or events",
      "properties": {
        "event_id": { "type": "string" },
        "event_name": { "type": "string" }
      }
    }
  }
}
```

Key rule: If any field contains conversation text, code, file paths, or stack traces, the Collector must reject the payload.

---

4. Privacy Model

Layer	Mechanism	
Extraction	Only metadata leaves the IDE. No prompt/response inspection.	
Anonymization	`session_id` is a rotating UUID, not tied to auth. No IP logging.	
Differential Privacy	Numerical metrics (tokens, duration) are perturbed with Laplace noise (ε = 1.0). Trends remain accurate; individual sessions are unidentifiable.	
Consent	Granular opt-in for every signal type. "Send technical summary to product team?" — not a blanket ToS acceptance.	
Retention	Collector aggregates and discards raw events within 24h. Only histograms and trend lines persist.	

---

5. Repository Structure

```
open-iscr/
├── packages/
│   ├── core/                    # TypeScript SDK: SessionMonitor, Detector, Extractor, PrivacyLayer, Transmitter
│   ├── adapter-vscode/          # VS Code / Cursor / Windsurf extension wrapper
│   ├── adapter-web/             # npm package for web IDEs (CodeSandbox, StackBlitz, Theia)
│   ├── adapter-jetbrains/       # IntelliJ / PyCharm / WebStorm plugin (Kotlin)
│   ├── collector/               # Fastify/Node.js ingestion server + schema validation
│   └── dashboard/               # Next.js self-hosted dashboard (aggregated views, alerts)
├── protocol/
│   ├── schema-v1.0.json
│   └── schema-v1.0.ts           # Zod / JSON Schema types
├── docs/
│   ├── architecture.md
│   ├── privacy.md
│   ├── integration-vscode.md
│   ├── integration-web.md
│   └── self-hosting.md
├── examples/
│   ├── vscode-mcp-integration/
│   └── web-cursorless-integration/
├── README.md
└── LICENSE (MIT)
```

---

6. Integration Examples

6.1 VS Code / Local IDE

```typescript
import { OpenISCR } from '@open-iscr/adapter-vscode';

const iscr = new OpenISCR({
  collectorEndpoint: 'https://your-collector.dev/ingest',
  apiKey: 'optional',
  features: ['mcp', 'agent', 'skills'],
  llmProvider: 'kimi',
  model: 'k2.7-code',
  privacy: {
    differentialPrivacyEpsilon: 1.0,
    includeUserSegment: true, // 'pro' | 'free' — no email, no userId
  }
});

// Start monitoring a session
iscr.startSession();

// Hook into your LLM client
llmClient.on('requestComplete', (meta) => {
  iscr.recordMetrics({
    tokens_prompt: meta.promptTokens,
    tokens_completion: meta.completionTokens,
    mcp_calls: meta.mcpCallCount,
    tools_used: meta.toolNames,
  });
});

// Frustration is detected automatically via input pattern + anomaly score.
// Or trigger manually if the user clicks "Report Issue":
iscr.triggerSignal({
  frustrationSignals: ['cost_complaint', 'abandonment_intent'],
  eventContext: { event_id: 'token-cup-2026', event_name: 'Kimi Token Cup' }
});
```

6.2 Web IDE

```typescript
import { OpenISCRWeb } from '@open-iscr/adapter-web';

const iscr = new OpenISCRWeb({
  collectorEndpoint: '/api/iscr', // Proxied through your backend to avoid CORS
  beaconMode: true, // Uses navigator.sendBeacon for unload reliability
});

// Integrate with your web-based LLM client
aiService.on('turn', (turnMeta) => {
  iscr.record({
    tokens_total: turnMeta.totalTokens,
    features: turnMeta.activeFeatures,
  });
});
```

---

7. Self-Hosting the Collector & Dashboard

```bash
git clone https://github.com/your-org/open-iscr.git
cd open-iscr/packages/collector
npm install
cp .env.example .env
# Set DATABASE_URL, REDIS_URL, optional SMTP for alerts
npm run migrate
npm run start

cd ../dashboard
npm install
npm run build
npm start
```

Collector endpoints:
- `POST /ingest` — Receives batched payloads
- `GET /health` — Service health
- `GET /aggregates?from=&to=` — Dashboard data

---

8. Roadmap

Phase	Deliverable	
MVP	Core SDK + VS Code adapter + Collector + Dashboard	
v0.2	Web SDK with beacon support; JetBrains adapter	
v0.3	Automatic compensation hook (product-defined remediation offers)	
v0.4	Federated learning option for anomaly detection without centralizing data	
v1.0	Standardized protocol submission to IETF / W3C community group	

---

9. Why This Matters

OpenISCR turns churn from a lagging indicator ("Oh, they left last month") into a leading signal ("This session pattern predicts churn in 72h").

It does so without surveillance, without breaking flow, and without giving the provider an excuse to say "We didn't know."

If a product team chooses not to act on the signal, that is a business decision, not a technical blindspot.

---

License

MIT — Open Source, provider-neutral, forever.