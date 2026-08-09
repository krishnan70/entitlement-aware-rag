# Fourteen Questions for Vendors

Ask these **in writing, before the demo**. The written answers are more informative than the demo.

The question that does *not* discriminate:

> ~~"Which vector database supports ACLs?"~~

Every vendor passes it, because metadata filtering is table stakes. Passing proves only that a filter syntax exists.

The question that does:

> **"Where is the authoritative authorization decision made, and how is that decision enforced at every retrieval boundary?"**

That forces a named source of truth, a named decision point, and a named enforcement mechanism at each of five boundaries: candidate generation, reranking, context construction, citation rendering, and cache lookup.

---

## The fourteen

1. Can security filters execute **before or during** vector retrieval?
2. How are groups represented?
3. What happens with **thousands of groups per user**?
4. How quickly can ACL changes propagate?
5. Can access be revoked immediately?
6. Is multi-tenant isolation supported — logical or physical?
7. How are hybrid search and filters combined?
8. Can policy decisions be audited?
9. Does caching respect entitlement context?
10. Can the system **fail closed** when authorization infrastructure is unavailable?
11. How are source permissions synchronized?
12. Can document-level permissions propagate safely to chunks?
13. How are citations secured?
14. Can the platform support continuous security evaluation?

---

## The three that separate marketing from engineering

**Q1 — filtered ANN.** Every vendor says yes. The follow-up that discriminates:

> *"Show recall and latency at 1% filter selectivity, on an index of our size."*

Some engines apply the filter as a post-pass over ANN results — post-filtering with a pre-filtering API. Others maintain filtered graph traversal properly. Approximate-nearest-neighbour indexes are built for unfiltered traversal; a highly selective filter can force far more graph exploration to find K results, or a fallback to brute force.

**Q3 — high cardinality.** In a 50,000-person enterprise, a user may resolve to thousands of groups once nested groups expand. That produces a filter with thousands of terms. Ask for the benchmark, not the feature.

**Q10 — fail closed.** Lead with this in regulated environments. When the policy engine is unreachable or times out, what happens? A surprising number of systems default to *allow* to preserve availability, converting an infrastructure blip into an enterprise-wide exposure.

> Test it by experiment, not by asking: take the policy engine down in staging and watch what gets served.

**A platform that fails open under load is a platform that leaks under load. A platform that cannot be continuously tested cannot be continuously trusted.**

---

## Capability categories

An enterprise implementation combines several categories rather than relying on one product.

| Category | Examples | Responsibility |
|---|---|---|
| Identity providers | Microsoft Entra ID, Okta | identity, group membership, authentication, lifecycle |
| Authorization / relationship engines | OpenFGA, SpiceDB | the ReBAC graph, answered in single-digit milliseconds |
| Search and retrieval | Azure AI Search, Elasticsearch, OpenSearch, Pinecone, Weaviate | metadata filtering, tenant isolation, filtered vector search, hybrid search, high-cardinality filters, index update latency, auditability |
| Enterprise knowledge platforms | Glean | permissions-aware access to connected enterprise content |

> Most enterprise RAG architectures have a clear identity layer, a clear retrieval layer, a clear generation layer — and **no authorization layer**. Authorization exists as fragments: a filter expression in the query builder, a mapping in a connector, an if-statement in application code, a field in the index. When it is not a layer, nobody owns it, nobody versions it, and nobody can answer "why did Alice see this document?"

*Product capabilities change. Evaluate current vendor documentation when selecting a production architecture.*
