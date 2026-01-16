# Agent Development Constitution

> **Thesis**: Agent frameworks introduce supply chain risk and opaque abstractions. LLMs are capable enough to orchestrate modular, hand-rolled agent systems—if built on first principles that enhance modularity and composability.

---

## Preamble

This constitution defines principles for building **tool-use agents**—systems guided by a system prompt that use tools to complete tasks. The LLM is the reasoning engine; tools are modular, composable units of capability.

### Foundational Assumptions

**Pattern**: We build tool-use agents. A system prompt defines purpose and constraints. The LLM decides which tools to call. Tools execute and return results. No separate "planner" or "orchestrator" layer—the LLM handles reasoning.

**Scope**: Task-oriented agents that execute workflows or automate processes. Not personal assistants. Long-term user memory is typically a liability. Prefer elegant solutions that leverage inherent state (Teams threads, email chains, document versioning, sharepoint file fields, order attributes) over building memory systems. 

**Modularity Goal**: Build so you can extend, compose, or swap components without rewriting the agent. Each tool is a clean, testable unit. Agents can be tools for other agents.

**Infrastructure**: Azure-native primitives managed through resource groups, RBAC, and unified billing. GitHub + GitHub Actions for source control and deployment.

---

## Part I: Agent Design

### Article 1: System Prompt

**1.1** Every agent **SHALL** have a system prompt that defines:
- Purpose (what it does)
- Constraints (what it cannot/should not do)
- Available tools (what it can use)
- Behavioral guidance (tone, escalation triggers, edge cases)

**1.2** System prompts **SHALL** be versioned and stored in source control.

**1.3** Constraints **SHALL** be enforced in code, not just stated in prompts. Prompts can be jailbroken; code cannot.

---

### Article 2: Tool Design

**2.1** Each tool **SHALL** have:
- A single, clear purpose
- A typed input schema (Pydantic, Zod, JSON Schema)
- A typed output schema
- Descriptive documentation (the LLM reads this to decide when to use it)

**2.2** Tools **SHALL** be stateless when possible. If state is needed, pass it explicitly.

**2.3** Tools **SHALL** be independently testable. If you can't unit test a tool in isolation, it's too coupled.

**2.4** Tools **SHALL** handle their own errors gracefully and return structured error responses, not exceptions.

```python
# Good tool design
class SearchDocsInput(BaseModel):
    query: str
    max_results: int = 5

class SearchDocsOutput(BaseModel):
    results: list[Document]
    error: str | None = None

def search_docs(input: SearchDocsInput) -> SearchDocsOutput:
    try:
        results = vector_store.search(input.query, limit=input.max_results)
        return SearchDocsOutput(results=results)
    except Exception as e:
        return SearchDocsOutput(results=[], error=str(e))
```

---

### Article 3: Modularity & Composability

**3.1** Agents **CAN** be tools for other agents. Design with this in mind.

**3.2** Tool interfaces **SHALL** be stable. Changing a tool's schema is a breaking change.

**3.3** Prefer many small tools over few large tools. Small tools are easier to test, reuse, and compose.

**3.4** Avoid implicit dependencies between tools. If Tool B needs output from Tool A, the LLM should orchestrate that, not hidden code.

---

## Part II: Security

### Article 4: Input Validation & Tool Gateway

**4.1** All tool inputs **SHALL** be validated against their schemas before execution.

**4.2** All tool calls **SHALL** be logged with: tool name, input parameters, output summary, timestamp, latency.

**4.3** Tools that access external systems **SHALL** use least-privilege credentials.

**4.4** Tools that execute code or access filesystems **SHALL** be sandboxed.

#### Don't Reinvent

| Need | Use |
|------|-----|
| Schema validation | Pydantic (Python), Zod (TypeScript) |
| Rate limiting | Azure APIM |
| Code sandboxing | Azure Container Apps, E2B |
| Observability | Azure Monitor + OpenTelemetry |
| Vector storage | Azure AI Search |

---

### Article 5: Prompt Security

**5.1** User inputs, retrieved documents, and external API responses **SHALL** be treated as untrusted.

**5.2** Use structural boundaries to separate trusted and untrusted content:

