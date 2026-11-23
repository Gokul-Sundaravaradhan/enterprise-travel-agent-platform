# EPIC SP03 — Stateless Streamable HTTP MCP Layer, Auth & Security Guardrails

**Status:** Placeholder — detailed task PRDs to be written when the epic starts (security scope expanded by the [threat model](../../02_AI_AND_MCP_SECURITY_THREAT_MODEL.md))
**Depends on:** SP02, and SP00 TASK-0.4 to TASK-0.6 (audit logging, telemetry, SBOM) when scheduled
**Master PRD:** [`../../00_MASTER_PRD.md`](../../00_MASTER_PRD.md)

## Goal

Replace per-request `stdio` subprocesses with network-addressable, horizontally scalable MCP services, and put a security boundary around the agent.

## Security Posture: the MCP Layer Is External

The flight and hotel MCP servers are our own and accept no external input. This epic nevertheless designs and **proves** the agent-side defenses as if every MCP server were an untrusted third party: it may describe its tools deceptively, change them after review, return hostile text, hang, flood, echo credentials or impersonate another server (rule R2 of the [threat model](../../02_AI_AND_MCP_SECURITY_THREAT_MODEL.md)). Every control is deterministic code, testable with a malicious server and a mock LLM, and free to run.

## Planned Tasks

| Task     | Summary                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TASK-3.1 | **Streamable HTTP transport**: run each MCP server as a stateless HTTP service; agent connects as a long-lived client. Transport hardening (C-10): HTTPS only, `Origin` and `Host` validation, no wildcard CORS, request size and time limits                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| TASK-3.2 | **JWT identity propagation** from the client through the agent to every MCP tool call and backing API. Tokens are short-lived and **audience-bound to one server**, and a server never forwards a received token downstream (no token passthrough, C-09)                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| TASK-3.3 | **Redis rate limiting** per user, per tool and per identity, with budget caps (C-10, C-12)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| TASK-3.4 | **PII redaction** on ingress, in logs and traces, and **egress DLP on tool arguments** so secrets and personal data cannot leave through the argument channel (C-08)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| TASK-3.5 | **Fast SLM intent router** at ingress to reject off-domain or hostile requests before any expensive model runs                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| TASK-3.6 | **Indirect prompt-injection defense** on tool outputs before they reach any model: structural separation, demarcation and screening of untrusted content (C-07)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| TASK-3.7 | **MCP trust boundary (external-server posture)**, new: agent-owned tool descriptions and schemas that are never forwarded from servers (C-01); server allow-list, TLS verification and Cloud Run identity (C-02); **tool manifest pinning** with quarantine on change, defeating rug pulls (C-03); namespacing and collision rejection (C-04); static scan of metadata for hidden Unicode and instruction patterns (C-05); result schema validation with size, depth and time limits (C-06); per-tool scopes; audit logging of every call and decision (C-19). Seeds the AI component inventory (`docs/security/ai-inventory.yaml`) with every MCP server and its tool-manifest hash, which SP06 turns into the AI-BOM |
| TASK-3.8 | **Adversarial MCP test harness and regression suite**, new: a test-only malicious MCP server and deterministic tests for scenarios S-01 to S-15 of the threat model, run with a mock LLM as a **required CI check** (C-21). Test names carry the scenario ID so coverage of the catalog is auditable, and a test fails if its control is removed                                                                                                                                                                                                                                                                                                                                                                       |
| TASK-3.9 | **MCP tool contracts (server-side)**, new: every tool validates its input with a typed Pydantic model (strict dates, IATA-code patterns, length and range bounds, **unknown arguments rejected**) and returns a typed, schema-described result instead of loose text; failures use a **structured error taxonomy** that never exposes internal URLs or stack traces; delivered together with TASK-3.1 because the servers are rewritten for HTTP anyway; backed by **contract tests** that fail if the agent's pinned tool manifest and a server's live `tools/list` disagree (C-24, LLM10). Design below                                                                                                              |

**Order.** TASK-3.1 and TASK-3.2 first (transport and identity), with TASK-3.9 (tool contracts) delivered alongside TASK-3.1 because the servers are rewritten for HTTP at that point. TASK-3.7 and TASK-3.8 then proceed together, test-first: each scenario test is written before the control it proves.

## MCP Tool Contracts (Server-Side Design)

