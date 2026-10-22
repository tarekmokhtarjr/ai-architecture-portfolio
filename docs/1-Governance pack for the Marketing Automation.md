# Marketing Automation with AI & Agents — Architecture Governance Pack

**Business:** Online coffee e-commerce | **Core platform:** Odoo | **Status:** Proposed MVP  
**Objective:** Increase repeat purchases, email engagement and conversion by recommending relevant coffee products from customer purchase history. An LLM product-advisor chatbot is optional and out of MVP scope.

## 1. Scope and target architecture

**MVP flow:** Odoo remains the system of record for customers, products, orders, consent and email campaigns. A scheduled pipeline exports only the data needed for recommendations to a separate Python service. A recommendation model scores eligible products per customer; a ranking/filtering step removes unsuitable products and stores the top recommendations. Odoo retrieves those results through an authenticated API or scheduled import and uses them in approved email campaigns. The following metrics are to be considered:

- Opens and Clicks
  
- Purchases
  
- Complaints
  
- Unsubscribes (optional)
  
  are measured to evaluate business impact.
  

The team (may) consider an automated E-mail right after a succssefull purchase order in case it doesn't increase latency or affect the platform's responsiveness and performance.

**AI responsibilities:** ML selects/ranks products; Odoo applies campaign rules and sends email; marketing staff approve campaigns. No autonomous agent may send campaigns, change prices or alter customer consent. The **optional** **chatbot** may later use an LLM to understand a customer's stated preferences and explain catalogue choices, but it does not replace or silently override the recommendation model.

## 2. Architecture principles

| ID  | Principle | Business goal / measurable intent |
| --- | --- | --- |
| AP-01 | **Business-owned system of record:** Odoo owns customer, product, order, consent and campaign data. AI services consume governed data and return recommendations; they do not become authoritative data stores. | Preserve reliable commerce operations and avoid conflicting customer/product records. |
| AP-02 | **Privacy and security by design:** Minimize and pseudonymize data sent to AI components; enforce purpose limitation, retention, access control and auditable consent/suppression checks. | Maintain customer trust and comply with applicable privacy law (including GDPR and, where applicable, PDPL). |
| AP-03 | **Measured, replaceable AI:** Start with the simplest model that can be evaluated; version datasets/models and measure recommendation quality and campaign outcomes. Keep the model behind a documented API. | Improve repeat purchase and conversion while controlling cost and avoiding vendor/model lock-in. |
| AP-04 | **Human-controlled marketing:** AI may propose products and draft copy, but campaign eligibility, content approval and sending remain governed by Odoo rules and authorized employees. | Protect brand reputation, prevent inappropriate messaging and retain accountable human oversight. |

## 3. Architecture Decision Records

### ADR-01 — Recommendation execution: separate Python API vs logic embedded in Odoo

**Options:** (A) Put model code inside Odoo; (B) run a separate Python recommendation service; (C) use a third-party recommendation SaaS.  
**Trade-offs:** Embedded logic is initially simple but couples Odoo upgrades/dependencies to ML training. SaaS reduces operations but adds recurring cost, data-sharing and vendor dependency. A separate service adds deployment/monitoring work but isolates ML dependencies and permits independent model lifecycle management.  
**Decision:** **B — separate Python service**, exposed through an authenticated, versioned API. Odoo remains the source of truth. The service receives pseudonymous customer identifiers and the minimum purchase/product features required. If unavailable, campaigns use safe fallback products or are skipped; they do not block checkout.

### ADR-02 — Recommendation data/model and no vector database in MVP

**Options:** (A) manual rules / popular products; (B) collaborative filtering, content based filtering or hybrid (C) embeddings plus vector database; (D) LLM-generated recommendations.  
**Trade-offs:** Rules are cheap and useful for cold-start fallback but weakly personalized. Collaborative filtering learns from order co-occurrence and purchase patterns, is explainable at a business level and does not require an LLM; it needs enough interaction data and has cold-start limitations. Embeddings/vector search add infrastructure and are more useful for semantic product descriptions or a future chatbot. LLM recommendations are less deterministic and harder to constrain/evaluate for ranking.  
**Decision:** **B for MVP; A as fallback. No vector database and no LLM in the recommendation path.** Exclude cancelled/refunded orders as appropriate, use only eligible products, and fall back to bestsellers/category rules for new customers or sparse history. Reassess when catalogue semantics or chatbot retrieval becomes a demonstrated need.
**Note**: the team wants to do R&D to decide the best approach

- collaborative filtering
  
- content based filtering
  
- hybrid
  

### ADR-03 — Recommendation freshness: scheduled batch vs real-time inference

**Options:** (A) score on every email/customer request; (B) scheduled batch generation; (C) hybrid batch plus event-triggered refresh.  
**Trade-offs:** Real-time inference is freshest but increases latency, availability coupling and compute/operational complexity. Batch is predictable, inexpensive and aligns with scheduled email campaigns, but recommendations can become stale. Hybrid adds complexity and should be justified by measured business need.  
**Decision:** **B — batch initially** (e.g. daily, with a configurable schedule). Persist a timestamp, model version and top-N results. Refresh after model retraining or scheduled data ingestion. Use current Odoo eligibility/consent checks at campaign execution time, not only at batch time. Consider hybrid refresh only if stale recommendations materially harm outcomes.

## 4. Architecture Review Board (ARB) checklist

Before approving an architectural change, the review board checks:

- [ ] Security: Is communication between Odoo and the recommendation service secure?
  
- [ ] Privacy (GDPR/PDPL): Is customer data protected? Are marketing consent and unsubscribe requests respected?
  
- [ ] AI quality: Are recommendations relevant, and are the recommended products active and available for sale?
  
- [ ] Cost: Are infrastructure and AI service costs reasonable for the business?
  
- [ ] Human oversight: Can marketing employees review and approve email campaigns before sending?
  
- [ ] Reliability: What happens if the recommendation service is unavailable? Is there a fallback mechanism?
  

## 5. Automated guardrail (CI)

Recommendation validation: Add a simple automated test to the CI pipeline that checks recommendations before deployment.

The test verifies that:

1. No more than three products are recommended per customer.
  
2. Recommendations contain no duplicate products.
  
3. Recommended products exist in Odoo.
  
4. Recommended products are active and available for sale.
  

If any check fails, the CI pipeline fails and prevents deployment.

This helps enforce the principles of AI quality, security and reliability.

## 6. Future Enhancement (Out of MVP Scope)

An optional LLM-powered chatbot could help customers discover suitable coffee products based on their preferences.

It could use the Odoo product catalogue to answer questions and suggest relevant products.

This chatbot is a future enhancement and is not required for the recommendation engine or email marketing automation.
