# EntryLens: Intelligent Voucher Classification Using Open-Source LLMs

**Hacktoberfest Hack Day | Challenge 4: Intelligent Voucher Classification Using Open-Source LLMs**

> **Members:** Sanket Kharche, Tanay Patil, Atharva Andhare

**EntryLens doesn't just predict a voucher type. It tests whether that prediction makes accounting sense.**

---

## 1. Problem Statement

Accounting systems need every transaction recorded under the correct voucher type. Keyword rules fail because many voucher types carry almost identical fields and differ only in accounting meaning.

This project addresses **Challenge 4**. The input is an Excel (`.xlsx`) file of already-structured transactions, where each row is a transaction or document and the **voucher-type column is intentionally missing**. This is **not** an OCR or invoice-extraction task. The system must predict **one voucher category per row** by reasoning over the complete transaction context.

Accuracy alone is not enough. A classifier that cannot tell which of its predictions to doubt forces accountants to re-check every row, and a confidently wrong voucher type silently misposts entries. EntryLens therefore answers two questions for every row: **what voucher is this?** and **does that answer make accounting sense, and how far can it be trusted?**

### Core challenge: semantically similar categories

| Look-alike pair or group | What actually separates them |
|---|---|
| Purchase vs Sales | Whether the company is the buyer or the seller |
| Purchase Return / Debit Note vs Sales Return / Credit Note | Return indicators plus direction: goods going back to a supplier, or coming back from a customer |
| Payment vs Receipt vs **Contra** | Money out vs money in with an external party, vs transfers between the company's own cash/bank accounts |
| Journal vs Purchase/Sales | Adjusting Dr/Cr entries with no goods or payment flow |
| Salary / Payroll vs Expense vs Attendance | Employee and payroll fields (basic pay, deductions, days worked) vs ordinary overheads |
| Inventory movement (Material In/Out, Receipt/Delivery Note, Stock Journal, Physical Stock, Rejection In/Out) vs Purchase/Sales | Quantity movement with little or no monetary or GST value; order, challan, or count references |
| Purchase Order / Sales Order vs Purchase/Sales | Order references with no invoice, payment, or stock movement yet |
| Import / Export vs domestic Purchase/Sales | Foreign currency, customs or port details, and zero-rated or IGST treatment |
| Advance / Prepayment vs Payment | Payment made before goods or invoice, tied to an order reference |
| Job Work In/Out Order vs Material In/Out | Material sent or received for processing, with a job-work reference |

## 2. Target Voucher Categories (27)

The classifier outputs exactly one label from this closed set:

| Group | Categories |
|---|---|
| Trade | Purchase, Sales, Import, Export |
| Returns and notes | Purchase Return / Debit Note, Sales Return / Credit Note, Rejection In, Rejection Out |
| Money movement | Payment, Receipt, Contra, Advance / Prepayment |
| Adjustments and expenses | Journal, Expense |
| People | Salary / Payroll, Attendance |
| Orders | Purchase Order, Sales Order, Job Work In Order, Job Work Out Order |
| Stock and logistics | Receipt Note, Delivery Note, Material In, Material Out, Stock Journal, Physical Stock |
| Fallback | Other / Miscellaneous |

## 3. Proposed Solution

A **hybrid classifier** in which a fine-tuned **open-weight small language model (SLM)** is the primary intelligence layer. A lightweight feature layer enriches its input, and a **trust layer** tests every prediction before it is accepted, so ambiguous rows are verified from several angles or sent to a reviewer instead of being silently mislabeled.

1. **Understand the transaction.** Ingest and normalize the Excel dataset: harmonize column names, coerce types, and mark missing fields explicitly.
2. **Engineer semantic signals** from the row: company-as-buyer or seller, GST presence, Dr/Cr flags, payroll fields, import/export markers, return indicators, order or challan references, and stock-only movement. Each signal keeps a link to the raw fields it came from.
3. **Serialize** each row and its signals into a compact structured prompt.
4. **Classify** with a QLoRA-fine-tuned open SLM using constrained (schema-guided) decoding, so the output is always valid JSON with a label from the 27-class set. The probability of every label is kept for the next steps.
5. **Test the prediction** (Section 4). A counterfactual probe checks that the label changes the way accounting says it should when the transaction changes. An evidence graph shows which input fields support or contradict the label. A conformal prediction set lists the plausible labels. An out-of-distribution score measures how familiar the row is.
6. **Route by confidence.** Clean rows are accepted. Uncertain rows go through multi-view verification, which includes a retrieved-examples view. Rows that stay split go to a human reviewer with their competing labels, and rows that remain very uncertain abstain to `Other / Miscellaneous` and are flagged for review.
7. **Output and audit.** Write machine-readable JSON/CSV, with an audit record for every row and a prioritized review queue.

