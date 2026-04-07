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

> DocuBot answers developer questions about a small documentation set stored as Markdown files. It can answer with retrieved passages only, or pair retrieval with Gemini to produce short natural-language answers grounded in those passages. The design prioritizes **evidence** and safe refusal over sounding fluent but unsupported.

**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

> A natural-language question from the user; the contents of the `docs/` folder (`.md` and `.txt`); optionally `GEMINI_API_KEY` in the environment (or `.env`) for modes that call the model.

**What outputs does DocuBot produce?**

> Plain text: either raw snippets with filenames (retrieval only), an answer string from Gemini (naive or RAG), or a single refusal sentence when there is not enough evidence.

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

> Each file is split into **paragraph-sized chunks** (blocks separated by blank lines; long blocks are capped around 1400 characters). An **inverted index** maps each lowercase word token to the chunk indices where it appears. For a query, candidate chunks are the union of index postings for all query tokens (or all chunks if none of the tokens appear in the index). Each candidate is scored by summing **whole-word** matches for **substantive** query tokens (common stopwords like “the” or “what” are dropped before scoring). A few **variants** help doc wording line up with questions (e.g. `connect` also counts `connection`). Chunks are sorted by score; the answer uses **at most one chunk per file** for the top results up to `top_k`, so snippets stay diverse and shorter than full documents.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

> **Precision vs recall:** Paragraph chunks are easier to read than whole files but can split a table or section awkwardly. **Guardrails vs coverage:** Requiring a minimum score relative to query length sometimes returns nothing (`“I do not know…”`) even when a weak partial match exists—which is safer than flooding the user with marginally related text. **Simplicity vs linguistic quality:** There is no stemming or embeddings; a small variant list covers a few predictable mismatches (e.g. list/lists) but not all paraphrases.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode:
- Retrieval only mode:
- RAG mode:

> **Naive LLM:** The full concatenated documentation text is sent in the prompt with instructions to answer **only** from that text and to refuse with a fixed sentence if the docs are insufficient. **Retrieval only:** No model call; the user sees ranked snippets (or the same refusal string if retrieval finds no evidence above the threshold). **RAG:** Retrieval runs first; only the returned snippets are passed to Gemini with strict “snippets-only” rules and the same refusal wording if the model cannot ground an answer.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

> Both naive and RAG prompts tell the model its **only** source of truth is the provided documentation or snippets, to **not invent** endpoints or config values, and to reply **exactly** with `I do not know based on these docs.` when evidence is missing. The RAG prompt also asks for a brief mention of which files were relied on when answering.

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| Where is the auth token generated? | Helpful if it cites `AUTH.md`; harmful if it invents file paths | **Helpful** — tight paragraph from token section | **Helpful** — can restate location in one sentence with file name | Naive can sound fluent even when wrong; retrieval shows exact phrase (`generate_access_token` in `auth_utils.py`). |
| How do I connect to the database? | Mixed — may over-claim if it skips `DATABASE_URL` detail | **Helpful** — focused connection / env-var chunks | **Helpful** — summarizes `DATABASE_URL` clearly | Compare whether naive mentions SQLite vs Postgres without conflating them. |
| Which endpoint lists all users? | Often **harmful** under pressure to be specific (invents routes) | **Helpful** — surfaces `GET /api/users` block | **Helpful** — natural phrasing + grounded route | Classic case where retrieval beats ungrounded fluency. |
| How does a client refresh an access token? | Can confuse login vs refresh unless it reads carefully | **Helpful** — `POST /api/refresh` section | **Helpful** — short procedural answer | RAG still fails if retrieval drops the refresh chunk (ranking error). |

**What patterns did you notice?**  

- When does naive LLM look impressive but untrustworthy?  
- When is retrieval only clearly better?  
- When is RAG clearly better than both?

> **Naive** can produce a confident paragraph that blends true doc facts with guessed API details, especially on narrow questions (“which endpoint…”), because the model sees the entire corpus and may overweight generic patterns over the exact sentence.  
> **Retrieval only** is clearly better when you need **verbatim evidence** or want to avoid any generative drift; it is weaker when the user wants a **short synthesis** and must read multiple snippets themselves.  
> **RAG** tends to balance clarity and grounding **when retrieval is good**: it can summarize several snippets and cite files, but it still **inherits retrieval mistakes** (wrong chunk ranked first) and may occasionally paraphrase too loosely unless the prompt is obeyed strictly.

**Illustrative comparison (same wording in each mode):**

| Mode | Strength | Weakness |
|------|----------|----------|
| **Naive** | One coherent narrative; good for overview questions | Weak grounding risk; long prompt = more chance to focus on the wrong section |
| **Retrieval only** | Fully inspectionable evidence; refuses when scores are low | Not a polished “answer”; user must interpret |
| **RAG** | Readable answer + file cues | Depends on snippet quality; can still “hallucinate” if it ignores the prompt |