The agent validates every call before it is sent (SP04 TASK-4.4), but the server must **never rely on that**: from its own point of view the caller is an untrusted client. Today the tools take plain `str` and `int` parameters, so type mismatches are caught by the SDK, but nothing checks that an "airport code" looks like one or that a "date" is a date, and a hallucinated argument name is probably dropped silently. A failing call surfaces as a raw HTTP exception that includes an internal URL.

**Input rules (examples for the existing tools)**

| Parameter               | Rule                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| `origin`, `destination` | Three letters after normalization (IATA pattern); anything else is `invalid_arguments`      |
| `date`, `check_in_date` | Strict ISO `YYYY-MM-DD` (not a timestamp, a number or free text), within a sane range       |
| `city`                  | Bounded length, letters, spaces and hyphens only, no control characters                     |
| `flight_id`, `hotel_id` | Integer between 1 and the database maximum                                                  |
| Any unknown argument    | **Rejected** (forbid extras), so an invented argument fails loudly instead of being ignored |

**Error taxonomy.** Errors are returned as structured tool errors with a stable code, a safe message, optional field-level details and a `retryable` flag. They never contain internal URLs, hostnames, stack traces or upstream response bodies.

| Code                   | Meaning                                                                             | Agent behavior                                               |
| ---------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `invalid_arguments`    | Input failed validation (field and rule are named)                                  | May be repaired and retried within the bounded repair budget |
| `not_found`            | A lookup by ID found nothing (a search with no hits is an empty list, not an error) | No retry; report to the user                                 |
| `upstream_unavailable` | The backing API or database failed or timed out                                     | Bounded retry with backoff, then fail with a clear message   |
| `rate_limited`         | A limit from TASK-3.3 was hit                                                       | Wait as indicated, bounded                                   |
| `unauthorized`         | Missing, expired or wrong-audience credentials                                      | No retry; stop and surface                                   |
| `internal_error`       | Anything unexpected                                                                 | At most one retry; generic message                           |

The REST APIs already return a 422 for a bad date (TASK-2.1); the MCP server maps that to `invalid_arguments`, a 404 to `not_found`, and 5xx or timeouts to `upstream_unavailable`.

**Output contract.** Each tool declares a typed result (for example a `Flight` and a list of flights) so the client receives structured content and a schema, not text blocks it has to reassemble. This removes the workaround the agent uses today for multi-block list results. Whether the 2.x SDK supports declared output schemas and strict extra-argument rejection on `MCPServer` tools directly, or needs a Pydantic model parameter, is verified when the task starts.

**Contract tests (the glue between the two sides).** The agent owns its tool manifest (C-01, C-03), so it does not import server code. A CI test compares the agent's pinned manifest with each server's live `tools/list`, and fails on any difference, so the two cannot drift silently. Further tests cover boundary values for every input rule (property-based testing with an open-source library is a good fit), the error taxonomy, and the 422-to-`invalid_arguments` mapping.

## Exit Criteria

- Any replica of any MCP server can serve any request; no per-session server state.
- Unauthenticated, wrong-audience and over-limit requests are rejected with correct status codes.
- The model never sees a tool description that did not come from this repository.
- A tool whose definition changes (name, description or input schema) is quarantined and an alert is logged; the change requires a reviewed update to the manifest lock.
- No server can cause a token to be forwarded; a probe server that requests or echoes the caller's token is refused.
- The adversarial regression suite (scenarios S-01 to S-15) passes in CI, and an injection embedded in tool data does not change agent behavior.
- The AI component inventory lists every MCP server, its endpoint, scopes and tool-manifest hash.
- Every tool validates its input with a typed model (unknown arguments rejected, strict dates, bounded values) and returns a typed result; failures use the structured error taxonomy and never expose internal URLs or stack traces.
- A contract test fails if the agent's pinned manifest and a server's live `tools/list` disagree.
- All of the above runs with no paid service: a mock LLM provider and open-source tooling only.

## Key Risks

MCP SDK 2.x transport and middleware APIs are new; false positives from the router and injection filters; added latency from guardrail hops; heuristic scanners (C-05, C-07) have false negatives, so they are defense in depth only and the structural controls (C-01, C-03, C-07) carry the weight; manifest pinning adds a review step whenever a tool legitimately changes.