## 4. Signature Features: Testing Whether a Prediction Makes Accounting Sense

EntryLens does not stop at predicting a label. Five features test whether each prediction makes accounting sense and decide how far to trust it. Each one is either cheap enough to run on every row or reserved for the rows that need it.

| # | Feature | Question it answers | Runs on |
|---|---|---|---|
| 4.1 | Counterfactual Accounting Engine | If the accounting relationship were reversed, does the label change the way accounting says it should? | One targeted probe per row; full suite in evaluation |
| 4.2 | Accounting Evidence Graph | Which actual input fields support or contradict this label? | Every row (deterministic) |
| 4.3 | Multi-View AI Verification | Do different views of the same row lead to the same label? | Difficult rows only |
| 4.4 | Conformal Prediction Sets | Which labels are plausible at 90% coverage, and is that set small enough to act on? | Every row (threshold on SLM probabilities) |
| 4.5 | Out-of-Distribution Detector | Does this row look like anything we have seen before? | Every row (one embedding lookup) |

### 4.1 Counterfactual Accounting Engine

Instead of only asking *"What voucher is this?"*, EntryLens also asks *"What would have to change for this transaction to become another voucher type?"* The engine applies a deterministic transformation to the row, states the label that accounting says the new row should have, and asks the same SLM to classify the transformed row.

```
Original    Company = Buyer, Supplier = ABC, Invoice = INV123, GST = IGST  ->  Purchase
Transform   swap buyer and seller
Expected    Sales
Check       SLM returns Sales     ->  consistent
            SLM returns Purchase  ->  inconsistent, escalate to verification
```

| Transformation | Expected label shift |
|---|---|
| Buyer ↔ Seller | Purchase ↔ Sales |
| Money in ↔ Money out | Receipt ↔ Payment |
| Domestic ↔ Foreign | Purchase ↔ Import (Sales ↔ Export on the sell side) |
| Invoice present ↔ removed | Purchase ↔ Purchase Order |
| Physical quantity ↔ Book quantity | Physical Stock ↔ Stock movement |

The same direction swap extends to the other paired categories, such as Receipt Note ↔ Delivery Note, Material In ↔ Material Out, and Purchase Return ↔ Sales Return. Where no transformation applies to the predicted label (payroll rows, for example), the probe is skipped.

The engine is used in three places:

- **Training:** hard look-alike pairs are generated as minimal counterfactual pairs, so the model sees exactly the one change that flips the label.
- **Inference:** one targeted probe per row, chosen to separate the predicted label from its closest competitor. A failed probe lowers confidence and escalates the row to verification.
- **Evaluation:** the full transformation suite runs on the held-out test set and reports a counterfactual consistency rate per transformation.

This is a metamorphic consistency layer rather than another classifier. It shows that the model has learned accounting relationships, not memorized phrases.

### 4.2 Accounting Evidence Graph

Instead of reporting only "Purchase, 94%", EntryLens shows why. Every prediction carries a graph that traces the decision from the raw input to the label: raw field → normalized field → semantic signal → candidate classes → model prediction.

```
                 ┌── Company acts as buyer
                 ├── Supplier identified
 Transaction ────┼── Invoice present
                 ├── Item quantities present
                 └── GST present
                          │
                          ▼
                   Evidence graph ──► PURCHASE
```

Each semantic signal declares which voucher types it supports and which it contradicts. Signals that agree with the prediction are shown as supporting evidence, and signals that disagree as conflicting evidence. A prediction that carries conflicting evidence, or that falls outside the candidate classes, is escalated to verification.

```
Prediction: Purchase

Supporting evidence:
  ✓ Company acts as buyer
  ✓ Supplier identified
  ✓ Invoice present
  ✓ Item quantities present
  ✓ GST present

Conflicting evidence:
  None
```

Every item links back to real input fields, so the explanation is traceable to the data rather than generated as free text. The graph extends the deterministic signal layer the pipeline already has. Signal-to-label relationships are kept in one declarative rulebook that is shared with the synthetic data generator (Section 11) and the counterfactual engine, so the three stay consistent.