> **Example — naive sounds confident but weakly grounded:** On an endpoint question, the model might describe a plausible REST layout that does not match `API_REFERENCE.md` (e.g. wrong HTTP verb or path) while sounding authoritative.  
> **Example — retrieval only accurate but hard to interpret:** You get two short chunks (e.g. `SETUP.md` and `DATABASE.md`) without a single sentence tying them together; expert readers are fine; beginners may not know which step comes first.  
> **Example — RAG balances clarity and evidence:** It answers in one paragraph, names `GET /api/users`, and says “per `API_REFERENCE.md`”—*if* that file was retrieved above the evidence threshold.  
> **Example — RAG still fails:** If the guardrail passes a marginal chunk (e.g. troubleshooting bullet that mentions “database” but not connection steps), the model may shorten or overgeneralize and omit `DATABASE_URL`.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

> **Failure 1 — Vague or off-topic question (e.g. “What is the best pizza topping?”)**  
> **What happened:** Retrieval finds no snippet above the evidence threshold and returns **no** snippets; the user sees `I do not know based on these docs.`  
> **What should happen:** Refusal is correct; optionally the product could distinguish “no match” from “nonsense question” with a friendlier message, but silence of evidence is the right **safety** outcome.

> **Failure 2 — Correct file, wrong paragraph**  
> **What happened:** For “How do I connect to the database?” a **troubleshooting** chunk from `SETUP.md` can outrank the **Connection configuration** chunk from `DATABASE.md` because shared keywords (e.g. `database`, `DATABASE_URL`) appear in both. Retrieval may show setup/troubleshooting text before the conceptual overview.  
> **What should have happened:** The highest-ranked snippet should preferably be the section that defines `DATABASE_URL` and SQLite vs production; mitigations include better chunking (e.g. split on `##` headers), positional boosts, or embeddings—not just bag-of-words counts.

**When should DocuBot say “I do not know based on these docs”?**  
Give at least two specific situations.

> The codebase standardizes on: **`I do not know based on these docs.`** Use that whenever: (1) retrieval returns an **empty** list because the best chunk score is **below** the evidence threshold; (2) the LLM (naive or RAG) cannot support the answer from the text it was given. (Phrasing in prompts and in this card may differ slightly in older runs; the intended behavior is the same: **do not guess**.)

**What guardrails did you implement?**  
Examples: refusal rules, thresholds, limits on snippets, safe defaults.

> **Evidence threshold:** The best matching chunk must reach at least `max(2, (n+1)//2)` where `n` is the number of substantive query tokens (after stopword removal). If not, retrieval returns nothing and the UI shows the refusal line.  
> **Stopword filtering:** Scoring ignores very common words so weak overlap on “what is the…” does not look like strong evidence.  
> **Chunking + one chunk per file (initially):** Reduces wall-of-text dumps and spreads evidence across files when scores allow.  
> **Naive + RAG prompts:** Explicit “documentation only” and exact refusal sentence.  
> **Pre-LLM gate in RAG:** If retrieval returns no snippets, Gemini is not called.

---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. **Lexical retrieval only** — synonyms and paraphrases (“sign in” vs “login”) are easy to miss without embeddings or a larger synonym table.  
2. **Chunk boundaries are naive** — Markdown structure (`##`) is not first-class; tables and lists may be split poorly.  
3. **Scoring is easy to game with frequent words** — a chunk that repeats `database` in troubleshooting can beat a more conceptual chunk.  
4. **Single-language, English-oriented stopwords** — non-English docs or code-heavy tokens would need different handling.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. **Structure-aware chunking** — split on headings so each chunk is one `##` section (or a sliding window within long sections).  
2. **Dense retrieval** — small embedding model + vector store for first-stage recall, with lexical scoring as rerank.  
3. **Calibrated thresholds** — learn a score cutoff from labeled Q/A pairs instead of a fixed formula.  
4. **Citation spans** — return character offsets or quoted lines so users can verify claims quickly.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

> Developers might **treat a fluent RAG answer as audited truth** without reading snippets, especially under time pressure. Wrong routing or security guidance (auth, secrets) is high impact. A **false refusal** can waste time, but it is usually lower risk than a **false assertion** about permissions or data handling.

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- **Always open the cited file** for anything security- or data-related (auth, keys, schema).  
- **Treat refusal as a signal** to search the repo or run tests, not as proof the repo lacks an answer.  
- **Prefer retrieval-only mode** when you need evidence for a code review or incident; use RAG when you want a summary after you trust the snippets.  
- **Keep the docs** the assistant reads in sync with production; stale docs defeat every mode equally.

---
