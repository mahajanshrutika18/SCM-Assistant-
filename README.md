# SCM Assistant — Supply Chain RAG Chatbot (Flowise)

A Retrieval-Augmented Generation (RAG) chatbot built on **Flowise Cloud** that answers questions about the BQBYTE Technologies supplier network, using the provided supplier performance dataset (2,000 purchase orders, 116 suppliers) and the Supplier Governance & Compliance Policy v3.2.

**🔗 Live chatbot:** https://cloud.flowiseai.com/chatbot/01b57783-a585-4a63-b70c-49dd33dcf776


---

## Stack

| Component | Choice | Why |
|---|---|---|
| Platform | Flowise Cloud (free tier) | Required by the task; visual chatflow builder with built-in Document Stores |
| Embeddings | Cohere `embed-english-v3.0` (1024-dim) | Batches ~96 texts per API call, so 2,022 chunks embed in ~21 requests — comfortably inside free-tier rate limits |
| Vector store | Pinecone Serverless (1024-dim, cosine, `us-east-1`) | Free tier easily holds 2k+ vectors; native Flowise integration |
| LLM | Google Gemini `gemini-2.0-flash`, temperature 0.2 | Strong free-tier model; low temperature keeps answers factual and grounded |
| Chain | Conversational Retrieval QA Chain, Top K = 10, Return Source Documents ON | Top-10 retrieval so list-style questions (e.g. 16 disrupted suppliers) get full coverage |

## Architecture

```
supplier_performance_data.csv ──(CSV loader, 1 row = 1 chunk)──┐
SupplyChain_Governance_Policy_v3.2.pdf ──(PDF loader + splitter)─┼─► Document Store ─► Cohere embeddings ─► Pinecone
                                                                                                              ▼
User question ─► Cohere (query embedding) ─► Pinecone top-10 retrieval ─► Gemini 2.0-flash ─► grounded answer
```


The raw CSV (2,000 row-chunks) still serves PO-level and supplier-level lookups; the summary serves aggregate questions; the policy PDF serves rule/threshold questions. Three granularities, one index.

## Document Store & Chunking Experiments

All three sources were loaded into a single Flowise Document Store (**BQBYTE KB v2**) and upserted into Pinecone (final record count: **2,022**).

| Source | Loader | Splitter | Chunks |
|---|---|---|---|
| `supplier_performance_data.csv` | CSV File (no column extraction) | None — 1 row = 1 chunk | **2,000** |
| `SupplyChain_Governance_Policy_v3.2.pdf` | PDF File (per page) | Recursive Character, **1200 / 150** | **17** |


### Two chunk configurations tested (policy PDF)

| Config | Chunk size / overlap | Chunks | Observation |
|---|---|---|---|
| A | 500 / 50 | **35** | Policy tables fragmented — e.g. the Tier classification matrix and OTD penalty bands were split mid-table, so a retrieved chunk often contained half a table without headers |
| B (kept) | 1200 / 150 | **17** | Each policy section/table fits in one chunk; retrieval returns self-contained, citable units (§-level granularity) |

**Conclusion:** for a structured policy document with embedded tables, larger chunks aligned to section size materially improve answer quality; the smaller config optimizes recall of fragments at the cost of losing table context. Screenshots of both previews are in `/screenshots/`.