### 4.3 Multi-View AI Verification

Most rows are accepted on the first pass. Only rows the confidence layer does not accept outright are re-read from four views. All four views use the same SLM, so no extra models are loaded.

| View | What the SLM reads | What it tests |
|---|---|---|
| Raw transaction view | The full normalized row | Baseline reading |
| Signal view | Semantic signals only | Whether the structured evidence alone supports the label |
| Narration-masked view | The row with free-text narration or remarks hidden (where such columns exist) | Whether the label depends on wording that may mislead |
| Retrieved-examples view | The row plus the most similar labeled examples from the library (FAISS) | Whether similar known cases agree |

| Case | Raw | Signals only | Narration-masked | Retrieved examples | Agreement | Outcome |
|---|---|---|---|---|---|---|
| A | Purchase | Purchase | Purchase | Purchase | 4/4 | Accepted as Purchase (`verified`) |
| B | Purchase | Receipt Note | Purchase | Receipt Note | 2/4 | Human review (`review`) with set {Purchase, Receipt Note} |

Model disagreement itself becomes an uncertainty signal. The agreement threshold is tuned on the calibration split.

### 4.4 Conformal Prediction Sets

Instead of forcing a single answer with a bare `confidence = 0.72`, EntryLens returns the set of voucher types that are plausible at a chosen coverage level (90% by default, configurable). Conformal prediction is designed to produce such sets with a coverage guarantee under its assumptions, rather than a single point prediction.

A held-out calibration split, separate from training and test, sets the probability threshold (split conformal prediction). At inference, the prediction set contains every label whose probability clears that threshold.

| Prediction set | Reading | Action |
|---|---|---|
| `{Purchase}` | High certainty | Accept |
| `{Purchase, Receipt Note}` | Moderate certainty | Verify (Section 4.3), then review if the views stay split |
| Empty, or too large to be useful | Very uncertain | Verify; abstain to `Other / Miscellaneous` if still unresolved |

The guarantee assumes that calibration rows and deployment rows are exchangeable, which real data may not satisfy. EntryLens therefore reports empirical coverage on the held-out test set instead of relying on the guarantee alone, and the OOD detector (Section 4.5) flags rows where the assumption is likely to fail.

### 4.5 Out-of-Distribution Detector

The model should be able to say *"this transaction doesn't look like anything I was trained on."* Each normalized transaction is embedded with the open sentence-embedding model used for retrieval and compared, through FAISS, with the known transaction library (the embedded training examples). The same index also serves the retrieved-examples view.

| Similarity to the nearest known transactions | Reading | Action |
|---|---|---|
| High (illustrative: 0.91) | Familiar | Normal path |
| Low (illustrative: 0.38) | Unfamiliar transaction | Always verified; review priority raised |

The threshold is set from the similarity distribution on the calibration split. An unfamiliar row loses the fast path and moves up the review queue, but it abstains only if verification also fails to settle on a label.

This matters because the hidden evaluation records may differ from our synthetic training data. Run on the provided dataset, the same score also measures that gap and shows which kinds of rows to add to training.

### 4.6 How the Features Combine: Confidence Layer and Routing

The confidence layer combines five signals: the label probabilities and prediction set (4.4), counterfactual consistency (4.1), conflicting evidence (4.2), similarity to known transactions (4.5), and view agreement when verification runs (4.3). It routes every row to one of three outcomes.

| Route | When | Result |
|---|---|---|
| **Accept** | Single-label prediction set, probe consistent or not applicable, no conflicting evidence, familiar row | Final label right away (`accepted`) |
| **Retrieval + Verify** | Anything else | Multi-view verification. Agreement gives `verified`; a split gives `review`, with the best-supported label emitted and the competing labels listed |
| **Abstain** | Very uncertain: verification still cannot settle on a plausible label | `Other / Miscellaneous`, queued for review (`abstain`) |

Every decision is written to an audit record: evidence graph, probe result, prediction set, view votes, OOD similarity, and final status. The review queue is ordered by review priority, with unfamiliar and low-agreement rows first, and a reviewer's confirmation or correction is logged next to the original prediction.

## 5. Target Users

- Accountants and bookkeeping firms processing high transaction volumes
- ERP and accounting platform teams (such as VYOM+) automating voucher creation
- SMEs without dedicated accounting staff
- Finance reviewers and auditors who need a traceable reason for every assigned voucher type
- Invoice-extraction pipelines that need a voucher type after OCR, so this system can serve as the bridge from extraction to automated voucher creation