```xml
<system>
You are an agent that helps with document analysis.
You may only access documents in the user's permitted collections.
</system>

<user_input>
<!-- UNTRUSTED -->
{user_message}
</user_input>

<retrieved_context>
<!-- UNTRUSTED - from external sources -->
{rag_content}
</retrieved_context>
```

**5.3** Constraints stated in prompts **SHALL** also be enforced in tool code. Defense in depth.

---

### Article 6: State & Memory

**6.1** Default to session-scoped or stateless. If you're persisting state across sessions, justify it.

**6.2** Leverage inherent state when available:
- Teams thread = conversation memory
- Email chain = context
- Document with version history = state over time

**6.3** If you use RAG, retrieved content **SHALL** include provenance metadata (source, timestamp) and be treated as untrusted.

**6.4** Memory systems **SHALL** use Azure-managed storage (Cosmos DB, AI Search) with encryption and RBAC.

---

## Part III: Operations

### Article 7: Observability

**7.1** Log tool calls, completions, and errors. Enough to reconstruct what happened.

**7.2** Use Azure Monitor + Application Insights for non-dev accessible dashboards.

**7.3** Alert on errors and anomalies. Route to Teams if appropriate.

**7.4** Don't log full prompts/responses in production (cost, privacy). Log structured summaries.

---

### Article 8: Robustness

**8.1** Handle tool failures gracefully. Return structured errors, don't crash.

**8.2** Set timeouts on external calls. Fail fast.

**8.3** If a tool is flaky, consider: retry with backoff, fallback behavior, or surfacing the error to the user.

**8.4** Set resource budgets where appropriate (max tokens, max tool calls per request).

---

### Article 9: Testing

**9.1** Unit test each tool independently.

**9.2** Test with malformed inputs (schema validation should catch these).

**9.3** Test with adversarial inputs (prompt injection attempts in user messages).

**9.4** Integration test the full agent flow for critical paths.

---

## Part IV: Deployment

### Article 10: Source Control & CI/CD

**10.1** All agent code, system prompts, and tool definitions **SHALL** be in GitHub.

**10.2** Deploy via GitHub Actions. No manual deployments to production.

**10.3** Version system prompts. Prompt changes are code changes.

---

## Part V: Anti-Patterns

### ❌ Don't Do This

1. **Implicit orchestration** — Hidden code that decides tool order. Let the LLM reason.
2. **Mega-tools** — One tool that does 10 things. Break it up.
3. **Stateful tools** — Tools that remember previous calls. Pass state explicitly.
4. **Prompts as the only guardrail** — Prompts can be bypassed. Enforce in code.
5. **Rolling your own primitives** — Don't build sandboxing, rate limiting, or crypto.
6. **Logging everything** — Full prompts in prod = cost explosion + privacy risk.

### ✅ Do This

1. **Small, focused tools** — Easy to test, reuse, compose.
2. **Typed schemas** — Pydantic/Zod for inputs and outputs.
3. **Defense in depth** — Prompt constraints + code enforcement.
4. **Leverage inherent state** — Use the platform's memory (Teams, email, docs).
5. **Azure-managed services** — Resource groups, RBAC, unified billing.

---

## Part VI: Quick Reference

### Before Building an Agent

1. What's its purpose? (One sentence)
2. What are its constraints? (What it cannot do)
3. What tools does it need?
4. How will you debug it when something goes wrong?

### Tool Checklist

- [ ] Single purpose
- [ ] Typed input/output schemas
- [ ] Good description for LLM
- [ ] Handles errors gracefully
- [ ] Independently testable
- [ ] Logged

### Deployment Checklist

- [ ] Code in GitHub
- [ ] System prompts versioned
- [ ] Deploys via GitHub Actions
- [ ] Observability configured
- [ ] Errors route to appropriate channel

---

## Appendix A: Attack Vectors

| Attack | Description | Defense |
|--------|-------------|---------|
| **Prompt Injection** | Malicious instructions in user input | Structural boundaries, code enforcement |
| **Tool Misuse** | Tricking agent into calling tools inappropriately | Constraints in prompt + validation in code |
| **Data Exfiltration** | Agent leaks data via tool calls | Least privilege, output validation |
| **Resource Exhaustion** | Infinite loops, excessive API calls | Timeouts, budgets |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-01-16 | Intial construction, tool-use agent pattern; removed planner/DAG emphasis |

---

*This is a living document. Update as patterns evolve.*
