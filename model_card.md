# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
Describe the overall goal in 2 to 3 sentences.

DocuBot is a lightweight assistant that answers developer questions using local project documentation files. It is designed to compare three behaviors: naive generation over all docs, retrieval only, and retrieval augmented generation (RAG). The goal is to improve answer trustworthiness by grounding responses in retrieved evidence and refusing weakly supported answers.

**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

- A user question from the CLI.
- Documentation files from the `docs/` folder (`.md` and `.txt`).
- Optional environment variables for LLM features (`GEMINI_API_KEY`) and app context from docs (`DATABASE_URL`, `AUTH_SECRET_KEY`, etc.).
- Runtime mode choice (naive LLM, retrieval only, or RAG).

**What outputs does DocuBot produce?**

- In retrieval-only mode: ranked paragraph snippets with source filenames.
- In RAG mode: an LLM-written answer constrained to retrieved snippets (or a refusal).
- In naive mode: an unconstrained LLM answer over the full corpus prompt.
- In low-evidence situations: a refusal message ("I do not know based on these docs.").

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

The retrieval pipeline is paragraph-based, not whole-document based. Each file is split on blank lines into paragraphs, then tokenized with a simple regex (`\b\w+\b`) and lowercased.

- **Indexing:** build an inverted index from token -> paragraph IDs.
- **Scoring:** score a paragraph by counting how many unique query tokens appear in that paragraph.
- **Selection:** gather candidate paragraph IDs from query tokens, score candidates, sort by score descending (then filename for stable tie-break), and return top_k.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

- Chose simple regex tokenization for speed and readability over linguistic accuracy.
- Chose paragraph chunks for better granularity than full docs, but this can still miss cross-paragraph context.
- Chose overlap-count scoring for interpretability; it is weaker than semantic similarity for paraphrased questions.
- Added conservative refusal thresholds to reduce hallucinations, at the cost of more "I do not know" outcomes.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode:
- Retrieval only mode:
- RAG mode:

- Naive LLM mode: always calls Gemini with a generic question prompt (not strongly grounded by retrieval context).
- Retrieval only mode: never calls Gemini; only returns retrieved paragraph snippets.
- RAG mode: calls Gemini only after retrieval, and now only when evidence passes guardrails.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

- Use only the provided snippets; do not invent endpoints, functions, or config values.
- If snippets are insufficient, respond with: "I do not know based on the docs I have."
- Mention which files were used when answering.
- Keep answers concise and tied to retrieved evidence.

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| Example: Where is the auth token generated? | Helpful but can omit exact function/module | Helpful | Most helpful | Retrieval surfaces `AUTH.md` paragraph mentioning `generate_access_token` in `auth_utils.py`; RAG summarizes cleanly. |
| Example: How do I connect to the database? | Mixed: may over-generalize engine details | Helpful | Most helpful | Docs mention `DATABASE_URL`, SQLite default, and PostgreSQL option; retrieval gives exact config text. |
| Example: Which endpoint lists all users? | Usually helpful, but can guess wrong path in other contexts | Helpful | Most helpful | `API_REFERENCE.md` explicitly states `GET /api/users`. |
| Example: How does a client refresh an access token? | Sometimes vague ("re-authenticate") | Helpful | Most helpful | Docs clearly specify `POST /api/refresh` and `Authorization: Bearer <token>`. |

**What patterns did you notice?**  

- When does naive LLM look impressive but untrustworthy?  
- When is retrieval only clearly better?  
- When is RAG clearly better than both?

- Naive LLM can sound polished even when it is missing file-specific evidence.
- Retrieval only is strongest when users need exact quoted facts (endpoint names, env vars, table fields).
- RAG is strongest when users need both grounding and concise explanation.
- Guardrails reduce risky confident answers by forcing abstention when overlap evidence is weak.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

**Failure case 1**  
Question: "Is there Stripe payment webhook support?"  
Observed behavior (before guardrails): naive mode produced a plausible but unsupported answer about payment handling.  
What should happen: explicit refusal, because payment processing is not documented.

**Failure case 2**  
Question: "What is the exact token refresh lifetime policy?"  
Observed behavior: retrieval found partial auth snippets but no full policy details; RAG could still over-interpret.  
What should happen: refuse or answer only the known part (`TOKEN_LIFETIME_SECONDS` default and `/api/refresh` behavior) with uncertainty.

**When should DocuBot say “I do not know based on the docs I have”?**  
Give at least two specific situations.

- When no paragraphs match the query tokens with meaningful overlap.
- When top evidence overlaps only generic words and misses key query terms.
- When the question asks about systems/features not present in the docs (for example payments, SSO providers, deployment topology).
- When retrieved snippets are contradictory or too incomplete to support a confident answer.

**What guardrails did you implement?**  
Examples: refusal rules, thresholds, limits on snippets, safe defaults.

- Added paragraph-level evidence scoring and retrieval.
- Added a meaningful-evidence check before answering (`min_evidence_score = 2`, `min_evidence_coverage = 0.4`).
- Added stopword filtering for query-term coverage checks.
- Both retrieval-only and RAG paths now refuse with: "I do not know based on these docs." when evidence is weak.

---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. Lexical token overlap misses semantic matches and paraphrases.
2. Paragraph splitting by blank lines can produce chunks that are too long or too short.
3. Naive mode prompt currently ignores the full docs text variable and is weakly grounded.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. Replace overlap scoring with TF-IDF/BM25-style weighting (still Python-only) for better ranking.
2. Add neighbor-paragraph context windows to preserve local continuity.
3. Improve evaluation with per-query precision/recall and threshold tuning for abstention quality.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

If used as an authoritative source for security or operations decisions, wrong answers could cause misconfigured authentication, incorrect API usage, or unsafe deployment actions. Harm is highest when users trust fluent LLM output without checking source evidence, especially in naive mode. Missing docs coverage can also create false confidence about what a system does or does not support.

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- Always verify critical answers (auth, security, data access) against source files.
- Prefer retrieval-only or RAG with refusal guardrails over naive generation for factual tasks.
- Treat "I do not know" as a safety feature, not a failure, and investigate manually.
- Expand and maintain docs quality; retrieval systems are only as reliable as their corpus.

---
