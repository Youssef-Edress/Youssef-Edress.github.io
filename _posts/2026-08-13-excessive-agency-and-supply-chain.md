---

title: "Hacking LLMs, Part 4: When the Model Can Act"

date: 2026-08-13

categories: [AI Hacking]

tags: [LLM Pentest]

--- 

# Blog 4 — Excessive Agency & Supply Chain Attacks

> *Part 4 of a 5-part series on testing LLM-powered applications. Covers OWASP **LLM06** (Excessive Agency) and **LLM03** (Supply Chain Vulnerabilities).*

Blog 1 ended on a load-bearing fact: **tool calls are model-requested but code-executed — defense lives in the code.** This post is that fact turned into an attack surface. When you give a model tools, its failure mode stops being "says something wrong" and becomes "*does* something wrong to everything those tools can reach."

---

# LLM06 — Excessive Agency

Give a model the ability to `read_file`, `fetch_url`, `run_query`, `send_email`, or `execute_shell`, and its failure mode becomes **the blast radius of whatever those tools can touch** — filesystems, internal networks, databases, customer data, outbound traffic.

## Why tools are the dominant productization

Every serious LLM product shipping today is an **agent**, not a chatbot. The industry moved from *"LLM as text generator"* to *"LLM as decision-maker plus tool user,"* and that shift expands the attack surface in a way chat-focused threat models miss entirely.

The core problem: the LLM went from a chatbot with no consequential decisions to an entity that can *act.* And if the model is compromised (via any injection from Blog 2), **the attacker inherits all of the agent's access.**

> So the goal of defense is **not** to make the agent refuse more. It's to make sure the agent *cannot* do anything the authenticated user couldn't do directly.

## The three attack shapes

### 1. Parameter injection

The attacker controls arguments the agent passes to a tool. If the tool validates arguments only by the *agent's natural-language rule* (e.g., *"don't read files under /secrets/"*), the attacker crafts arguments that bypass it.

The archetype is **path traversal**: a `read_file(path)` tool whose authorization does a string-prefix match against `/secrets/` will happily read `/home/user/../secrets/flag.txt` — because the path is checked as a *string*, not as a *resolved filesystem location.*

The same pattern generalizes to every classic web attack, now aimed at tool arguments:

- **SSRF** — a `fetch_url` with no allowlist reaches `http://169.254.169.254/latest/meta-data/` (AWS metadata), `http://localhost:6379/` (internal Redis), `file:///etc/passwd`, or any internal hostname.
- **SQL injection via generated queries** — `run_query(sql)` that lets the model compose SQL is web-app SQLi, except the untrusted input is the *entire model output.*
- **Shell injection** — `execute_shell` tools are almost never safe. Attackers inject `;`, `&&`, `|`, backticks, subshells.
- **Encoding / traversal tricks** — URL-encoded, null-byte, and Unicode-homoglyph traversal. Twenty years of WAF-era web attacks now apply to tool arguments.

### 2. Authorization bypass (including confused-deputy)

The agent calls a tool with arguments that are *technically valid* but reach resources the authenticated user shouldn't see.

A `search_logs(tenant_id, query)` tool that accepts `tenant_id` *as a parameter* lets the attacker pass **any** tenant ID. The correct design derives `tenant_id` **server-side from the authenticated session** and *ignores* whatever the model put in the argument.

Variants:

