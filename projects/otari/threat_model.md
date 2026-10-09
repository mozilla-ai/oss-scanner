# Threat model

## What this project does and where untrusted input enters
otari is a self-hostable, OpenAI-compatible LLM gateway: it sits between client applications and LLM providers (OpenAI, Anthropic, Bedrock, and others via the any-llm SDK), and adds API key management, per-user and per-organization budgets, usage tracking, built-in tools (web search, web fetch, code execution, MCP), and an admin dashboard.

Untrusted input enters through:
- The HTTP API (`src/gateway/api/routes/`): `/v1/chat/completions`, `/v1/messages`, `/v1/responses`, files, MCP, hooks. Callers hold a gateway API key but are not trusted beyond the budget and permissions attached to it.
- Request bodies forwarded to providers, and provider responses (including streamed SSE) flowing back.
- Model output that drives built-in tools: URLs passed to web fetch, queries to web search, code to the sandbox, MCP tool calls. Treat model output as attacker-controlled (prompt injection).
- Uploaded files (PDF/image text extraction).
- The dashboard sign-in flow (OAuth) and session cookies (`web/`, `src/gateway/api/routes/`).
- In hybrid mode, responses from the otari.ai control plane (`docs/hybrid-mode-protocol.md`).

## Components that matter most / least
Most important:
- Authentication and key handling: master key, API key verification, dashboard sessions, OAuth.
- Tenant and budget isolation: one key, user or organization reading or spending against another's data or budget; budget reservation bypass (`src/gateway/services/budgets/`).
- SSRF in web fetch, provider base URLs, MCP server URLs and webhooks (`url_safety`).
- Leaking secrets: provider credentials, API keys or internal error details in responses or logs.
- Authorization on management routes (users, keys, budgets, pricing, providers).

Less important: `scripts/`, `tests/`, load-test tooling, docs. `any-search/` and `any-fetch/` are in scope as libraries the gateway calls.

## How to exercise it
- `otari serve` starts the gateway (config in `config.yml`, see `docs/`); a SQLite `database_url` works without PostgreSQL.
- `tests/unit` runs offline. `tests/integration` needs PostgreSQL (`TEST_DATABASE_URL`).
- `scripts/oss_edition_smoke.py` boots the packaged CLI against a mock provider and walks key creation, a completion and usage recording; standard library only, SQLite by default.
- The OpenAPI spec is `docs/public/openapi.json`.

## How you rate severity
- Critical: unauthenticated access to management APIs, cross-tenant data access, extraction of provider credentials or other users' API keys, RCE on the gateway host.
- High: authenticated privilege escalation, budget bypass allowing unbounded spend, SSRF reaching internal networks or cloud metadata, sandbox escape from code execution into the gateway.
- Medium: leaks of internal error details or non-secret metadata, DoS by a single authenticated client.
- Low: issues requiring an operator to misconfigure deliberately.

## Anything to leave alone
- The one-time bootstrap master key printed to stdout at first start is deliberate.
- Provider-side behavior of the any-llm SDK itself belongs upstream (mozilla-ai/any-llm), unless otari's use of it creates the bug.