## 6. Selected Open-Source AI Technology

| Component | Choice (final pick benchmarked during the hackathon) |
|---|---|
| Primary model | Open-weight SLM, 1B to 4B parameters. Candidates: **Qwen**, **Gemma**, **Llama**, **Phi**, **Mistral** |
| Adaptation | **LoRA / QLoRA** fine-tuning with 4-bit quantization |
| Structured output | Grammar or JSON-schema guided decoding (e.g., Outlines or llama.cpp grammars; vLLM also supports choice, JSON-schema, and grammar constrained outputs) |
| Embeddings (retrieval view and OOD detection) | Open sentence-embedding model (BGE or E5 family) |
| Vector search | **FAISS**: nearest labeled examples for the retrieved-examples view, nearest-neighbor similarity for the OOD detector |
| Uncertainty | Split conformal prediction on the SLM's label probabilities, calibrated on a held-out split |
| Evidence graph | NetworkX graph built from the signal layer and rendered in the review UI |
| Serving | Local inference via llama.cpp, vLLM, or Transformers |

**No proprietary API is used for classification**, in line with the challenge requirement.

## 7. Role of AI

The SLM is the **decision-maker**. It reads the whole transaction context and chooses the voucher type. Rules and features do not decide the label. They only enrich the input so the model can reason about accounting meaning: for example, "company is the buyer, GST charged, items received" suggests Purchase, while "quantities only, no monetary value" suggests Stock Journal or Material In. Fine-tuning teaches the boundaries between look-alike categories that keyword rules and prompt-only approaches miss.

The trust layer follows the same principle. Counterfactual probes and multi-view verification re-query the same SLM with modified inputs. Conformal calibration and the OOD detector measure how far its output can be trusted. The evidence graph is built from deterministic signals and only explains or cross-checks the prediction. None of these replaces the model's judgement. They decide whether to accept it, verify it, or hand it to a human, which is how EntryLens tests whether a prediction makes accounting sense.

## 8. Architecture

> **TODO: Architecture diagram goes here.** Reserved for the end-to-end EntryLens pipeline.

<!--
Components to show (left to right):
Transaction Understanding (ingestion and normalization) and Semantic Signals
→ Prompt Builder
→ Fine-tuned SLM (QLoRA, constrained decoding, 27 labels)
→ Counterfactual Checking, Evidence Graph, OOD detector
→ Confidence Layer (conformal prediction sets)
→ Accept, Retrieval + Verify (multi-view), or Abstain
→ Final Label
→ Audit + Review
→ JSON / CSV output

Image option: ![EntryLens architecture](docs/architecture.png)
-->

## 9. Data Flow

1. **Input:** an `.xlsx` file in which each row holds fields such as seller/supplier, buyer/customer, invoice number and date, item descriptions, quantities, taxable value, GST, discounts, freight, payment details, currency, import/export details, payroll information, debit/credit information, return information, order references, and delivery information.
2. **Preprocess:** clean headers and types. Missing fields are marked "unknown" rather than silently dropped.
3. **Features:** compute the signal flags and record the raw fields behind each one.
4. **Prompt:** serialize the row and signals into a fixed template.
5. **Inference:** the SLM returns a label constrained to the 27 allowed categories, together with the probability of each label.
6. **Trust checks:** build the evidence graph, run the counterfactual probe, compute the conformal prediction set, and score similarity to the known transaction library.
7. **Confidence routing:** clean rows are accepted. The rest go through multi-view verification (raw, signal, narration-masked, and retrieved-examples views). Rows that stay split are flagged for review, and rows that stay very uncertain abstain.
8. **Output:** one record per transaction, plus an audit record and a review queue.

## 10. Expected Output

Minimum output follows the structure given in the challenge:

```json
{ "invoice_number": "INV-2026-1042", "voucher_type": "Purchase" }
```

Optional fields, which we plan to include:

