---
name: rt-security-playbook
description: Hunter's playbook for the rt-security red-team agent, organized by OWASP Top 10 (2025), OWASP LLM Top 10 (2026), OWASP API Security Top 10 (2023), and OWASP Top 10 for Agentic Applications (for 2026) with sources, sinks, grep seeds, evidence bars, and false-positive traps. Reference material only. Do not activate on your own; it is preloaded into rt-security by the /red-team command.
---

# rt-security Playbook

For each category: where untrusted data enters (**Sources**), where it hurts (**Sinks**), what to **Grep**, what a **Trail** must show to count, and **Traps** a skeptic will use to refute you. A claim needs a cited source-to-sink trail. A missing control with no reachable path is not a claim.

Grep seeds are starting points; adapt to the stack. Always read the surrounding code before claiming.

## Universal evidence bar
- Name the attacker and their access (anonymous, user, tenant admin, insider, third-party service, model output). That is likelihood, not severity.
- Cite every hop as an absolute `path:line`: entry point, each transform or check, sink.
- Show the check you expected is absent or bypassable, not merely that you did not find it in one file. Look at middleware, decorators, base classes, framework defaults, and gateway config.
- Universal traps: dead or test-only code; the value is server-set or a constant; the framework escapes or parameterizes by default; the sink is only reachable by the trusted operator; the "secret" is a placeholder or public key.

## Part A: OWASP Top 10 (2025)
Sections use the 2025 IDs with the 2021 IDs in brackets. SSRF (2021 A10) is folded into A01:2025; Supply Chain Failures widens 2021's Vulnerable Components; Mishandling of Exceptional Conditions is new.

