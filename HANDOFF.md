# Governed Knowledge Assistant — Architecture & Build Handoff

**Status:** partially built, one defect blocking. **Date:** 11 September 2026
**Platform:** Dify Cloud (Professional, free via education plan)
**Owner account:** `24f2005387@ds.study.iitm.ac.in` — signed in under the Chrome profile named **School**

---

## 1. Objective

A policy question-answering assistant that is *governed*, not merely retrieval-augmented. Three properties distinguish it from a chatbot over PDFs:

1. **Revision discipline** — superseded policy revisions exist in the corpus (audit retention) but can never reach the model.
2. **Regional entitlement** — a user only retrieves documents applicable to their region.
3. **Honest refusal** — when the corpus does not cover a question, it says so rather than answering from the nearest-matching passage.

---

## 2. Identifiers

| Thing | Value |
|---|---|
| Knowledge base name | `governed-policy-corpus` |
| Dataset ID | `a2918caa-5ac2-40d4-b6b9-dbd82df087c8` |
| App name | `Governed Knowledge Assistant` (Chatflow) |
| App ID | `8ad0a77a-f945-4411-bc4a-dd9963e3453c` |
| Workflow URL | `https://cloud.dify.ai/app/8ad0a77a-f945-4411-bc4a-dd9963e3453c/workflow` |
| Knowledge API base | `https://api.dify.ai/v1` |
| Knowledge API key | in `~/.dify-dataset-key` (mode 600) — **read from file, never print or commit** |
| Corpus source files | `~/Projects/dify-gka/corpus/` (6 markdown files) |

---

## 3. Current state — VERIFIED WORKING

Do not rebuild these. Each was confirmed against the live API.

### 3.1 Knowledge base

| Setting | Value |
|---|---|
| Chunking | **Parent-child**, Paragraph mode |
| Parent delimiter / max | `\n\n` / 1024 chars |
| Child delimiter / max | `\n` / 512 chars |
| Index method | High Quality |
| Embedding model | `gemini-embedding-001` — **permanent, cannot be changed without full re-index** |
| Retrieval | Hybrid Search + Weighted Score, 0.7 semantic / 0.3 keyword |
| Documents | 6 |

### 3.2 Document metadata — all 6 documents labelled and verified

| Document | status | region | doc_id | revision |
|---|---|---|---|---|
| `01_HR-POL-014_remote-work_rev3.2_CURRENT.md` | current | global | HR-POL-014 | 3.2 |
| `02_HR-POL-014_remote-work_rev2.8_SUPERSEDED.md` | superseded | global | HR-POL-014 | 2.8 |
| `03_FIN-POL-007_expense-india_rev2.0_CURRENT.md` | current | india | FIN-POL-007 | 2.0 |
| `04_FIN-POL-008_expense-emea_rev1.1_CURRENT.md` | current | emea | FIN-POL-008 | 1.1 |
| `05_SEC-POL-002_data-classification_rev1.4_CURRENT.md` | current | global | SEC-POL-002 | 1.4 |
| `06_HR-POL-021_parental-leave_rev1.0_FUTURE.md` | future | global | HR-POL-021 | 1.0 |

Metadata field IDs (if writing via API):

```
status    071909a8-8aed-4506-ad1a-f62608b95259
region    016cbe3d-9ab1-4a9a-80f5-6e5363795fbf
doc_id    08d450b3-4973-4705-86f7-78aeda09f76a
revision  b3cd9d4f-986d-45c4-b0e6-79eba36da5e8
```

### 3.3 Retrieval filtering — proven at API level

Filtering on `status` removes the superseded revision entirely (not merely down-ranks it):

```
Query: "What is the internet stipend in the India region?"

WITHOUT filter:
  0.853  rev2.8 SUPERSEDED  "INR 1,800 ..."   <-- WRONG ANSWER RANKS FIRST
  0.829  rev3.2 CURRENT     "INR 2,500 ..."

WITH status filter:
  0.829  rev3.2 CURRENT     "INR 2,500 ..."
         rev2.8             ABSENT
```

Region filtering produces correctly divergent answers for the same question:

```
Query: "Can I get a travel advance for business travel?"
  region in [emea, global]  -> FIN-POL-008 "Travel advances are NOT offered in the EMEA region"
  region in [india, global] -> FIN-POL-007 "Travel advances up to INR 25,000 may be requested"
```

### 3.4 Score threshold — calibrated at **0.74**

```
ANSWERABLE                          NOT IN CORPUS
  0.850 expense approval limits       0.698 company car policy
  0.829 internet stipend              0.684 gym membership
  0.779 remote work eligibility       0.676 notice period
  0.776 screen lock requirement       0.662 annual leave days
  ---------------------------         ---------------------------
  lowest true hit  0.776              highest false hit  0.698
```