```json
{
  "invoice_number": "INV-2026-1042",
  "voucher_type": "Purchase",
  "confidence": 0.94,
  "status": "accepted",
  "prediction_set": ["Purchase"],
  "evidence": {
    "supporting": [
      { "signal": "company_is_buyer", "fields": ["buyer", "supplier"] },
      { "signal": "supplier_identified", "fields": ["supplier"] },
      { "signal": "invoice_present", "fields": ["invoice_number"] },
      { "signal": "item_quantities_present", "fields": ["quantity"] },
      { "signal": "gst_present", "fields": ["igst"] }
    ],
    "conflicting": []
  },
  "counterfactual": {
    "probe": "buyer_seller_swap",
    "expected": "Sales",
    "observed": "Sales",
    "consistent": true
  },
  "view_agreement": null,
  "ood_similarity": 0.91,
  "review_priority": "low",
  "explanation": "Company is the buyer; GST charged; item quantities received."
}
```

A row that needs review looks like this (other optional fields omitted for brevity):

```json
{
  "invoice_number": "GRN-0457",
  "voucher_type": "Receipt Note",
  "confidence": 0.51,
  "status": "review",
  "prediction_set": ["Purchase", "Receipt Note"],
  "view_agreement": "2/4",
  "ood_similarity": 0.38,
  "review_priority": "high"
}
```

| `status` | Meaning |
|---|---|
| `accepted` | Passed every first-pass check |
| `verified` | Cleared multi-view verification with agreement |
| `review` | Views stayed split. The best-supported label is emitted and the competing labels are listed in `prediction_set` |
| `abstain` | Too uncertain. `voucher_type` is `Other / Miscellaneous` and the row is queued for review |

`confidence` is the SLM's probability for the emitted label, and `prediction_set` is the conformal set at the target coverage. `view_agreement` is `null` when verification did not run. `explanation` is a short text rendered from the evidence graph. Only `invoice_number` and `voucher_type` are required by the challenge, and every other field is additive.

Output is available as JSON and CSV, in a fixed schema that can be evaluated programmatically.

## 11. Training Data and Reproducible Evaluation

The organizers provide an Excel dataset without voucher-type labels, so we will **generate our own labeled training data**:

- Synthesize transactions for all 27 categories from accounting rules, with varied field formats, parties, items, and GST patterns. The provided dataset is used only to study the schema and field distributions, so the synthetic data stays realistic.
- Add **hard look-alike pairs** (the table in Section 1) and **noisy or incomplete rows** (missing fields, inconsistent naming). Hard pairs are generated as **minimal counterfactual pairs**: two rows that differ only in the one attribute that flips the label, such as buyer and seller swapped. The model learns the accounting relationship rather than surface wording, and the same pairs double as the metamorphic test set for Section 4.1.
- Render training prompts in all four view formats (raw, signals only, narration-masked, and with retrieved examples) so the same SLM reads every verification view reliably.
- Build the held-out **test set** with different templates, parties, and wording from training, so evaluation uses genuinely unseen records and is not inflated by leakage.
- Keep a separate **calibration set**, never used for training or testing, to set the conformal threshold, the view-agreement threshold, and the OOD similarity threshold. The OOD library is built from training examples only.

**Evaluation method:** one script, runnable with a single command, that produces:

| Challenge evaluation criterion | How we measure it |
|---|---|
| Accuracy, precision, recall, F1 | Overall accuracy, macro and weighted F1 |
| Per-category performance | Per-class precision, recall, and F1 table |
| Semantically similar categories | Confusion matrix focused on the look-alike pairs |
| Ambiguous or incomplete records | Accuracy on a robustness set with fields masked or removed, plus how often the trust layer flags those rows |
| Output consistency | Rate of valid JSON and valid labels, plus repeat-run agreement |
| Inference speed and efficiency | Rows per second, latency, and memory footprint on a single GPU, with the trust layer on and off |

and the trust-layer metrics:

| Trust-layer feature | How we measure it |
|---|---|
| Counterfactual consistency | Share of counterfactual pairs whose label flips to the expected counterpart, per transformation |
| Evidence graph | Every supporting or conflicting item must trace to an existing input field (checked automatically); share of wrong predictions that carry conflicting evidence |
| Multi-view verification | Accuracy on unanimous vs split rows; share of wrong first-pass predictions caught by disagreement |
| Conformal prediction | Empirical coverage on the test set against the 90% target, average set size, and share of single-label sets |
| OOD detection | AUROC of the similarity score for separating withheld-template rows from training-template rows; similarity distribution of the provided dataset against our synthetic set |
| Selective accuracy | Accuracy on accepted and verified rows against the abstain rate, next to overall accuracy with abstentions counted as `Other / Miscellaneous` |

## 12. Technology Stack

