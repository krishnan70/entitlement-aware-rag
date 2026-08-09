# Security Metrics for Entitlement-Aware RAG

Four security metrics, two error rates, two quality metrics, two operational SLAs. **Never averaged into a single score.**

---

## The four security metrics

### Unauthorized Retrieval Rate — URR

```
URR = unauthorized chunks retrieved / total chunks retrieved
```

**Target: 0.**

Measured at the **retriever output, before any post-filter runs** — otherwise you are measuring your own cleanup, not your exposure.

Interpretation depends on architecture. Under pure pre-filtering, any non-zero URR is a filter defect. Under post-filtering, non-zero URR is expected and is a *risk-surface* measure, not a leak — but it means unauthorized content existed in your process: in memory, in traces, possibly in a candidate cache.

### Unauthorized Context Exposure — UCE

```
UCE = unauthorized chunks entering the LLM / total context chunks
```

**Target: 0.**

This distinguishes a retrieval anomaly from an actual security-boundary violation. Requires an `entered_llm_context` boolean per chunk in your telemetry.

**Reading the pair:**

| Signature | Meaning |
|---|---|
| URR > 0, UCE = 0 | Untidy but benign — the retriever reached, the gate held. No incident. |
| URR > 0, UCE > 0 | Gate failure and a **genuine incident** — content reached the model, therefore the inference provider, therefore your request logs and traces. |
| URR = 0, UCE > 0 | **Architectural bypass.** Content entered the prompt by a path that skipped retrieval: a semantic cache hit, session memory, an agent scratchpad, or hard-coded context. Investigate urgently. |

### Answer Leakage Rate — ALR

```
ALR = answers containing unauthorized facts / restricted test queries
```

**Target: 0.**

`UCE = 0` does **not** imply `ALR = 0`. Three routes bypass the context boundary:

- **Parametric memory** — the model knows something about your organization from pretraining or a prior fine-tune
- **Transitive quotation** — an unrestricted document quotes or summarizes a restricted one, and your ACLs never modelled the dependency
- **Inference** — several permitted fragments compose into a restricted conclusion

Measure at **claim level** — decompose the answer into atomic claims and test each against the restricted fact set. Keyword matching misses paraphrase and misses inference entirely.

### Citation Leakage Rate

Measure whether generated citations expose unauthorized document titles, URLs, filenames, repository names, project names, snippets, or metadata.

Citations are usually assembled from index metadata **after** generation, by code that never consulted the policy engine. Treat citation validation as a distinct pipeline stage with its own tests.

**Distinguish content denial from resource-discovery denial.** A system that denies content while confirming existence is still leaking.

---

## The authorization confusion matrix

|  | Authorized resource | Unauthorized resource |
|---|---|---|
| **Retrieved** | True Allow | **False Allow — security failure** |
| **Blocked** | **False Deny — utility failure** | True Deny |

```
FAR = false allows  / unauthorized access attempts      target: 0
FDR = false denies  / authorized access attempts        target: workload-specific
```

The diagonals are **not symmetric**. A false deny frustrates a user, who files a ticket. A false allow can end a career, breach a contract or trigger a regulatory disclosure — and **nobody files a ticket saying "I received a document I should not have."** That invisibility is why tests, canaries and telemetry are the only detection mechanism.

Track both. A programme reporting only FAR drifts toward over-blocking; one reporting only FDR drifts toward leaking.

---

## Retrieval quality — over the authorized corpus

```
AuthorizedRecall@K    = relevant authorized documents retrieved / relevant authorized documents available
AuthorizedPrecision@K = relevant authorized documents retrieved / total documents retrieved
```

The denominator of authorized recall is **not** all relevant documents in the corpus — it is all relevant documents *this user was allowed to see*. A document the user cannot access is not a miss; it is correctly absent.

> Use the wrong denominator and you incentivize weakening the filter, because that becomes the only way to move the number.

Report at the K you actually ship, and **broken down by persona** — aggregate numbers are dominated by broad-entitlement users and hide the narrow-entitlement majority.

---

## Operational security

```
REL           = T_effective_denial - T_permission_revoked
EntitlementLag = T_index_updated   - T_permission_changed
```

Measure REL to **effective denial**, not to index update — the gap between them is usually a cache.

Worked example:

```
Permission revoked:       10:00:00
Index ACL updated:        10:02:10
Access actually blocked:  10:02:15     →   REL = 135 s
```

Both belong on the scorecard as SLA-defined numbers, not targets of zero — zero is unachievable in a distributed system, and claiming it produces an unfalsifiable statement.

---

## The scorecard

| Metric | Goal |
|---|---|
| Unauthorized Retrieval Rate | **0%** |
| Unauthorized Context Exposure | **0%** |
| Answer Leakage Rate | **0%** |
| False Allow Rate | **0%** |
| Cross-User Cache Leakage | **0** |
| Canary Leakage | **0** |
| Authorized Recall@10 | e.g. > 95% |
| Authorized Precision@10 | e.g. > 95% |
| False Deny Rate | workload-specific |
| Revocation Enforcement Latency | SLA-defined |
| ACL Sync Lag P95 | SLA-defined |
| Authorization Check P95 | platform SLA |

**Do not average security failures with retrieval-quality metrics into a single score.** Two panels, always.

## Release gates

```
URR > 0            ⇒  FAIL
UCE > 0            ⇒  FAIL
ALR > 0            ⇒  FAIL
CrossUserLeak > 0  ⇒  FAIL
```

Boolean, not threshold — a threshold invites negotiation, and negotiation under launch pressure moves in one direction. Exposure is an equality constraint at zero: a condition you satisfy, not a quantity you trade.

Only after the security gates pass should teams optimize relevance, latency and cost.
