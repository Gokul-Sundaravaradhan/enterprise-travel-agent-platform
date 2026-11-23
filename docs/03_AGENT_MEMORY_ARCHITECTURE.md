# 03 — Agent Memory Architecture

| Field               | Value                                                                                                                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Status              | Decision record, planned ahead of the work. Implementation is SP04 TASK-4.7 (with inputs from SP02, SP03 and SP06). Revisit at the start of SP04                                       |
| Decision            | Persist **thread state** (required) and a **small, structured, application-controlled** long-term memory. **Do not** build MemGPT-style, model-managed memory                          |
| Standing constraint | **Nothing may cost money**: Postgres and LangGraph primitives only, deterministic writes, no extra LLM calls for memory management                                                     |
| Related             | [`02_AI_AND_MCP_SECURITY_THREAT_MODEL.md`](02_AI_AND_MCP_SECURITY_THREAT_MODEL.md) (controls C-25 to C-27, scenarios M-01 to M-08), [`00_MASTER_PRD.md`](00_MASTER_PRD.md), SP04, SP06 |

---

## 1. Terms

Four things are easy to confuse, and each has a different home.

| Concept                       | What it is                                                                              | Where it lives                                                                     | Persistence                            |
| ----------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------- |
| **Thread state** (short-term) | The state of one conversation or run: messages, plan, intermediate results              | LangGraph **checkpointer** in Postgres (SP04 TASK-4.2)                             | Per thread, with a retention limit     |
| **Long-term memory**          | Facts about a user that should survive across threads, such as a preferred home airport | A **store** keyed by user, in Postgres (this document)                             | Per user, with expiry and user control |
| **Knowledge retrieval**       | Corporate policy documents the agent consults                                           | Policy RAG (SP05). This is retrieval, **not** memory                               | Corpus managed by ingestion            |
| **System of record**          | What actually happened: bookings, cancellations, approvals                              | The business databases (SP02). **Trip history comes from here, never from memory** | Authoritative                          |

## 2. Do We Need Long-Term Memory?

**Thread persistence: yes, and it is already planned** (checkpointer, SP04 TASK-4.2). It gives crash recovery, human-approval pauses and resumable runs.

**Cross-session memory: yes, but narrowly.** A corporate travel agent has genuine, low-risk use cases: "book my usual flight" (home airport, preferred airline, cabin), hotel preferences (area, nightly budget), and recurring constraints. These save the user from repeating themselves, and they make the project demonstrate something most agent demos skip: memory that is **scoped, auditable and deletable**.

**What it is not for:** reconstructing trip history (use bookings), remembering whatever was said (raw transcripts are a liability), storing anything sensitive (see 5.3), or giving the model a free-form notebook.

## 3. Options Assessed

| Option                                                    | Verdict          | Reasoning                                                                                                                                                                                                                                                                                                                                                                            |
| --------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A. Checkpointer only (no long-term memory)                | Fallback         | Safe and simple, but the agent forgets everything between sessions. Acceptable if time runs out; it is stage M0                                                                                                                                                                                                                                                                      |
| **B. Structured, application-controlled profile memory**  | **Chosen**       | Typed keys, deterministic writes, per-user scoping enforced in the database. Testable without a model, free to run, and defensible in a security review                                                                                                                                                                                                                              |
| C. MemGPT/Letta-style self-managed memory                 | **Rejected**     | The idea (tiered memory the LLM edits through tool calls) is the opposite of this project's rules. The model decides what to store, so a hijacked model can **write** to memory (poisoning, ASI06); it costs extra LLM calls (the $0 rule); and scoping and behavior become probabilistic. **Borrow the concepts** (tiering, compaction under token pressure), **not the mechanism** |
| D. Third-party memory libraries or hosted memory services | Rejected for now | They add a dependency and often a paid or rate-limited backend for something Postgres and LangGraph's own store already do. Their ideas (durability tiers, validity windows, version chains, consolidation) are borrowed in section 5                                                                                                                                                |
| E. Semantic memory over raw conversations (vector recall) | Deferred (M3)    | Biggest privacy and poisoning surface, an embedding model to run, and little value for structured travel preferences. Only if B proves insufficient, behind a flag                                                                                                                                                                                                                   |

## 4. Staged Plan

Each stage ships independently and is worth stopping at.

| Stage  | Scope                                                                                                                                          | Owner                            |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| **M0** | Thread persistence with the Postgres checkpointer, a retention limit on old threads, and no personal data beyond what the conversation needs   | SP04 TASK-4.2                    |
| **M1** | **Structured profile memory**: typed preferences, deterministic writes with confirmation, per-user isolation, expiry, list and forget controls | **SP04 TASK-4.7 (new)**          |
| **M2** | **Thread summaries** that keep long conversations inside the context budget, with pinned structured facts                                      | SP06 TASK-6.2                    |
| M3     | _Optional stretch:_ semantic recall of consented, summarized episodes, behind a flag and the same scoping rules                                | Only if M1 and M2 are not enough |