The system prompt instructs the model to answer strictly from retrieved context, treat the analytics summary as authoritative for aggregates, cite policy sections, and admit when an answer is not in the data. (Implementation note: in Flowise's Conversational Retrieval QA Chain, a custom Response Prompt **must include the `{context}` placeholder** — omitting it silently discards retrieved chunks and produces pure hallucination. Diagnosed during testing via the Document Store Retrieval Playground.)

## Sample Q&A (verbatim from the deployed bot)

> **Note on figures:** all answers are computed from the CSV file provided with this task. They differ from the sample answer key in the task PDF, which appears to have been generated from a different version of the dataset (e.g. answer key total spend $399.5M vs $356.0M in the provided file; 11 vs 16 Tier-3 disrupted suppliers). A discrepancy table is included below. Counts are computed at **supplier level** (per-supplier averages / current flags), not per-PO row.

**Q1. Which Tier-3 suppliers have an active disruption flag, and what response level applies per policy?**

> There are 16 Tier-3 suppliers with active disruption flags. For these suppliers, if they are classified as High Risk, a Level 3 Activate response applies (§9). This response includes immediate escalation to the CPO, activation of an alternate supplier for a minimum of 40% of volume within 10 business days, and a 50% adjustment to safety stock (§9).

**Q2. Which suppliers qualify for the annual Volume Rebate Program and how many are there?**

> [PASTE VERBATIM ANSWER — expected: 0 suppliers meet all four §4.2 criteria (Tier-1, OTD ≥ 93%, Defect < 0.5%, Sustainability ≥ 85) in this dataset; closest candidates listed]

**Q3. Which region has the highest total PO value, and does it breach the concentration limit?**

> [PASTE VERBATIM ANSWER — expected: APAC at $131,620,356.14, ~37.0% of $356,045,248.18 total; below the 45% cap (§5.3), no breach]

**Q4. Which suppliers are on Supplier Watch List (SWL) status and what does it restrict?**

> [PASTE VERBATIM ANSWER — expected: Buenos Aires Pack (SUP-092) and Maghreb Castworks (SUP-080), avg Compliance Score < 60; SWL limits new PO issuance to 20% of prior quarter volume (§3.4)]

**Q5. Which product category has the highest average defect rate and does it exceed the Tier-2 limit?**

> [PASTE VERBATIM ANSWER — expected: Packaging Materials at 1.91% average across 429 POs; below the 2.50% Tier-2 ceiling (§3.2), no breach]

### Dataset discrepancy vs. the task's sample answer key

| Question | Task answer key | Provided CSV (this build) |
|---|---|---|
| Q1 Tier-3 + disruption | 11 suppliers | 16 suppliers (supplier-level) |
| Q2 Rebate qualifiers | 19 suppliers | 0 qualify; 8 near-misses |
| Q3 Top region | EMEA, $193.99M, 48.5% — breach | APAC, $131.62M, 37.0% — no breach |
| Q4 SWL | 11 suppliers | 2 suppliers |
| Q5 Highest defect category | Mechanical 2.12% / 360 POs | Packaging 1.91% / 429 POs |

The figures in the answer key are internally consistent with a *different* dataset (total spend $399.5M; the provided file totals $356.0M and contains different supplier flags). This build answers truthfully from the file supplied.

## Repository structure

```
scm-assistant-bot/
├── scm_assistant.json          # Exported Flowise chatflow
├── derived_analytics_summary.md# Precomputed aggregates embedded alongside source docs
├── screenshots/                # Document store, chunk configs (35 vs 17), upsert, Pinecone count, chatflow, public chat
├── README.md
└── .gitignore                  # excludes .env and key files
```

## Reproducing

1. Create a Flowise Cloud account and add credentials for Cohere, Pinecone, and Google Generative AI (keys are **not** included in this repo).
2. Create a Pinecone serverless index: dimension **1024**, metric **cosine**.
3. Create a Document Store, add the three loaders with the settings in the table above, and Upsert All (embeddings: Cohere `embed-english-v3.0`, type `search_document`).
4. Import `scm_assistant.json` (Chatflows → Add New → Load Chatflow), re-attach your credentials, select the document store, and Save.
5. Share → Make Public to obtain a public URL.

## What I'd improve

- **Live aggregations instead of precomputed ones** — replace the analytics summary with an Agentflow using a SQL/code tool (or an MCP server) that queries the CSV at runtime, so arbitrary aggregate questions ("average lead time of EMEA Tier-2 suppliers") are computed exactly rather than retrieved.
- **Hybrid retrieval** — add a keyword/sparse retriever (or Cohere Rerank) on top of dense retrieval; exact identifiers like `SUP-017` benefit from lexical matching.
- **Record Manager** — enable Flowise's record manager so re-upserts deduplicate instead of appending.
- **Evaluation harness** — script the 5 canonical questions against the prediction API and assert key figures, so any flow change is regression-tested automatically.
- **Production LLM quota** — the free Gemini tier has small daily caps; a production deployment would use a paid key or a higher-headroom provider (e.g. Groq) to survive evaluation traffic.


*Built with Flowise Cloud · Cohere · Pinecone · Google Gemini — submitted by Shrutika Mahajan