The gap is only 0.078 — narrow. If it closes as the corpus grows, add a **Cohere** reranker (installable from the Dify Marketplace; Tongyi's `qwen3-rerank` must NOT be used — see §7).

---

## 4. Target architecture

```
┌─────────┐   ┌──────────────────────┐   ┌─────────┐   ┌────────┐
│  START  │──▶│ KNOWLEDGE RETRIEVAL  │──▶│   LLM   │──▶│ ANSWER │
│ region  │   │ filtered, thr 0.74   │   │ cited   │   │        │
└─────────┘   └──────────────────────┘   └─────────┘   └────────┘
```

### 4.1 START node

| Field | Value |
|---|---|
| Variable | `region` |
| Type | select |
| Options | `india`, `emea` |
| Required | yes |

Note: this Dify version also exposes `userinput.query` and `userinput.files` on Start. **The chat query variable is `userinput.query`, not `sys.query`** — see §5.2.

### 4.2 KNOWLEDGE RETRIEVAL node

| Field | Value |
|---|---|
| Query text | the chat query variable — select via the variable picker, do not type it |
| Knowledge | `governed-policy-corpus` |
| Rerank setting | **Weighted Score** (never Rerank Model) |
| Weights | 0.7 semantic / 0.3 keyword |
| Top K | 3 |
| Score threshold | enabled, **0.74** |
| Metadata filtering | **Manual** |

Metadata conditions:

| Field | Operator | Value |
|---|---|---|
| `status` | **is not** | `superseded` |
| `region` | **in** | `{{region}}` (from variable picker) + `global` |

`status is not superseded` is deliberate: it admits both `current` and `future` with one simple comparison, and fails safe — a document with an unanticipated status is included rather than silently dropped.

### 4.3 LLM node

| Field | Value |
|---|---|
| Model | `gemini-flash-latest`, provider `langgenius/gemini/google` |
| Temperature | 0 |
| Context | bound to the Knowledge Retrieval node's `result` output |
| Memory | on, window 6 |

SYSTEM prompt — use verbatim:

```
You answer questions about company policy using ONLY the CONTEXT below.

1. Use only facts stated in the CONTEXT. Never draw on outside knowledge of company
   policies, and never infer, estimate or round a figure that is not written there.

2. Cite every factual claim inline as [doc_id rev X, §N] — e.g. [HR-POL-014 rev 3.2, §5].
   Derive doc_id, revision and status from the SOURCE DOCUMENT NAME of the passage you
   used: "01_HR-POL-014_remote-work_rev3.2_CURRENT.md" means doc_id HR-POL-014,
   revision 3.2, status CURRENT. Take the section number from the nearest heading in
   the CONTEXT. If some citation detail is missing, cite what you can identify — never
   withhold an answer because the citation is incomplete.

3. If the CONTEXT contains a policy whose status is not yet effective, you MUST state
   its effective date and say it does not apply yet, BEFORE describing what it contains.

4. If two passages conflict, prefer the one with the later effective date, and say
   explicitly that you did so and why.

5. Refuse ONLY when the CONTEXT does not contain the factual answer. Missing citation
   metadata is NOT a reason to refuse. When refusing, reply with exactly this and
   nothing more:
   "I don't have that in the current policy set. Please contact your regional HR partner."
   Do not apologise, speculate, or offer adjacent information.

6. Answer in plain language. Be brief. Do not add caveats beyond those required above.

7. The user's region is stated in their message. When a passage gives different values
   per region, report ONLY the value for the user's region, and name the region in your
   answer. Never present another region's figure as theirs.

CONTEXT
{{#context#}}
```

USER prompt — **both variables must be inserted via the variable picker, not typed**:

```
Region: {{region}}
Question: {{query}}
```

Rule 7 and the USER prompt exist because of a real gap: `HR-POL-014 §5` is a `global` document containing **both** regional figures in one passage (`INR 2,500` for India, `EUR 40` for EMEA). Document-level filtering cannot separate them — the model must know who is asking.

### 4.4 ANSWER node

`{{#llm.text#}}`

### 4.5 App features

Enable **retriever_resource** (citations panel) so sources render in the UI.

---

## 5. OUTSTANDING DEFECTS — what needs fixing

> **UPDATE, end of 11 Sep 2026.** The app is now **published and working end to end for
> high-scoring queries**. The governance thesis is proven live:
>
> ```
> [india] "What is the internet stipend in the India region?"
>   ANSWER:  "In the India region, the internet stipend is INR 2,500 per month."
>   SOURCES: 01_HR-POL-014_remote-work_rev3.2_CURRENT
> ```
>
> Before the metadata filter was enabled the same question answered **INR 1,800** from
> `02_..._rev2.8_SUPERSEDED`. Enabling `status is not superseded` flipped it. That is the
> whole point of the build, demonstrated in the live app.
>
> **Fixes that were applied tonight (all confirmed):**
> - LLM node CONTEXT field was unbound → bound to Knowledge Retrieval → `result`.
>   This was the root cause of every "refuses but shows citations" result.
> - LLM USER prompt `region` was literal text → re-inserted via the variable picker.
> - SYSTEM prompt `{{#context#}}` was literal text → replaced with a real context chip.
> - Rebuilt node had defaulted to **Rerank Model / `qwen3-rerank`** (Tongyi, AI credits)
>   → switched to Weighted Score.
> - Region metadata reshaped: `global` → `india,emea`, so a single
>   `region contains {{region}}` condition expresses "my region or global". Dify's
>   conditions are AND-joined and cannot express OR across two region conditions.
>
> **REMAINING DEFECT — unresolved.** Only queries scoring roughly **≥ 0.82** retrieve
> anything through the app. Everything below that returns zero rows, even though the
> **dataset API returns them correctly with the identical filter**:
>
> | Query | dataset API | through app |
> |---|---|---|
> | "What is the internet stipend in the India region?" | 0.829 | ✅ rows |
> | "What is my internet stipend?" (emea) | 0.792 | ❌ none |
> | "Can I get a travel advance for business travel?" | 0.770 | ❌ none |
> | "internet stipend" | 0.834 | ❌ none |
> | "Tell me the internet stipend in the India region?" | — | ❌ none |
>
> Ruled out: it is **not** exact-match caching (punctuation and case variants of the
> working query still retrieve), and **not** the node's Score Threshold (disabling it
> changed nothing). `GET /datasets/{id}` returns an empty `retrieval_model`, so the
> dataset's own settings could not be inspected via API.
>
> **Check first when resuming:** Knowledge → `governed-policy-corpus` → **Settings** →
> **Retrieval Setting** → is **Score Threshold** enabled there, at roughly 0.8? A
> dataset-level threshold would apply regardless of the node's setting and matches the
> observed behaviour exactly. If it is on, turn it off or set it to 0.70 and re-test.
>
> Second candidate: the node's Weighted Score arithmetic may normalise differently from
> the dataset API's default, compressing scores. If so, lower the node threshold
> empirically from the app's own behaviour rather than from the dataset API's numbers —
> the two scales are evidently not the same.