## 5. Design (stages M1 and M2)

### 5.1 Technology

- **LangGraph checkpointer plus store.** The checkpointer holds per-thread state; the store holds cross-thread memory under tuple **namespaces** such as `("users", <user_id>, "preferences")`. The Postgres-backed store keeps everything in the database already in use. Whether the store offers built-in time-to-live is verified at implementation; if not, the schema carries an `expires_at` column and a small sweeper enforces it.
- **A dedicated `memory` schema** in the shared Postgres database, following the one-schema-per-component rule from SP02, owned by the orchestration side (not by the flight or hotel services).
- **No vector index in M1 or M2.** Preferences are keyed facts; semantic search is M3.

### 5.2 Data model (sketch)

| Table                | Purpose                                                                                                                                                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `memory.preferences` | One row per user and key: `user_id`, `tenant_id`, `key` (from the registry), `value` (validated JSON), `source` (`user_confirmed`, `booking_event`), `created_at`, `updated_at`, `last_used_at`, `expires_at`, `version` |
| `memory.events`      | Append-only audit of every write, merge, expiry and deletion: who, what key, when, why, and the previous version number (values only where non-sensitive)                                                                |

A **key registry** in code defines every allowed key with its Pydantic type and expiry class, for example `home_airport` (an IATA code), `preferred_airlines` (a short list of IATA airline codes), `cabin_class` (an enum), `hotel_area` and `hotel_nightly_budget` (a bounded number). A key that is not in the registry cannot be written.

### 5.3 What is never stored

Payment card data, passport and government identifiers, health information, loyalty-programme credentials, passwords and tokens, and free-form personal narrative. If a user types one of these, it is not persisted to memory, and (outside the thread state it already sits in) it is redacted from logs by the SP00 logging rules. This list is a deny-list enforced in code and tested.

### 5.4 Write policy: who and what may write memory

| #   | Rule                                                                                                                                                                                                                                                      |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| W1  | **Only deterministic code writes memory.** The model never has a "save memory" tool. At most it proposes a candidate fact as typed data, which application code validates, and then asks the user                                                         |
| W2  | **Trusted sources only:** a statement made by the authenticated user in the chat, confirmed by them, or a system event such as a confirmed booking. **Never** tool output, retrieved documents, or text from any other channel (defeats poisoning, ASI06) |
| W3  | **Confirmation:** new or changed preferences are shown to the user ("Save home airport as BLR?") and stored only on a yes. A silent write is never allowed                                                                                                |
| W4  | **Registry validation:** key, type, bounds and size are checked; unknown or oversized values are rejected                                                                                                                                                 |
| W5  | **The server stamps ownership:** `user_id` and `tenant_id` come from the authenticated principal, never from the model or the request body                                                                                                                |
| W6  | **Rate and size limits** per user bound growth and abuse                                                                                                                                                                                                  |

### 5.5 Scoping and isolation: preventing leaks on read

| #   | Rule                                                                                                                                                                                                             |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| S1  | **The namespace is derived server-side** from the authenticated principal. The model cannot name a user, and a request for another user's memory is not expressible                                              |
| S2  | **Row-Level Security in Postgres** on the memory tables, using a per-transaction setting for the current user, so a coding mistake in the application still cannot return another user's rows (defense in depth) |
| S3  | **Filter before ranking.** If semantic search ever exists (M3), the user filter is applied inside the query before similarity ranking, never after                                                               |
| S4  | **No cross-user or cross-tenant sharing.** There is no global or "organization" memory in M1; shared knowledge is the policy RAG                                                                                 |
| S5  | **Direct-SQL test:** a test connects as the application role with another user's identity set and asserts zero rows, independent of the application code                                                         |
| S6  | **Memory content is never logged** beyond key names and version numbers                                                                                                                                          |

### 5.6 Merge, decay and compaction: avoiding memory rot

| Mechanism                  | Rule                                                                                                                                                                                                                              |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Upsert by key**          | One current value per user and key. A new confirmed value replaces the old one and bumps `version`; the previous value goes to the audit event, not into a growing pile                                                           |
| **Contradiction handling** | A conflicting value is never silently merged. It is shown to the user ("You said BLR before; change to DEL?") and applied only on confirmation. A booking event may update a derived preference only with the same confirmation   |
| **Decay and expiry**       | Each key class has a time-to-live (for example 12 months for stable preferences). Using a preference refreshes `last_used_at`; an expired one is removed and, when relevant, re-confirmed rather than assumed                     |
| **Consolidation job**      | A deterministic periodic job removes expired rows, enforces per-user limits and writes audit events. It needs no LLM                                                                                                              |
| **Thread compaction (M2)** | Long threads are summarized into a rolling summary **plus pinned structured facts** (dates, routes, booking IDs, confirmed choices) that are never replaced by prose. The summarizer's output is schema-validated (SP06 TASK-6.2) |
| **Checkpoint retention**   | Old threads are deleted after a retention period, so conversation text does not become an unbounded second memory                                                                                                                 |