- **Cross-tenant reads/writes** — multi-tenant SaaS where the tool accepts customer IDs from the agent's reasoning. Attacker passes another tenant's ID.
- **Privilege escalation via the agent** — the agent runs with broader permissions than the user. *"Help me check my billing settings"* reaches an admin-only page because the agent's service account has admin rights.
- **Scope creep** — a read-scoped agent that also has a write tool "just in case." Attackers find the write tool.
- **Confused deputy** — the classic pattern, updated for agents: the agent has access the user doesn't, and the attacker just *asks* the agent to use it. The framing is innocuous — *"look up my account"* (without verifying which account maps to the session), *"pull context from the `acme-auth` repo"* (that the engineer shouldn't see), *"help me understand this error"* (returning log fields they shouldn't get). The user's request looks benign; the agent's execution is over-privileged. **The attacker didn't have to craft anything suspicious — they just asked.**

> Cross-tenant vs. confused deputy is often more academic than operational. Both are *"the agent did something for the user that the user was not themselves authorized to do."*

### 3. Tool-chain attacks

The canonical, high-impact chain:

- **Indirect prompt injection** delivers the hostile instruction via content the agent retrieves (a support ticket, a wiki page, a web fetch).
- **A read-side tool** is invoked with parameters drawn from the injection — often `search_logs`, `lookup_account`, `read_file`, or a RAG fetch.
- **A write-/output-side tool** exfiltrates: `send_email` to an attacker address, `fetch_url` to an attacker server with data in the query string, `create_ticket` into a public tracker, or markdown output that renders as an image pointing at an attacker URL.

This is where the whole series converges: injection (Blog 2) + retrieval/vector abuse (Blog 3) + over-privileged tools = a full read-then-exfiltrate exploit.

## Tool discovery is the first step

Before abusing tools, enumerate them:

- **Direct enumeration** — *"What tools do you have access to?"* Many agents happily list them.
- **Indirect enumeration** — *"If I needed to check a log file, what could you do?"* The agent describes the tool without calling it.
- **Error fingerprinting** — request a call that fails with a detailed error (`read_file('/nonexistent')`); errors leak which tools exist and their names.
- **System prompt extraction** — the fastest route (Blog 2). Every tool-enabled agent embeds the tool list in its prompt; extract the prompt, read the tools directly.

## Why "just tell the agent not to" fails

Developers reach first for natural-language restrictions: *"Never read files under /secrets/,"* *"Don't call send_email with external addresses,"* *"Only access the current user's data."*

These are **suggestions the model weighs against the user's request. They are not boundaries.** Under any sufficiently determined attack — direct injection, indirect injection via retrieved content, or multi-turn social engineering — the model complies with the attacker *often enough to matter.* The compliance fraction varies by model and phrasing; **it is never zero.**

> **The only boundary that matters is the one enforced in the tool's code.** If the code allows the call, it succeeds regardless of what the prompt said. If the code rejects it, the prompt doesn't matter.

## Real incidents

- **DuneSlide** — two critical RCE vulnerabilities in an AI tooling path.
- **Vanna.AI (CVE-2024-5565)** — prompt injection led to code execution through a `run_query`-style surface.
- **MCP tool-poisoning** — Invariant Labs showed how a malicious MCP tool description can hijack an agent (a supply-chain + tool-abuse crossover — see below).

## LLM06 defenses

- **Enforce authorization in tool code, not in the prompt.** Derive identity/scope server-side; ignore model-supplied IDs.
- **Least privilege per tool.** Read-scoped agents get no write tools. Scope service accounts to *exactly* what's needed — never "admin, just in case."
- **Validate arguments as data, not language.** Resolve paths before checking them; allowlist URLs/hosts; parameterize SQL; never pass model output to a shell.
- **Separate read and write capabilities**, and put human-in-the-loop confirmation on irreversible or outbound actions (send, delete, transfer).
- **Constrain output rendering** — allowlist image/link origins so markdown-image exfiltration can't fire.

---

# LLM03 — Supply Chain Vulnerabilities

Picture the LLM stack as a pyramid: base model, fine-tunes, libraries, plugins, datasets, model hubs, serving infra. **Supply chain attacks target one block at the base — and the whole structure inherits the compromise.**

You can pick up a vulnerability through:

- a compromised or malicious **library / dependency**,
- a **third-party plugin** or MCP tool,
- a **pre-trained model or dataset** pulled from a public hub (this overlaps directly with Model Poisoning in Blog 3),
- a tampered **model artifact** or config (recall the Hugging Face Transformers config-injection RCE).

## Impact: cascade risk

The defining property is **cascade risk**: once one part of the pyramid is compromised, everything built on top of it is affected. A backdoored base model taints every fine-tune; a malicious dependency taints every service that imports it; a poisoned dataset taints every model trained on it.

Because it lands *upstream*, supply-chain compromise shares Model Poisoning's nastiest trait — **it's already baked in by the time you're serving traffic**, so runtime defenses don't help.

## Where it crosses the other classes

- **Model Poisoning (LLM04)** — pulling a poisoned checkpoint from a hub is *both* a supply-chain and a poisoning attack; the model-artifact entry point is the cheapest poisoning path precisely because it's a supply-chain move.
- **Excessive Agency (LLM06)** — a malicious MCP tool or plugin is a supply-chain artifact that hands the attacker tool-level control (MCP tool-poisoning).

## LLM03 defenses

- **Vet and pin dependencies** — lockfiles, version pinning, SBOMs; scan for known CVEs.
- **Verify model provenance** — prefer signed artifacts and known-good sources; scan model configs before loading (a download can be RCE).
- **Vet plugins and MCP tools** — treat third-party tool descriptions as untrusted; review what capabilities you're importing.
- **Isolate and least-privilege the serving environment** — so a compromised component's blast radius is contained.
- **Monitor upstream** — track advisories for every model, dataset, library, and plugin in your stack.

---

**Next up — Blog 5: Misinformation & Unbounded Consumption.** The two failure modes that don't need an attacker at all — the model is confidently wrong, or it's asked to do too much work — but that turn into real incidents and denial-of-service the moment someone points them on purpose.