### A01:2025 Broken Access Control (IDOR, privilege escalation, CSRF, CORS) [2021 A01]
- **Sources:** object IDs in path/query/body, role or tenant fields in tokens or bodies, headers (`X-User-Id`, `X-Forwarded-*`), mass-assigned fields.
- **Sinks:** DB fetch/update/delete by ID, file serving, admin actions, cross-tenant queries.
- **Grep:** `findById|findOne|get\(.*params|req\.params|req\.body\.(role|isAdmin|userId|tenant)|@PreAuthorize|login_required|permission_classes|authorize|skip_before_action|csrf_exempt|Access-Control-Allow-Origin|allowedOrigins`.
- **Trail:** route to handler to query with no ownership/role predicate on the path (check middleware and the query's `WHERE`), and a second identity that can reach the first's object.
- **Traps:** ownership enforced in the repository, ORM scope, row-level security, or gateway; IDs are unguessable but ownership is still checked later; "public" objects by design; CORS `*` without credentials.

### A01:2025 (SSRF case) Server-Side Request Forgery [2021 A10]
- **Sources:** URLs or hosts in body/query/headers, webhook targets, image/feed/import fetchers, redirects, file URLs in documents.
- **Sinks:** HTTP clients, URL fetchers, headless browsers, PDF/image renderers, SSO metadata fetch.
- **Grep:** `requests\.(get|post)|urllib|axios|fetch\(|http\.Get|HttpClient|curl_exec|file_get_contents\(|URL\(|ImageIO\.read|puppeteer|wkhtmlto|webhook|callback_?url`.
- **Trail:** attacker-chosen URL reaches the client with no allow-list or with a bypassable check (string prefix, hostname-only without resolving IPs, redirects followed), and the response or side effect returns to the attacker or reaches an internal service/metadata endpoint.
- **Traps:** client-side fetch/URL in browser code is not SSRF; host is fixed and only the path varies; strict allow-list on resolved IP; response never returned and no state change (blind SSRF is lower likelihood of impact, say so); URL supplied only by an admin.

### A02:2025 Security Misconfiguration [2021 A05]
- **Sources:** config, env, defaults, deployment manifests, error output.
- **Sinks:** debug mode, default credentials, open admin/actuator endpoints, permissive CORS/CSP, verbose errors, public buckets, container/IaC permissions.
- **Grep:** `DEBUG\s*=\s*True|app\.debug|actuator|/admin|/debug|swagger|default.*(password|secret)|0\.0\.0\.0|AllowAny|privileged:\s*true|publicRead|Access-Control-Allow-Credentials|stack.*trace|printStackTrace`.
- **Trail:** the setting is the one actually deployed (prod config or the default when unset) and exposes a concrete capability or secret to a named attacker.
- **Traps:** dev/test-only config; "header missing" with no exploit; internal-only listener; value overridden in the production manifest.

### A03:2025 Software Supply Chain Failures (vulnerable components, dependency risks) [2021 A06]
- **Sources:** manifests and lockfiles; vendored code.
- **Sinks:** the vulnerable function in the dependency.
- **Grep:** read `package.json|package-lock.json|requirements*.txt|poetry.lock|go.sum|Cargo.lock|pom.xml|Gemfile.lock`, then grep for calls to the affected API.
- **Trail:** a specific advisory-level flaw in a pinned version, plus a cited call from slice code with attacker-influenced input to the vulnerable function. You cannot run audits: state the version and flaw from knowledge; an advisory recalled from memory without certainty is a Hunch.
- **Traps:** dev-only dependency; vulnerable function never called; version in the manifest is a range resolved to a fixed version; unsure of the advisory (then it's a Hunch).

### A04:2025 Cryptographic Failures [2021 A02]
- **Sources:** passwords, tokens, PII, keys, anything stored or sent.
- **Sinks:** hash/encrypt/sign/verify calls, random generators, TLS config, key storage.
- **Grep:** `md5|sha1|\bDES\b|RC4|ECB|createCipher\b|Math\.random|random\.random|rand\(\)|rejectUnauthorized.*false|verify\s*=\s*False|InsecureSkipVerify|jwt\.decode\(.*verify|alg.*none|== .*(token|hmac|signature)`.
- **Trail:** the weak primitive protects something security-relevant (password, session, reset token, signature) and the attacker can reach or observe the output; or a secret/token compared non-constant-time on an attacker-reachable check.
- **Traps:** MD5/SHA1 for cache keys or ETags; `Math.random` for non-security IDs; test-only TLS bypass; JWT verification done by a library wrapper elsewhere.

### A05:2025 Injection (SQL, NoSQL, command, template, XSS, header, LDAP, XXE) [2021 A03]
- **Sources:** any request data, file contents, DB rows written by users, third-party and model output.
- **Sinks:** string-built SQL/NoSQL operators, `exec`/shell, `eval`, template compile, HTML/DOM write, HTTP header/redirect, XML parser, regex (ReDoS).
- **Grep:** ``execute\(\s*f["']|execute\([^,)]*(\+|%\s*\()|query\(\s*[`"'].*\$\{|raw\(|\$where|\$ne|child_process|(^|[^.\w])exec\(|system\(|os\.popen|shell=True|Runtime\.exec|eval\(|new Function|render_template_string|innerHTML|dangerouslySetInnerHTML|v-html|\|safe|html_safe|Location:|res\.redirect|DocumentBuilder|resolveEntities``.
- **Trail:** attacker value reaches the sink without parameterization or context-correct encoding, and the sanitizer on the path (if any) does not cover that context or character set.
- **Traps:** values from an enum/allow-list or parsed to int; parameterized queries where only a table/column name is dynamic from a constant; framework auto-escaping; output to a non-HTML context; input only an admin controls.

### A06:2025 Insecure Design (business logic, abuse of flows) [2021 A04]
- **Sources:** step order, quantities, prices, coupon/ID reuse, replayed requests, parallel requests.
- **Sinks:** money/credit/quota/inventory changes, state machines, password reset, invitations, approvals.
- **Grep:** `price|amount|quantity|discount|coupon|balance|credit|status\s*=|approve|transfer|refund|invite|reset`.
- **Trail:** a legitimate-looking sequence yields an unauthorized outcome (negative quantity, skipped step, double spend, client-supplied price accepted), with each server-side check cited as missing.
- **Traps:** server recomputes the value later; check enforced in a transaction or DB constraint; flow guarded by a state machine in another module.

### A07:2025 Authentication Failures [2021 A07]
- **Sources:** credentials, session IDs, reset/invite/OTP tokens, JWTs, API keys, cookies.
- **Sinks:** login, token issue/verify, session create/destroy, password reset, MFA, remember-me.
- **Grep:** `jwt\.(verify|decode)|session|cookie|httpOnly|sameSite|reset.*token|otp|login|bcrypt|compare|expires|secret.*=|SECRET_KEY`.
- **Trail:** a concrete way to authenticate as, or take over, another identity: unverified JWT, guessable or reusable token, no expiry or invalidation, session fixation, user enumeration that feeds a working attack, hardcoded signing secret used in prod.
- **Traps:** lockout or rate limit in front of the route; token is single-use and checked elsewhere; hardcoded secret is test-only; missing-flag cookie that holds no auth state.

### A08:2025 Software or Data Integrity Failures (deserialization, unsigned updates, CI) [2021 A08]
- **Sources:** serialized blobs, cookies, cached/queued messages, update feeds, plugins, CI inputs (PR titles, branch names).
- **Sinks:** deserializers, `eval`-like loaders, auto-update, build scripts, workflow `run:` steps.
- **Grep:** `pickle\.loads?|yaml\.load\(|ObjectInputStream|readObject|unserialize|Marshal\.load|BinaryFormatter|TypeNameHandling|node-serialize|vm\.run|\$\{\{\s*github\.event|pull_request_target|curl .*\|\s*(ba)?sh`.
- **Trail:** attacker-controlled bytes reach an unsafe deserializer or loader, or attacker text is interpolated into a privileged CI step.
- **Traps:** `yaml.safe_load`; JSON; data signed/HMACed before load and key not attacker-known; blob written only by the server and never user-influenced.

### A09:2025 Security Logging and Alerting Failures [2021 A09]
- **Sources:** secrets and PII in request data; attacker-controlled strings.
- **Sinks:** log and audit writers, error reporters.
- **Grep:** `logg(er|ing)\.(info|debug|warn\w*|error)\(.*(password|token|secret|authorization|req\.body|headers)|console\.log\(.*(req|token)`.
- **Trail:** a credential or sensitive field written to logs readable by a wider audience than the data's owner, or newline injection forging log lines that a consumer trusts. A missing audit log is a claim only if the slice's own design requires it.
- **Traps:** value is redacted by a logging filter; logs are operator-only and the data is the operator's own; debug logging off in prod.

### A10:2025 Mishandling of Exceptional Conditions [new in 2025]
- **Sources:** malformed or oversized input that makes a parser throw; timeouts and outages of an auth, policy, or DB dependency the attacker can induce or exploit; resource exhaustion; partial failures.
- **Sinks:** authn/authz or validation steps inside `try`/`catch`, error handlers that return success or fall through to the protected action, swallowed exceptions that skip a check, missing rollback leaving exploitable partial state, error output that leaks internals.
- **Grep:** `catch\s*(\([^)]*\))?\s*\{\s*\}|except(\s+Exception)?\s*:\s*pass|rescue\s*(=>\s*\w+)?\s*$|\.catch\(\(\)\s*=>\s*(\{\}|null|true)\)|catch.*(return true|next\(\))|if err != nil \{\s*(continue|return nil)|fail.?open|default.*allow`.
- **Trail:** the attacker can trigger the exception or dependency failure, the error path (cite the `try`/`catch` and the fall-through) grants access, skips validation or authorization, or leaves exploitable partial state, and the protected action then runs.
- **Traps:** fail-closed handlers (rethrow, deny, return 401/500); a catch that only logs and then returns an error; exception only triggerable by a trusted operator; the swallowed error guards non-security cleanup.

## Part B: OWASP Top 10 for LLM Applications (2026)
Item order from secondary sources; confirm against the official OWASP PDF (genai.owasp.org/resource/owasp-genai-llm-top-10-2026/). IDs are 2026, with the 2025 ID in brackets.
Applies to code that calls a model, builds prompts, runs tools, or stores embeddings. Treat model output as untrusted input.

### LLM01:2026 Prompt Injection [2025: LLM01]
- **Sources:** user text, retrieved documents, web pages, emails, tool results, file contents, other agents' output.
- **Sinks:** prompts that also hold instructions or secrets; tool/function calls; actions taken with the app's privileges.
- **Grep:** `messages\s*=|system.*prompt|\.chat\.completions|anthropic|openai|langchain|llm\.|prompt\s*=\s*f|tools\s*=|function_call|tool_use|mcp`.
- **Trail:** attacker-controlled text lands in model context (cite where), and the model's output then drives a privileged action or exposes data the attacker could not reach directly (see LLM10, LLM03).
- **Traps:** model output only shown as text to the same user who wrote the input; no tools or secrets in context; human approval gate before the action.

### LLM02:2026 Sensitive Information Disclosure [2025: LLM02]
- **Sources:** training/fine-tune data, RAG corpora, conversation history, other users' data in shared context or cache.
- **Sinks:** model responses, logs, shared caches, prompts sent to third-party providers.
- **Grep:** `chat_history|ConversationBufferMemory|messages\.append|semantic_cache|GPTCache|lru_cache` on prompt or response data, plus PII/secret fields entering a prompt.
- **Trail:** data belonging to user/tenant A is placed in context or a cache that a request by B can reach, or secrets/PII are sent to a provider the data policy forbids.
- **Traps:** context scoped per user/tenant; data intentionally public; provider is the approved one.

### LLM03:2026 Excessive Agency [2025: LLM06]
- **Sources:** model-chosen tool calls and arguments.
- **Sinks:** tools with write/delete/send/pay/exec capability; broad credentials given to the agent.
- **Grep:** `tools\s*=|@tool\b|allowed_tools|tool_choice|function_call|auto_approve|--dangerously|bypassPermissions|send_email|run_shell|subprocess`.
- **Trail:** the agent holds a capability broader than its task, uses the app's or user's full privileges, and no confirmation or scope check stands between a model decision (steerable per LLM01) and an irreversible effect.
- **Traps:** per-call human approval; tool enforces the end user's own authorization; read-only tools.

### LLM04:2026 Supply Chain [2025: LLM03]
- **Sources:** model weights, adapters, datasets, plugins/MCP servers, prompt templates, SDKs.
- **Sinks:** model loaders, plugin/tool registries, `trust_remote_code`, unpinned downloads.
- **Grep:** `trust_remote_code|from_pretrained\(|torch\.load|pickle\.load|hf_hub_download|snapshot_download|npx -y|uvx |mcp.*(url|command)|:latest\b|@latest\b`.
- **Trail:** unpinned or unverified artifact (or pickle-based weights) loaded at runtime from a source an attacker can influence, with execution on load.
- **Traps:** pinned to a hash; safetensors; dev notebook only.

### LLM05:2026 Data and Model Poisoning [2025: LLM04]
- **Sources:** user feedback, scraped or user-contributed training/RAG data, shared fine-tune sets.
- **Sinks:** training/fine-tune pipelines, RAG indexes, memory stores that later steer behavior.
- **Grep:** `fine.?tune|\.fit\(|feedback.*(train|dataset)|upsert\(|add_documents|add_texts|memory\.(save|add)`.
- **Trail:** attacker-writable data enters an index or training set with no provenance or review step, and later changes outputs or actions for other users.
- **Traps:** per-user index only; curated or review-gated ingest.

### LLM06:2026 Unbounded Consumption (including denial of wallet) [2025: LLM10]
- **Sources:** user-controlled prompt size, loop counts, concurrency, file sizes; attacker-driven spend (denial of wallet).
- **Sinks:** model calls billed per token, agent loops, embeddings jobs.
- **Grep:** `max_tokens|while True|for .* in .*agent|retry|recursion|n=|batch|limit`.
- **Trail:** an unauthenticated or low-privilege user can trigger unbounded or amplified model calls (no token cap, loop limit, per-user quota), with availability impact or attacker-driven cost (denial of wallet: no per-user or global spend budget).
- **Traps:** gateway or provider quota; hard caps set in a wrapper elsewhere.

### LLM07:2026 Misinformation [2025: LLM09]
- **Sources:** model hallucination; **sinks:** automated decisions or code/package suggestions acted on without verification.
- **Grep:** `pip install|npm install` or auto-applied model output; decisions on model-generated facts.
- **Trail:** unverified model output directly triggers a consequential action (installs a package, approves a transaction, gives medical/legal answers as authoritative). Usually Low/Info unless the trail is concrete.
- **Traps:** human review; output used only as a suggestion.

### LLM08:2026 Hidden Context Exposure [2025: LLM07 System Prompt Leakage]
- **Sources:** user requests that elicit hidden context (direct asks, injection, error messages, debug or verbose modes, API responses that echo internals).
- **Sinks:** anything the user is not meant to see: system prompts, injected retrieval context, tool definitions and schemas, hidden instructions, returned to the client or into model output.
- **Grep:** `system.*prompt|SYSTEM_PROMPT|instructions\s*=` near `api[_-]?key|password|secret|internal|http`; responses that include raw context: `return.*(messages|system_prompt|tool_definitions|context)|include_prompt|debug.*prompt|show_context`.
- **Trail:** a secret or authorization rule lives only in hidden context and is trusted for enforcement, or hidden context holding data the user is not entitled to (other tenants' retrieved chunks, tool schemas with credentials) reaches them (cite the response path). An extractable prompt alone is Info.
- **Traps:** hidden context holds no secrets; the user is entitled to the data; real authorization is enforced in code.

### LLM09:2026 Vector and Embedding Weaknesses [2025: LLM08]
- **Sources:** documents ingested into a vector store; queries.
- **Sinks:** retrieval results placed in context; shared indexes.
- **Grep:** `similarity_search|retriever|vectorstore|embed|namespace|collection|filter=`.
- **Trail:** retrieval has no tenant/ACL filter, so one user's query returns another's chunks, or attacker-uploaded documents are retrieved into other users' context (feeds LLM01).
- **Traps:** per-tenant namespace or metadata filter enforced server-side.

### LLM10:2026 Improper Output Handling [2025: LLM05]
- **Sources:** model output.
- **Sinks:** HTML/markdown render, SQL/shell/code exec, `eval`, URL fetch, file writes, downstream API calls.
- **Grep:** `eval\(|(^|[^.\w])exec\(|innerHTML|dangerouslySetInnerHTML|markdown|subprocess|execute\(|json\.loads\(.*(response|completion)|open\(.*(response|output)`.
- **Trail:** model output (influenced per LLM01) flows to an injection sink (A05:2025) without validation, encoding, or sandboxing. Cite it as a normal injection trail with the model as the intermediate hop.
- **Traps:** output schema-validated and allow-listed; rendered through a sanitizer or in a sandbox.

## Part C: OWASP API Security Top 10 (2023), items beyond Part A
- **API1 BOLA / API5 BFLA** (extends A01): enumerate every route, including secondary ones (bulk, export, GraphQL fields/resolvers, websocket handlers, nested resources, `PUT` beside `GET`). BOLA = object-level ownership missing; BFLA = a role-restricted function (admin, delete, export) reachable by a lower role because the check is on the UI or on one verb only. Grep: `router\.|@(Get|Post|Put|Delete)Mapping|app\.(get|post|put|delete)|resolvers|Query:|Mutation:`. Trap: authorization applied by a router-level or schema-level directive.
- **API3 Broken Object Property Level Authorization:** excessive data exposure (serializer returns whole model, password hash, internal flags) and mass assignment (request body bound straight to the model). Grep: `Object\.assign\(.*req\.body|\.\.\.req\.body|update\(req\.body|\*\*request\.(data|json)|fields\s*=\s*['"]__all__|attr_accessible|toJSON|serialize`. Trail: a sensitive field is both writable by the attacker or returned to them, shown from the serializer/binder. Trap: explicit allow-list or DTO.
- **API6 Unrestricted Access to Sensitive Business Flows:** signup, checkout, voucher, scraping, SMS/email send with no per-actor limit, where automation causes concrete business harm. Claim only with the harm stated (cost, fraud), else it is not a claim. Trap: limits at a gateway.
- **API9 Improper Inventory Management:** old API versions, debug or internal routes, shadow endpoints still mounted without the new version's controls. Grep: `/v1/|/v2/|/internal|/beta|/test|deprecated`. Trail: the old route reaches the same data without the check the new one has.
- **API10 Unsafe Consumption of APIs:** responses from third-party APIs trusted like first-party input (fed to sinks, followed redirects, unvalidated URLs or IDs). Trail: third-party-controlled field reaches an injection (A05:2025) or SSRF (A01:2025) sink. Trap: schema-validated response.
- **API4 Unrestricted Resource Consumption** (extends LLM06:2026): list/search endpoints with no pagination or maximum page size, no body or upload size limits, GraphQL with no depth, complexity, alias, or batching limits, expensive per-request work (export, regex, image processing), unbounded fan-out. Grep: `per_page|pageSize|limit\(|take\(|findAll\(\)|\.all\(\)|depthLimit|maxDepth|complexity|MAX_CONTENT_LENGTH|bodyParser|multer`. Trail: an attacker-controlled size or count reaches an unbounded query, loop, or allocation with concrete availability or cost impact. Trap: limits at a gateway, framework defaults (e.g. body-parser's default size cap), or ORM caps.

## Part D: OWASP Top 10 for Agentic Applications (for 2026, published Dec 2025)
IDs and names come from secondary sources; confirm against genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/. Apply Part D only when the target builds or runs agents (tool calling, multi-agent, memory, autonomous loops); otherwise skip it. Where an item overlaps Part B, use that entry and keep only the agent-specific angle here.

### ASI01 Agent Goal Hijack (see LLM01:2026)
- **Sources/sinks:** instructions inside tool results, documents, emails, calendar items, or peer-agent messages that rewrite the agent's plan or objective; persisted goal, plan, or task state.
- **Grep:** `goal|objective|plan\s*=|task_queue|planner`.
- **Trail:** untrusted content (cite where it enters context) alters a stored goal or plan that the loop then executes autonomously with the agent's tools.
- **Traps:** goal fixed in code and re-injected each step; per-step human approval; content only summarized, never acted on.

### ASI02 Tool Misuse & Exploitation (see LLM03:2026)
- **Sources/sinks:** model-chosen arguments and tool chains; third-party tool descriptions; individually legitimate tools combined (read a secret, then send it).
- **Grep:** `tool_choice|@tool\b|tools\s*=|\*\*tool_input|json\.loads\(.*arguments`.
- **Trail:** a tool executes model-supplied arguments with no schema, allow-list, or scope check at the tool boundary, or a chain of individually allowed tools reaches an unauthorized effect (cite each call).
- **Traps:** the tool validates and authorizes against the end user's identity; read-only tools; allow-listed arguments.

### ASI03 Identity & Privilege Abuse
- **Sources/sinks:** credentials the agent holds or inherits: shared service accounts, passed-through user tokens, long-lived or cached tokens, delegation to sub-agents.
- **Grep:** `service_account|api_key\s*=|token.*(cache|session)|impersonat|on_behalf|delegat`.
- **Trail:** the agent acts with broader or another principal's credentials than the requesting user holds (confused deputy), or a token cached across users or sessions is reused; cite where it is obtained and where it is used.
- **Traps:** per-user, scoped, short-lived tokens; the downstream service re-authorizes the end user.

### ASI04 Agentic Supply Chain Vulnerabilities (see LLM04:2026)
- **Sources/sinks:** tools, MCP servers, agent cards, plugins, and prompt templates discovered or loaded at runtime; tool descriptions that change after approval.
- **Grep:** `npx -y|uvx |mcp.*(url|command|servers)|agent[_-]?card|load_plugin|importlib|:latest\b`.
- **Trail:** a tool or server fetched or updated at runtime from a source an attacker can influence is trusted (description or code) with no pin, signature, or re-approval.
- **Traps:** pinned versions or hashes; allow-listed internal registry; re-approval on change.

### ASI05 Unexpected Code Execution (RCE) (see LLM10:2026, A05:2025)
- **Sources/sinks:** model-written code or commands reaching interpreters, shells, notebooks, `eval`, package installs, or write-then-execute paths.
- **Grep:** `code_interpreter|exec_code|run_python|PythonREPL|subprocess|(^|[^.\w])exec\(|eval\(|pip install`.
- **Trail:** model output (steerable per ASI01) reaches an executor in a host or credential context beyond a sandbox; cite the executor and what the sandbox does not cover.
- **Traps:** isolated container or VM with no credentials and no network; allow-listed commands; execution gated by approval.

### ASI06 Memory & Context Poisoning (see LLM05:2026, LLM09:2026)
- **Sources/sinks:** persistent memory, conversation summaries, and shared stores written from untrusted content, then re-injected as trusted context.
- **Grep:** `memory\.(save|add|update)|summar|long_term|scratchpad|store\.put`.
- **Trail:** untrusted content is written to memory with no provenance, and later sessions or other users' runs read it as trusted instruction or fact; cite the write and the read.
- **Traps:** memory scoped per user and session; writes only from trusted sources; entries stored as data and labeled untrusted.

### ASI07 Insecure Inter-Agent Communication
- **Sources/sinks:** messages between agents over A2A, MCP, queues, HTTP, or shared state; agent identity claims.
- **Grep:** `a2a|agent[_-]?to[_-]?agent|send_message|publish\(|subscribe\(|handoff|from_agent`.
- **Trail:** a message is accepted as coming from a trusted agent with no authentication, integrity, or replay protection, and it drives tool calls or state changes; cite the receiver's trust decision.
- **Traps:** mutual auth or signed messages; in-process calls with no external writer; the receiver treats content as untrusted data.

### ASI08 Cascading Failures
- **Sources/sinks:** one agent's wrong output, error, or retry loop propagating across chained agents, delegation, or fan-out.
- **Grep:** `max_(iterations|steps|turns|depth)|while True|recursion|delegate|spawn|fan.?out|circuit`.
- **Trail:** show propagation across at least two hops and the missing bound (no step limit, breaker, or validation between agents) that turns one fault into a loop, outage, or mass wrong action.
- **Traps:** iteration and depth caps; validation or a human checkpoint between hops; failures isolated per task.

### ASI09 Human-Agent Trust Exploitation
- **Sources/sinks:** agent-written text that a human approver relies on: approval prompts, summaries, rationales, "safe" labels.
- **Grep:** `confirm|approv(e|al)|human_in_the_loop|ask_user|permission_prompt`.
- **Trail:** the approval UI shows agent-authored text in place of the real action and arguments, and attacker-influenced content can shape that text; cite the display path.
- **Traps:** the approval shows the raw command or diff; the action is low impact or reversible; the text is user-authored.

### ASI10 Rogue Agents
- **Sources/sinks:** an agent's ability to change its own permissions, prompt, tools, or config, spawn agents, or persist and schedule itself.
- **Grep:** `create_agent|spawn_agent|add_tool|edit.*(prompt|permission|settings)|schedule|cron`.
- **Trail:** the agent can write its own policy or create broader-scope agents, with no external enforcement, audit, or kill switch; cite the write path and the missing control.
- **Traps:** policy lives outside the agent's write access; privileged actions need out-of-band approval; spawned agents inherit narrower scope.