### 5.7 Recall: reading memory back safely

- **Deterministic load.** At the start of a turn, application code loads the user's active preferences (a small, bounded set). The model does not search memory.
- **Untrusted when read back.** Recalled values are inserted as demarcated data with provenance, never as system instructions, and never executed as commands. A value that looks like an instruction is still just a value, because it must have passed the typed registry.
- **Transparent.** When a saved preference influences an answer, the agent says so ("using your saved home airport, BLR"), and the user can correct it.

### 5.8 User controls

List, correct, export and **forget** (one key or everything). Forgetting must reach every copy: the preference rows, the audit values, the user's thread checkpoints and any summaries. Completeness is a tested requirement, not a promise.

## 6. Threat Mapping

Scenarios and controls are defined in the [threat model](02_AI_AND_MCP_SECURITY_THREAT_MODEL.md): scenarios M-01 to M-08, controls C-25 (write policy), C-26 (isolation) and C-27 (retention, forgetting and recall handling), with C-07 and C-13 for untrusted content and state integrity. Standards covered: LLM01 (including stored prompt injection), LLM02, LLM05, ASI06 and ASI03.

## 7. Test Plan (deterministic, free)

| Test                                            | Proves                                                                                                 |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Cross-user leakage (agent-level and direct SQL) | User A cannot read user B's memory by any path, including a prompt that asks for it                    |
| Poisoning attempts                              | Text from a tool result or retrieved document cannot cause a write; only confirmed user statements can |
| Stored-injection recall                         | A stored value shaped like an instruction is rendered as data and does not change agent behavior       |
| Sensitive-data deny-list                        | A pasted card or passport number is never persisted to memory                                          |
| Registry and bounds                             | Unknown keys, wrong types and oversized values are rejected                                            |
| Merge, conflict and expiry                      | Upserts bump versions, conflicts require confirmation, expired rows are removed and re-confirmed       |
| Forgetting completeness                         | After "forget", no copy remains in preferences, audit values, checkpoints or summaries                 |
| Growth limits                                   | Per-user limits and rate limits hold                                                                   |
| Compaction (M2)                                 | Pinned facts survive summarization unchanged                                                           |

All of these run against a real Postgres and a mock LLM, so they are free and deterministic.

## 8. Cost and Free-Tier Check

Postgres (already provisioned) and LangGraph's own primitives. No extra LLM calls, because writes are deterministic and summaries (M2) run on the free routing path. No embedding model until M3. Storage is a few kilobytes per user. **No paid service is introduced.**

## 9. Where It Lands in the Plan

| Item                                                      | Task                                                                               |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Stable opaque `user_id` and tenant used as the scope key  | SP02 TASK-2.2 (users and role tiers)                                               |
| Authenticated principal that stamps ownership             | SP03 TASK-3.2 (JWT identity propagation)                                           |
| Thread persistence and retention                          | SP04 TASK-4.2                                                                      |
| **Structured long-term memory (M1)**                      | **SP04 TASK-4.7 (new)**                                                            |
| Thread summaries with pinned facts (M2)                   | SP06 TASK-6.2                                                                      |
| Leakage, poisoning and forgetting tests in the eval suite | SP06 TASK-6.7                                                                      |
| The memory store listed in the AI inventory               | SP06 TASK-6.6 (AI-BOM)                                                             |
| Controls and scenarios                                    | [Threat model](02_AI_AND_MCP_SECURITY_THREAT_MODEL.md): C-25 to C-27, M-01 to M-08 |

## 10. Risks and Open Questions

| Item                                                                                                        | Handling                                                                                                                          |
| ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Over-engineering for a portfolio project                                                                    | Staged plan: M0 and M1 are small; M2 and M3 are optional and independent                                                          |
| Confirmation prompts feel tedious                                                                           | Only new or changed preferences prompt; the registry stays small                                                                  |
| LangGraph store features (time-to-live, namespaces, Postgres RLS compatibility) differ from what is assumed | Verify in the TASK-4.7 PRD; the schema carries its own expiry and the isolation does not depend on store internals                |
| Forgetting must reach checkpoints, which are managed by the framework                                       | Checkpoint retention plus a delete-by-user path, tested for completeness                                                          |
| Personal-data obligations if this ever handled real users                                                   | The design (consent, minimization, access, deletion, audit) follows common privacy principles; it is not a legal compliance claim |
| The framework and its memory APIs change quickly                                                            | Memory sits behind a small interface in the orchestrator, so the store can change without touching agent logic                    |

## 11. Deliberately Not Doing

Model-written memory, a free-form notebook, global or shared memory, vector recall of raw chats (before M3), storing secrets or identifiers, or letting memory override a fact from the system of record.
