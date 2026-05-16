# Inventory Reconciliation Automation

An AI-powered automation that processes warehouse documents (pick tickets and packing slips) in real time, extracts structured data using Google Gemini Vision AI, and updates inventory instantly — eliminating a 2–3 month backlog.

**Built with Make.com (no-code automation) + Google Gemini 2.0 Flash Vision API**

---

## The Problem

At a small auto parts distributor, inventory was tracked manually from paper documents. Pick tickets and packing slips sat unbilled and unprocessed for **2–3 months** before anyone updated the system.

The result:
- Staff quoting part availability based on data that was months out of date
- Warehouse workers spending up to an hour searching for parts already sold
- Customers called back to say parts weren't available after they'd been promised
- Lost sales, wasted labor, frustrated customers

The manager knew it was a problem. But they used legacy bookkeeping software and had no appetite for replacing it. The automation had to **work around the existing system**, not replace it.

---

## The Solution

A Make.com workflow monitors a Google Drive folder for incoming scanned documents. When a file lands, the workflow:

1. Downloads it
2. Passes it through Google Gemini Vision AI for data extraction
3. Scores confidence on the extraction
4. Routes high-confidence results to automatic inventory updates
5. Routes low-confidence results to a manual review queue + email alert
6. Updates the correct Google Sheet based on document type
7. Sends low-stock alerts when parts fall below reorder threshold

**50 documents/day. 98% accuracy. Real-time vs. 2-3 month delays.**

---

## Workflow Architecture

```
Google Drive (watch folder)
  └─ Download file
       └─ Make AI Content Extractor (base module)
            └─ Google Gemini 2.0 Flash Vision API
                 └─ JSON Parse (structured output)
                      └─ Router: confidence score
                           ├─ ≥ 0.7 → Iterator → Router: doc type
                           │              ├─ pick_ticket → Master_Inventory (deduct qty)
                           │              └─ packing_slip → Transaction_Log (record)
                           │
                           └─ < 0.7 → Review_Queue Sheet + Gmail alert
```

---

## Data Model

Three Google Sheets act as the database:

| Sheet | Purpose |
|-------|---------|
| `Master_Inventory` | Current stock levels per part number |
| `Transaction_Log` | Record of every processed document |
| `Review_Queue` | Low-confidence extractions awaiting human review |

### Sample Master_Inventory schema

| part_number | description | qty_on_hand | reorder_point | reorder_qty |
|-------------|-------------|-------------|---------------|-------------|
| AB-1042 | Air Brake Valve | 12 | 5 | 20 |
| HL-2210 | Headlamp Assembly | 3 | 5 | 15 |

### Sample Transaction_Log schema

| timestamp | doc_type | part_number | qty | confidence | status |
|-----------|----------|-------------|-----|------------|--------|
| 2025-01-15 09:32 | pick_ticket | AB-1042 | 2 | 0.94 | processed |
| 2025-01-15 09:41 | packing_slip | HL-2210 | 5 | 0.61 | review_queue |

---

## Gemini Prompt

The core extraction prompt sent to Gemini 2.0 Flash:

```
You are a warehouse document processor. Extract the following fields from this document image:
- document_type: "pick_ticket" or "packing_slip"
- part_number: the part/SKU identifier
- quantity: numeric quantity (integer)
- confidence: your confidence score from 0.0 to 1.0

Return ONLY valid JSON. No explanation, no markdown.

Example output:
{
  "document_type": "pick_ticket",
  "part_number": "AB-1042",
  "quantity": 2,
  "confidence": 0.94
}
```

---

## Technical Challenges

These were the real bugs — the ones that looked like they should work until they didn't.

### 1. JSON parsing failures
The first version used Make's built-in document extractor. Gemini returned prose mixed with JSON — sometimes adding explanation text before or after the JSON block. The JSON parser choked on it.

**Fix:** Rewrote the Gemini prompt to explicitly say "Return ONLY valid JSON. No explanation, no markdown." Added strict output structure with an example. Pass rate went from ~60% to ~97%.

### 2. Confidence scoring unreliable on scanned documents
The document extractor module returned inconsistent confidence scores on grayscale scans. Same document, different scores on repeat runs.

**Fix:** Switched from the Make AI Content Extractor (document mode) to image mode, passing the file as a raw image to Gemini Vision rather than as a "document." This bypassed Make's pre-processing layer and let Gemini assess the image directly. Scores became consistent.

### 3. Duplicate email alerts
The low-stock alert triggered once per processed line item. A packing slip with 8 parts at or below reorder threshold sent 8 separate emails.

**Fix:** Added an array aggregator module before the Gmail step. Aggregates all low-stock items from a single document run into one array, then sends a single email listing all flagged parts.

---

## Sample Documents

The workflow was tested on fictional documents for a fictional company — **Automation Auto Parts** — modeled closely on the real operational patterns at the actual business.

See [/sample-data](./sample-data/) for document formats.

---

## Results

| Metric | Before | After |
|--------|--------|-------|
| Processing delay | 2-3 months | Real-time |
| Accuracy | Manual, error-prone | 98% |
| Daily throughput | ~0 (backlogged) | 50 documents/day |
| Staff search time | Up to 1 hr/day | Eliminated |
| Low-stock alerts | None | Automated |

---

## Why Make.com (No Code)

This was an intentional choice, not a limitation. The target business runs on legacy software and has no technical staff. A Python script or custom API integration would have been unmaintainable the moment I walked out the door.

Make.com gives non-technical staff a visual workflow they can look at and understand. It's self-documenting. It's maintainable without a developer. For this use case, it was the right tool.

See: [You wouldn't crack a walnut with a sledgehammer, would you?](https://medium.com/@devonclemente/local-vs-cloud-llm-benchmark-ops-data-be4e0f93b46e) — my thinking on matching the tool to the job.

---

## Links

- Full project case study: [devonclemente.com/inventory-reconciliation-project](https://devonclemente.com/inventory-reconciliation-project)
- Medium article: [From Legacy to Llama: How to Spot and Seize AI Automation Opportunities](https://medium.com/@devonclemente/from-legacy-to-llama-how-to-spot-and-seize-ai-automation-opportunities-in-existing-business-2a220e11faa1)
- More projects: [devonclemente.com](https://devonclemente.com)

---

Built by [Devon Clemente](https://devonclemente.com) — AI Process Automation
