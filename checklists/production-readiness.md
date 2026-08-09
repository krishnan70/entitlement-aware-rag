# Production Readiness Checklist — Entitlement-Aware RAG

Thirty checks across seven categories. Before deployment, every one should be true — and **provable**.

This is not a compliance exercise. If an item is untrue, one of the known failure modes is live in your system right now.

---

## Identity

- [ ] Every request has an authenticated identity.
- [ ] Group / role / relationship claims are trustworthy.
- [ ] Tenant context is explicit.

> **Most commonly false:** group claims. Identity providers truncate group claims past a threshold (Entra ID does this). A user in more groups than the limit arrives with an incomplete claim set — silently under-privileged if your filter is allow-based, and far worse if any code path treats a missing claim as unrestricted. **Test with your most group-heavy user, not a clean test account.**

## Ingestion

- [ ] Source permissions are ingested.
- [ ] ACLs survive chunking.
- [ ] Permission deletions and revocations propagate.
- [ ] Entitlement synchronization lag is measured.

> **Most commonly false:** "ACLs survive chunking" — usually assumed, rarely tested. Pick ten chunks at random from your index and ask: *if this chunk were retrieved on its own, could the system say who is allowed to see it?*
>
> **Also commonly false:** entitlement lag is not measured at all — which means it is unbounded, and an unbounded staleness window is an unbounded leak window.
>
> **Derived artifacts inherit permissions too:** summaries, embeddings of summaries, extracted entities, knowledge-graph nodes, cached rewrites. A summary of a confidential document is confidential.
>
> **Re-index hazard:** changing chunk size rebuilds the corpus, which is a fresh opportunity to lose every descriptor.

## Retrieval

- [ ] Authorization is a hard constraint.
- [ ] Unauthorized resources do not enter LLM context.
- [ ] Authorization failures fail closed.
- [ ] Hybrid retrieval respects authorization.
- [ ] Reranking cannot reintroduce unauthorized content.

> **Verify fail-closed by experiment, not by assertion.** Take the policy engine down in staging and watch what the system serves. Three outcomes are possible: serve unfiltered (catastrophic), serve nothing (correct but blunt), serve from a bounded-age cached entitlement set (usually the right engineering answer). Teams are frequently surprised by what they find.
>
> **Reranking and fusion are reintroduction hazards.** A reranker that re-queries by document id, an RRF step that unions a filtered dense list with an unfiltered lexical list, a "related documents" expansion — each is a place an excluded document walks back in.

## Generation

- [ ] Context is revalidated before LLM invocation.
- [ ] Citations are entitlement-aware.
- [ ] Restricted metadata is not exposed.
- [ ] Tool calls inherit appropriate user authorization.

> **Citations are assembled by plumbing, not by the model** — application code walks chunk ids and looks up display metadata, frequently without consulting the policy engine. An answer can be perfectly clean while the source list names `Acquisition-of-Company-X.pdf`.
>
> **Tool calls:** an agent running under a broad service account with the user's identity passed as *context* rather than as an enforced credential is a privilege-escalation path. Effective permissions should be the **intersection** of what the user may do and what the task requires — never the union of what the agent's tools can reach.

## Cache

- [ ] Cache keys / security design incorporate entitlement context.
- [ ] Permission changes invalidate or safely isolate cached results.
- [ ] Cross-user leakage tests exist.

> A cache keyed on `hash(question)` bypasses your entire authorization architecture: a cache hit means no retrieval runs, so no filter runs, so no gate is consulted.
>
> Safer: `hash(normalized_query + tenant + entitlement_context + policy_version)` — using a stable, canonical entitlement fingerprint, not a raw group list in arbitrary order.
>
> **Isolation is not invalidation.** A user's entitlements can shrink after an entry was created for them.
>
> **Test with a paraphrase.** The point of a semantic cache is that near-miss questions hit the same entry; a test using the identical string will pass against a cache that is leaking freely.

## Evaluation

- [ ] Positive authorization tests exist.
- [ ] Negative authorization tests exist.
- [ ] Identity differential tests exist.
- [ ] Revocation tests exist.
- [ ] Cross-tenant tests exist.
- [ ] Adversarial tests exist.
- [ ] Canary tests exist.

> This column doubles as a work plan for the eval suite. See [`../evaluation/`](../evaluation/).
>
> **Both directions are required.** An excessively restrictive system has zero leaks and zero value. Report leak metrics alongside utility metrics, or the programme drifts toward over-blocking.

## Monitoring

- [ ] Retrieval decisions are auditable.
- [ ] Policy versions are traceable.
- [ ] Unauthorized context exposure generates alerts.
- [ ] ACL synchronization lag is monitored.
- [ ] Canary leakage is monitored.
- [ ] Security logs do not create a secondary data leak.

> **Policy version traceability** is trivially cheap to add at the start and painful to retrofit. Without it, a decision made under `v41` cannot be explained after you ship `v42`.
>
> **The last item is the trap.** The security telemetry you build to prove you are not leaking is itself a corpus of sensitive metadata — who asked what, about which restricted documents, when. It needs its own access controls, retention policy and review. Log identifiers, decisions and policy versions; **not content**.

---

## Alerts worth having

Page someone:

- unauthorized candidate entering context
- policy-engine errors or fail-open behaviour
- cross-tenant access attempts
- canary-token detection

Investigate today:

- unusually high denied-result ratios
- ACL synchronization lag exceeding SLA
- permission-source outages
- cache authorization mismatch
- restricted citation exposure
- abnormal access patterns
- unexpected authorization-policy version

> Alert on the conditions that **precede** a leak, not only on the leak itself. By the time you detect exposure, the content has already reached a person.
