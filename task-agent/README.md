# Seed Export Compliance Agent

A small, self-contained Camunda 8 example illustrating the **Task agent** pattern
from [camunda.com/orchestrate/agents](https://camunda.com/orchestrate/agents/):

> Acts on systems. Queries databases, calls APIs, updates records, sends
> communications. The LLM reasons about which tool to invoke and when.

The process model lives at [models/seed-export-compliance-agent.bpmn](models/seed-export-compliance-agent.bpmn),
with a start form at [models/seed-export-shipment-ready.form](models/seed-export-shipment-ready.form).

## What this demonstrates

An AI Agent (ad-hoc sub-process + AI Agent connector) is given four tools, one
per bullet in the source copy above, each hitting a **real, live, public,
keyless** service — no mocks, no stubs, no local fixtures pretending to be
external systems:

| # | Bullet it demonstrates | Protocol | Public service | BPMN element |
|---|---|---|---|---|
| 1 | Queries databases | SQL | UCSC public MySQL server (`hg38`) | `VerifyGeneticMarker` |
| 2 | Calls APIs | GraphQL | [Countries GraphQL API](https://countries.trevorblades.com/) | `CheckDestinationCountry` |
| 3 | Updates records | REST | [api.mathjs.org](https://api.mathjs.org) expression evaluator | `ComputeComplianceScore` |
| 4 | Sends communications | REST | [httpbin.io](https://httpbin.io) echo endpoint | `NotifyExportTeam` |

Tools 3 and 4 both use REST rather than four distinct protocols — the SOAP
connector is Self-Managed/Hybrid only and never available on SaaS, which
would conflict with the SaaS-first, zero-config-LLM design below. See
"Design notes" at the bottom for more on why the tools are wired the way
they are.

**Scenario (intentionally stretched):** a shipment of seed stock is ready for
export. Before it clears, an agent reads the shipment's free-text notes,
works out the genetic marker and destination country referenced in them,
checks the destination country's basic profile (used to pick the right
paperwork/ruleset), runs the marker and country through a legacy scoring
engine, and notifies the export team with a clearance decision.

The input is deliberately unstructured — the agent has to extract the gene
marker symbol and convert the destination country to an ISO code itself,
rather than receive them pre-parsed. That's the actual point of using an
agent here: a fixed BPMN sequence of connector calls can't casually do that
extraction and conversion step; an LLM can. The system prompt describes the
goal and constraints, not a numbered call sequence — the agent decides which
tools to call and when.

### Disclaimer

> NOTE: This demo uses public reference-genome and calculator services as
> stand-ins for a seed-genetics database and a compliance scoring engine.
> No real regulatory, health, or personal data is involved. Swap the SQL
> endpoint and the two REST endpoints for real systems in any non-demo use.

Nothing in this repo — the BPMN element names, the notification payload, or
this README — should be read as real regulatory guidance. It is illustrative
only.

## Zero-config LLM on Camunda SaaS

Running on Camunda SaaS? Skip the hassle of setting up an external AI
provider. This blueprint is pre-configured to use the **Camunda-provided
LLM** — a fully managed model that works out of the box. The required
secrets (`CAMUNDA_PROVIDED_LLM_API_ENDPOINT` and
`CAMUNDA_PROVIDED_LLM_API_KEY`) are automatically available on Camunda
SaaS — no external API keys, no third-party accounts, no configuration
required.

👉 Learn about the Camunda-provided LLM:
https://docs.camunda.io/docs/components/agentic-orchestration/camunda-provided-llm/

Just deploy your process to your SaaS cluster and start routing
intelligently from day one. The AI Agent connector in the BPMN file is
already wired up for it:

- **Provider**: `OpenAI Compatible`
- **API endpoint**: `{{secrets.CAMUNDA_PROVIDED_LLM_API_ENDPOINT}}`
- **API key**: `{{secrets.CAMUNDA_PROVIDED_LLM_API_KEY}}`
- **Model**: `{{secrets.CAMUNDA_PROVIDED_LLM_DEFAULT_MODEL}}` (SaaS-populated
  default; swap in any other [supported Camunda-provided LLM
  model](https://docs.camunda.io/docs/components/agentic-orchestration/camunda-provided-llm/#supported-models)
  with tool-calling support in the properties panel if you want a specific one)

## Deployment

### Option A — Camunda 8 SaaS (recommended)

1. Deploy `models/seed-export-compliance-agent.bpmn` and
   `models/seed-export-shipment-ready.form` to your cluster (Web Modeler
   "Deploy", or `zbctl`/`camunda-8-cli`).
2. No environment variables to set — Camunda-provided LLM secrets are
   already populated on SaaS trial and AI-features-enabled enterprise orgs.
3. All four tools work out of the box — none of them need the
   Self-Managed/Hybrid-only SOAP connector.

### Option B — Camunda 8 Run (local, Docker-based, self-managed)

1. Start [Camunda 8 Run](https://docs.camunda.io/docs/self-managed/quickstart/developer-quickstart/c8run/).
2. Set one environment variable for the LLM key, e.g. for Anthropic:
   ```
   export ANTHROPIC_API_KEY=sk-ant-...
   ```
   Camunda connector secrets read `secrets.<NAME>` from environment
   variables of the same name.
3. In the BPMN file (or via Web Modeler/Desktop Modeler properties panel),
   switch the AI Agent connector's provider inputs from the
   Camunda-provided-LLM (`openaiCompatible`) values to your provider, e.g.
   Anthropic:
   - `provider.type` → `anthropic`
   - `provider.anthropic.authentication.apiKey` → `{{secrets.ANTHROPIC_API_KEY}}`
   - `provider.anthropic.model.model` → e.g. `claude-sonnet-4-5-20250929`
4. Deploy `models/seed-export-compliance-agent.bpmn` and
   `models/seed-export-shipment-ready.form`. No connector besides the three
   built-in ones (SQL, GraphQL, REST) needs installing — REST is used
   twice, for tools 3 and 4.

## Starting an instance

The start event carries a **Camunda Form**, pre-filled with sample data —
the easiest way to start is Tasklist's or Web Modeler's "Start instance"
flow, which will render it directly. It asks for two fields:

```json
{
  "shipmentId": "SHIP-2026-0731",
  "shipmentNotes": "Shipment SHIP-2026-0731: seed stock ready for export to Brazil. Lab reference marker on the paperwork is TP53."
}
```

`shipmentNotes` is free text — write it like real shipment paperwork or an
email, not a structured record. The agent reads it to find the gene marker
symbol and the destination country (converting the country to its
ISO-3166 alpha-2 code itself). You can still start an instance by supplying
these two variables directly (Operate "Start instance", `zbctl`, or the
REST API) instead of using the form — both paths work.

Watch the agent work out what it needs from the notes, call the tools it
decides it needs, and reach a `cleared`/`flagged-for-review` decision,
visible end-to-end in Operate.

## Want to see the notification land live?

By default, tool 4 (`NotifyExportTeam`) posts to `https://httpbin.io/post`,
which just echoes the request back in its own JSON response — visible in
Operate's variable view, but not in a live feed anywhere else.

For a live demo, generate a fresh, session-specific URL at
[webhook.site](https://webhook.site/), open it in a browser tab, and replace
the `url` input on the `NotifyExportTeam` service task with that URL. Because
the URL is unique per session, it's intentionally **not** hardcoded into the
BPMN file — generate your own before a live demo.

## Etiquette note

The UCSC MySQL server (`genome-mysql.soe.ucsc.edu`) is shared public
infrastructure used by many researchers. This process only ever issues a
single, indexed `LIMIT 1` lookup per instance — please don't wrap it in a
retry loop or use it for load testing. The same restraint applies to the
GraphQL and REST endpoints: each is called at most once per process
instance (each tool task has a small, bounded job retry count of 2 —
3 attempts total, 5 seconds apart — to ride out one-off network blips on
these shared third-party services, not to hammer them).

## Design notes for anyone extending this example

- **Don't use the SOAP connector here.** It's Self-Managed/Hybrid only,
  never available on SaaS — a SOAP-based tool would conflict with the
  SaaS-first, zero-config-LLM design this demo leads with. `ComputeComplianceScore`
  uses the REST connector against `api.mathjs.org` instead.
- **Pass values derived from earlier tool results via `fromAi()`, not a
  direct FEEL reference to another tool's result variable.** Inside an AI
  Agent ad-hoc sub-process, a downstream tool's own `zeebe:input` mapping
  reading a prior sibling tool call's result directly (e.g.
  `length(markerRecord.geneSymbol) + ...`) is unreliable. The model already
  has the real prior results in its own conversation history (the ad-hoc
  sub-process feeds each tool's result back to it as part of the loop), so
  have it carry the value forward as a tool-call argument instead, e.g.
  `fromAi(toolCall.intA, "The character length of the gene symbol confirmed
  by VerifyGeneticMarker...", "number")`. Only variables set once, before
  the agent loop starts (the process start variables `shipmentId` and
  `shipmentNotes`), are safe to read directly; anything a sibling tool call
  produces mid-loop should go through `fromAi()`.
- **Treat `resultExpression` as a pure function of `response` only.** Never
  reference `toolCall.*` or other process variables inside it — not even a
  tool's own `fromAi()`-bound argument. This is why `ComputeComplianceScore`
  derives `complianceScore` straight from `response.body` (the number
  `api.mathjs.org` echoes back) rather than from its own `toolCall.*`
  arguments.
