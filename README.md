# Private RAG Copilot — Monk demo kit

A fully self-hosted RAG/chat stack, composed and wired by Monk with **one command**.
No cloud API keys — the LLM runs locally via Ollama.

## What's in the stack

The backends now **inherit official monk.io hub packages** (`inherits:`) instead of
self-contained image wrappers. Only **LightRAG** stays a custom wrapper, because no
hub package exists for it yet. **Langfuse** is self-contained: `langfuse/server`
brings its own database (`langfuse/database`), declares its own connection +
`depends.wait-for`, builds `DATABASE_URL` internally, and does **not** use the
shared Postgres/Redis.

| Component        | Source                                | Inherits / image                | Service wired via | Role |
|------------------|---------------------------------------|---------------------------------|-------------------|------|
| `ollama`         | hub package                           | `inherits: ollama/ollama`       | `web` :11434      | Local LLM + embeddings (OpenAI-compatible) |
| `qdrant`         | hub package                           | `inherits: qdrant/server`       | `qdrant` :6333    | Vector database |
| `postgres`       | hub package                           | `inherits: postgresql/db`       | `postgres` :5432  | Relational state (app only) |
| `redis`          | hub package                           | `inherits: redis/redis`         | `redis-svc` :6379 | Cache / KV |
| `langfuse/server`| hub package (self-contained, own DB)  | `langfuse/server`               | `langfuse` :3000  | LLM observability dashboard |
| `langfuse/database` | hub package (Langfuse's own DB)    | `langfuse/database`             | `postgres` (internal) | Langfuse metadata store |
| `app`            | **custom wrapper** (no hub package)   | `ghcr.io/hkuds/lightrag:latest` | host :9621        | LightRAG UI — RAG over the backends above |
| `stack`          | process-group entrypoint              | —                               | —                 | Brings up all of the above |

### Inherited service names

When inheriting hub packages, connections must target the **package's** service
names (not the names from the old wrappers):

| Runnable          | Inherits          | Service name |
|-------------------|-------------------|--------------|
| `rag-copilot/ollama`   | `ollama/ollama`   | `web`        |
| `rag-copilot/qdrant`   | `qdrant/server`   | `qdrant`     |
| `rag-copilot/postgres` | `postgresql/db`   | `postgres`   |
| `rag-copilot/redis`    | `redis/redis`     | `redis-svc`  |

The `app` runnable connects to each backend through Monk **`connections`** (resolved
over the overlay DNS at runtime via `connection-hostname(...)` / `connection-port(...)`),
not hardcoded hostnames. See the header comment in `stack.yaml` for the wiring idiom.

### What the inherited runnables override

- **ollama** — adds a first-boot `bash` that runs `ollama serve` and pulls the
  configured chat + embedding models (the base package only runs `serve`), plus a
  `/root/.ollama` volume so pulled models survive restarts. The readiness probe
  targets the inherited `web` service.
- **postgres** — overrides the base variables `db_user` / `db_name`
  (`ragcopilot`), `db_pass` (from the `postgres-password` secret) and pins
  `version: "16"`.
- **qdrant** / **redis** — inherited as-is (volume and config already built into
  the packages).

## Runbook

```bash
# 1. Load the kit
monk load ./MANIFEST          # from kits/rag-copilot/, or pass the absolute path

# 2. Provide the ONE user secret (one-time, global scope)
monk secrets add -g postgres-password

# 3. Bring up the whole stack with ONE command
monk run rag-copilot/stack
```

`postgres-password` is the **only** user-provided secret. Langfuse's
`nextauth-secret` and `salt` default for the demo (`mysecret` / `mysalt` in the
package), so they are **not** collected here — override them in `langfuse/server`
variables if you need production-grade values.

Ollama pulls the configured models automatically on first boot (see the container
`bash` block). They are cached on the persistent volume, so restarts/upgrades are fast.

UIs after it's up: **LightRAG** on `:9621`, **Langfuse** on `:3000`.

## Demo-tunable knobs (in `stack.yaml` `variables`)

- `llm-model` (default `llama3.1`) — set on both `ollama` and `app`
- `embedding-model` (default `nomic-embed-text`) — set on both `ollama` and `app`
- Postgres `db_user` / `db_name` (default `ragcopilot`)

## Suggested 4-act demo flow

1. **Compose** — `monk run rag-copilot/stack` → full stack up, auto-wired from hub
   packages. Ask a question in the LightRAG UI, answered by the local model. Show
   the trace in Langfuse.
2. **Swap** — change the `qdrant` wiring to another vector DB package (one
   connection edge) and redeploy.
3. **Portability** — run the same template on a cloud VM (add a `nodes:` block /
   ingress for TLS). Same artifact, now public.
4. **Day-2** — bump a component's `image-tag` / package version; persistent volumes
   survive the upgrade.

## Known gaps / things to verify before a live run

- **LightRAG storage wiring.** The app selects backends via the
  `LIGHTRAG_*_STORAGE` selector env vars (the names the image actually reads —
  an earlier `VECTOR_STORAGE` name was wrong and silently fell back to the
  Nano/JSON/NetworkX file defaults): vector → Qdrant
  (`LIGHTRAG_VECTOR_STORAGE=QdrantVectorDBStorage`, `QDRANT_URL`), KV → Redis
  (`LIGHTRAG_KV_STORAGE=RedisKVStorage`, `REDIS_URI`), and doc-status → Postgres
  (`LIGHTRAG_DOC_STATUS_STORAGE=PGDocStatusStorage`, `POSTGRES_*`). The graph
  store stays on the default `NetworkXStorage`: this stack has no graph DB
  (Neo4j/Memgraph) and `PGGraphStorage` needs the Apache AGE extension, which the
  stock postgres image lacks.
- **Readiness endpoint** assumes Ollama `/api/tags` returns 200 (on the inherited
  `web` service).
- **Deployed once locally; not yet re-verified end-to-end.** The kit has been
  brought up locally and the backends are wired, but a full RAG round-trip with
  the corrected `LIGHTRAG_*_STORAGE` selectors has not been re-verified yet.
