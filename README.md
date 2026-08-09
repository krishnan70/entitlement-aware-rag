# Entitlement-Aware RAG

**Secure retrieval for enterprise AI — course material, checklists, and evaluation artifacts.**

Enterprise RAG cannot simply retrieve the most semantically relevant information. It must retrieve the best information the current user is **authorized to access**.

```
SecureRetrieval(q, u) = TopK Relevance(q, d)   subject to   Authorized(u, d) = 1
```

The central question this material exists to answer:

> **How do you prove your RAG isn't leaking information?**

---

## What's here

| Path | Contents |
|---|---|
| [`slides/`](slides/) | The full 3-hour course deck (88 slides, PDF + PPTX) and a companion quiz deck (13 questions with answers) |
| [`checklists/`](checklists/) | Production readiness checklist — 30 checks across identity, ingestion, retrieval, generation, cache, evaluation, monitoring |
| [`evaluation/`](evaluation/) | Security metric definitions, the entitlement-aware eval record schema, example eval cases, and a red-team attack catalog |
| [`reference/`](reference/) | Normalized security descriptor, security telemetry event schema, and 14 questions to ask vendors |

Start with [`slides/entitlement-aware-rag-course.pdf`](slides/entitlement-aware-rag-course.pdf). The speaker notes in the PPTX carry the full narration — the PDF is the visual layer only.

---

## The problem in one example

An employee asks your enterprise assistant:

> *"What are the company's plans for Project Phoenix?"*

The best semantic match is an executive acquisition memo. Your retriever did its job perfectly.

**That is precisely the problem.** Relevance does not imply authorization, and a perfect retriever over a mixed corpus is an efficient leak. The failure mode is inverted from what most teams expect: the better your embeddings, the more precisely you surface the most sensitive document.

---

## Six places a RAG system leaks

Retrieval is only the first surface.

1. **Unauthorized retrieval** — the chunk enters the candidate set
2. **Unauthorized LLM context** — the chunk crosses into the prompt
3. **Generated-answer leakage** — the fact appears in the response
4. **Citation / metadata leakage** — the title, filename or URL is exposed
5. **Cache leakage** — one user's privileged answer is served to another
6. **Logs / traces / observability leakage** — the secret lands in your SIEM

Surfaces 4, 5 and 6 are the ones teams most often have no control for at all. Even the *existence* of a confidential document can be the secret:

> *"I found `Acquisition-of-Company-X.pdf`, but you don't have access to it."*

Correct denial. Material disclosure.

---

## Three principles

**Authorization is a hard constraint, not a ranking signal.** Do not model it as `Score = Relevance + 0.2 · Permission` — that is exploitable (a good enough query outranks the penalty), unauditable (a weighted sum is not an answer to "why did Alice see this?"), and untestable (there is no invariant to assert in CI).

```
A(u, d) = 0   ⇒   d is not in the candidate set
```

**Authorization is not a prompt.** A system instruction saying "do not reveal confidential information" is a request to a stochastic system, not an access control. Enforce it in deterministic infrastructure outside the LLM.

**Security metrics are release gates, not dashboard entries.** Do not average a leak metric with retrieval relevance. A system with excellent NDCG and occasional confidential-data leakage is not "mostly good."

```
URR > 0            ⇒  FAIL
UCE > 0            ⇒  FAIL
ALR > 0            ⇒  FAIL
CrossUserLeak > 0  ⇒  FAIL
```

---

## Evaluation must include identity

A conventional RAG eval record cannot detect an authorization failure, no matter how many cases you write — it has no user, so it cannot express "Alice may see this and Bob may not."

```
EvalCase = (User, Query, Corpus, Entitlements, ExpectedBehavior)
```

See [`evaluation/eval-record.schema.json`](evaluation/eval-record.schema.json) and [`evaluation/example-eval-cases.json`](evaluation/example-eval-cases.json). The key design point: relevance and authorization are **separate labels**, and the interesting cases are exactly where they disagree.

---

## Proving it — seven layers

There is no useful proof based only on an LLM saying "I cannot access that."

| Layer | Evidence |
|---|---|
| 1 · Policy correctness | `Authorized(u,d)` matches source-of-truth permissions |
| 2 · Retrieval correctness | `Retrieved(u,q,d) ⇒ Authorized(u,d)` — the key invariant |
| 3 · Context correctness | `InContext(u,q,d) ⇒ Authorized(u,d)` |
| 4 · Output testing | red-team for facts, identifiers, citations, metadata, inferred secrets |
| 5 · Dynamic authorization | revocation propagates within the defined SLA |
| 6 · Cache isolation | one user's privileged result cannot be served to another |
| 7 · Continuous monitoring | audit telemetry and canaries detect regressions after deployment |

The enterprise claim is not *"our prompt tells the model not to leak."* It is:

> **"We can trace every piece of retrieved context to an authorization decision, continuously test the authorization invariants, and alert when those invariants are violated."**

---

## Course outline

| Module | Topic |
|---|---|
| 1 | The enterprise entitlement-RAG problem |
| 2 | Identity, ACL, RBAC, ABAC, ReBAC |
| 3 | Mathematics of secure retrieval — pre-filter, post-filter, authorized recall |
| 4 | Enterprise reference architecture — freshness, revocation, cache security |
| 5 | Evaluation, metrics, red teaming, leak detection |
| 6 | Enterprise products and implementation patterns |
| 7 | Architecture and red-team design lab |
| 8 | Production readiness checklist |

Audience: AI/RAG engineers, architects, platform and security engineers, technical leaders.
Prerequisites: basic RAG, embeddings and vector search, metadata filtering, basic IAM concepts.

---

## Using this material

The slides are built in the **Assertion–Evidence** style (Michael Alley, Penn State): every headline is a full-sentence claim, every slide body is one piece of visual evidence, and the narration lives in the speaker notes. If you are presenting, open the PPTX and read the notes — they are written to be delivered.

The deck is deliberately over-built at 88 slides so it can be cut to fit your timeline. If you need a shorter version, the module dividers, the access-model comparison, and one of the two adversarial-attack slides are the easiest cuts. Protect the hard-constraint slide, the eval record, identity differential testing, the seven layers, and the release gates.

---

## License

Content is released under [CC BY 4.0](LICENSE). Use it, adapt it, teach it — attribution appreciated.

---

## Contributing

Corrections, additional red-team attack classes, and vendor-capability data points are welcome. Open an issue or a PR.

If you have run these checks against a real system and found a failure mode not listed here, that is the most valuable contribution possible — open an issue describing the surface and the detection.