### 5.1 LLM node: invalid variable — CONFIRMED

Dify's own checklist reports:

> **LLM** — *Invalid variable. Please select an existing variable.*

Cause: the `region` reference in the USER prompt is stored as literal text rather than a variable binding. It renders as `{x}region ⚠️` while `{x}query` resolves cleanly.

**Fix:** delete the broken chip in the USER prompt, type `/` to open the variable picker, select **Start → region**.

### 5.2 Knowledge Retrieval returns `{"result": []}` — ROOT CAUSE UNCONFIRMED

Symptom, from the node's LAST RUN tab:

```
STATUS  SUCCESS
INPUT   {"query": "What is the internet stipend for my region?"}   <- query arrives correctly
OUTPUT  {"result": []}
```

The same query against the same dataset **returns rows via the API** (see §6), so the data and filter logic are sound — the node's configuration differs from the working API call.

Already ruled out by testing:

- ❌ Not the query variable — INPUT shows correct text
- ❌ Not the metadata filter — still empty with filtering set to Disabled
- ❌ Not Top K — confirmed 3
- ❌ Not the score threshold — confirmed 0.74, and the query scores 0.79
- ❌ Not a missing `region` variable — present and required on Start

Remaining suspects, in order:

1. **QUERY TEXT selector mismatch.** The node was created by DSL import with `query_variable_selector: [sys, query]`, but this Dify version exposes the chat query as **`userinput.query`**. Re-pick it from the variable picker.
2. **Residue from DSL import.** The node was generated from inferred schema. A field may be present but malformed in a way the UI does not surface.

**Recommended fix — do not keep patching the imported node.** Delete it and add a fresh Knowledge Retrieval node through the UI, configured per §4.2. Then **re-point the LLM node's CONTEXT** at the new node — a stale reference to the deleted node produces empty context, reproducing the same symptom from a different cause.

---

## 6. Verifying retrieval without the UI

Use this to confirm whether a failure is in the data or in the node. It has been run successfully many times.

```bash
KEY=$(tr -d '\n\r ' < ~/.dify-dataset-key)
DS=a2918caa-5ac2-40d4-b6b9-dbd82df087c8
FILT='{"logical_operator":"and","conditions":[
  {"name":"status","comparison_operator":"in","value":["current","future"]},
  {"name":"region","comparison_operator":"in","value":["india","global"]}]}'

curl -s -X POST "https://api.dify.ai/v1/datasets/$DS/retrieve" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d "{\"query\":\"What is the internet stipend in the India region?\",
       \"retrieval_model\":{\"search_method\":\"hybrid_search\",
       \"reranking_enable\":false,\"top_k\":3,
       \"score_threshold_enabled\":true,\"score_threshold\":0.74,
       \"metadata_filtering_conditions\":$FILT}}"
```