- **Language:** Python
- **Data:** pandas, openpyxl
- **ML:** PyTorch, Hugging Face Transformers, PEFT, bitsandbytes (QLoRA), TRL
- **Constrained decoding:** Outlines or llama.cpp grammars, or vLLM structured outputs (choice, JSON schema, grammar)
- **Retrieval and OOD detection:** sentence-transformers and FAISS
- **Uncertainty:** NumPy and scikit-learn for split conformal calibration
- **Evidence graph:** NetworkX
- **Evaluation:** scikit-learn, matplotlib
- **Interface:** Streamlit or Gradio for uploading an Excel file and viewing predictions, evidence graphs, prediction sets, and the review queue
- **Compute:** a single consumer or cloud GPU (Colab/Kaggle class)

## 13. Implementation Plan (Final Hackathon)

| Phase | Work |
|---|---|
| 1 | Finalize the 27-category schema and field-to-signal mapping; build ingestion, the feature layer, and the evidence-graph builder |
| 2 | Generate the synthetic labeled dataset, including hard pairs as minimal counterfactual pairs and noisy rows; split into train, calibration, and test |
| 3 | Benchmark 2-3 candidate SLMs zero-shot and few-shot as a baseline |
| 4 | QLoRA fine-tune the best candidate; add constrained decoding |
| 5 | Build the trust layer: counterfactual engine, conformal calibration, OOD detector, and multi-view verification with retrieval; tune thresholds and routing |
| 6 | Evaluation script, confusion analysis, trust-layer metrics, and error-driven data iteration |
| 7 | Inference optimization (quantization, batching), review UI with evidence graph and audit view, and demo polish |

## 14. Scalability

- Small quantized models allow **local, on-prem inference**, which keeps financial data private.
- Rows are classified independently, so batching and parallel processing scale to large files.
- Verification is **tiered**. Every row costs up to two short constrained-decoding calls (the prediction and one counterfactual probe) plus cheap checks (evidence graph, conformal threshold, one embedding lookup). Multi-view verification runs only on rows that are not accepted outright.
- One FAISS index serves both the retrieved-examples view and the OOD detector, and keeps lookups fast as the example library grows.
- Audit records and a prioritized review queue let reviewers focus on the rows that need attention instead of re-checking every row.
- New voucher types can be added by extending the label set, the rulebook, and the training data, then recalibrating the conformal threshold, with no architecture change.
- The classifier can sit directly after an OCR/invoice-extraction stage to automate voucher creation end to end.

## 15. Dependencies

Python 3.10+, PyTorch, Transformers, PEFT, bitsandbytes, TRL, pandas, openpyxl, NumPy, scikit-learn, sentence-transformers, FAISS, NetworkX, Outlines (or llama.cpp / vLLM), and Streamlit or Gradio. All are open-source.

## 16. Expected Challenges and Mitigations

| Challenge | Mitigation |
|---|---|
| Fuzzy boundaries across 27 categories | Hard-pair synthetic data built as minimal counterfactual pairs, signal features, targeted confusion-matrix analysis |
| Right label for the wrong reason | Counterfactual Accounting Engine tests whether labels move the way accounting says they should; the evidence graph shows which fields drove the label |
| No labels provided | Rule-guided synthetic data generation with varied formats and noise |
| Synthetic-to-real distribution gap | Diverse templates, noise injection, schema study of the provided dataset, retrieved-examples view; the OOD detector flags unfamiliar rows and measures how far real rows sit from the training library |
| Missing or ambiguous fields | Explicit "unknown" tokens, conformal prediction sets, multi-view agreement, `Other / Miscellaneous` fallback |
| Silent errors on uncertain rows | Confidence-layer routing, audit records, and a prioritized review queue |
| Class imbalance (rare types such as Physical Stock, Job Work orders) | Balanced sampling, class weighting, per-class evaluation |
| Inference speed and compute limits | Small quantized model, constrained decoding, batching, tiered verification (multi-view runs only on rows that are not accepted outright) |

## 17. Compliance with Challenge Rules

- One track only: **Challenge 4**.
- Primary classification engine is an **open-weight LLM/SLM**; no proprietary API. The trust layer reuses the same model and open-source libraries, and it never replaces the SLM as the classifier.
- Qualifier repository contains **README.md only**; implementation happens in the final round.
- Predictions are structured and evaluable programmatically, with a reproducible evaluation method.