Two API constraints learned the hard way:

- If you pass a `weights` block, it **requires** `vector_setting.embedding_provider_name` and `embedding_model_name`. Omitting the whole `weights` block is simpler and makes Dify use the dataset's own configuration.
- The Knowledge/Service API is **runtime only**. It cannot create or edit apps or workflows. App authoring is UI or DSL import only.

---

## 7. Constraints and hard-won gotchas

**Credits.** The education plan grants Professional quotas but pins AI credits at **200 one-time, no refill**. 199 remain.

- In the model picker, provider groups labelled **"AI credits"** (OpenAI, Anthropic, Tongyi, xAI) spend that allowance. Groups labelled with a credential name (`gemini-dify-embedding`) bill to the user's own Google account.
- **Never select an OpenAI, Anthropic, Tongyi or xAI model anywhere in this app.**
- Templates ship with OpenAI models hardcoded — the starter Chatflow arrived with `gpt-5.4-mini`. Check the model selector on every node added from a template or the Marketplace.
- The one credit already spent was `qwen3-rerank` (Tongyi) firing during a retrieval test. Rerank fires per *query*, not per document, so it compounds.

**Retrieval.** Always use **Hybrid Search → Weighted Score**, never Rerank Model, unless a non-credit reranker (Cohere with its own key) is installed first. Watch for multi-path retrieval across several knowledge bases, which reaches for a rerank strategy by default.

**Embedding.** `gemini-embedding-001` accepts **2048 input tokens**; chunk sizes must stay under that. The embedding model is fixed per knowledge base at creation and cannot be changed without rebuilding it.

**Uploads.** The document upload screen caps at **5 files per batch** and silently drops extras. The first attempt at this corpus dropped `01_..._rev3.2_CURRENT.md`, leaving only the superseded revision indexed.

**Auth.** A `401` on `upload?source=datasets` after a plan change is a stale session token. Log out and back in.

**DSL import.** Node types copied from a real export (start, llm, answer) import perfectly. The `knowledge-retrieval` node written from API documentation did not. If generating Dify apps programmatically, **export one node of each type you need first**, then edit — do not infer the schema.

---

## 8. Acceptance tests

Run in Preview. All must pass.

| # | Region | Question | Required behaviour |
|---|---|---|---|
| 1 | india | What is the internet stipend in the India region? | **INR 2,500**, cited `[HR-POL-014 rev 3.2, §5]`. Must never say 1,800. |
| 2 | emea | What is my internet stipend? | **EUR 40** only. Must not show the India figure. |
| 3 | india | Can I get a travel advance? | Up to **INR 25,000**, cited FIN-POL-007 |
| 4 | emea | Can I get a travel advance? | **Not offered in EMEA**, cited FIN-POL-008 |
| 5 | india | What is the parental leave policy? | States it is **effective 1 January 2027** and does not apply yet, *before* describing it |
| 6 | india | What is the company car policy? | Exactly: *"I don't have that in the current policy set. Please contact your regional HR partner."* |
| 7 | india | Can a contractor work remotely? | **No** — sourced from SEC-POL-002, a document the question does not obviously point at |

Test 1 is the core governance proof. Test 6 must still pass after any prompt loosening. Test 2 is the within-document regional split that filtering alone cannot solve.

After all pass, confirm `AI CREDITS` still reads **1 / 200 used** — any increase means a credit-consuming model is wired somewhere.

---

## 9. Corpus design notes

The corpus at `~/Projects/dify-gka/corpus/` is synthetic and built with deliberate traps:

- **Revision conflict** — rev 2.8 and 3.2 of HR-POL-014 disagree on every figure (stipend, eligibility period, screen lock, exception limit). The superseded version is *shorter and blunter*, so it scores **higher** on semantic similarity. This is not incidental: newer revisions accumulate qualifications and cross-references that dilute their embeddings. Semantic search structurally favours obsolete documents.
- **Regional conflict** — FIN-POL-007 and FIN-POL-008 disagree on deadlines, receipt thresholds, approval limits, and whether travel advances exist at all.
- **Future effective date** — HR-POL-021 is approved but not effective until 1 January 2027.
- **Cross-document** — contractors are out of scope in HR-POL-014; the answer lives in SEC-POL-002.

Swapping in a real corpus (regulatory circulars with genuine supersession chains, for example) requires only re-ingestion plus metadata labelling; no workflow changes.
