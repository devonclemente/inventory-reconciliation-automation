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
[1] Google Drive — Watch Files in a Folder
  └─ [5] Google Drive — Download a File
       └─ [38] Make AI Content Extractor — Extract text from an image
            └─ [2] Google Gemini AI — Generate a response (full prompt below)
                 └─ [6] JSON — Parse JSON
                      └─ [8] Router — overall_confidence score
                           │
                           ├─ ≥ 0.7 (High Confidence)
                           │    └─ Iterator (one item at a time)
                           │         └─ [22] Router — document_type
                           │              ├─ packing_slip (Incoming Delivery)
                           │              │    └─ [23] Sheets: Search Rows
                           │              │         └─ [25] Sheets: Update a Row (add qty)
                           │              │              └─ [29] Sheets: Add a Row (Transaction_Log)
                           │              │
                           │              └─ pick_ticket (Outgoing Delivery)
                           │                   └─ [24] Sheets: Search Rows
                           │                        └─ [26] Sheets: Update a Row (deduct qty)
                           │                             └─ [30] Sheets: Add a Row (Transaction_Log)
                           │                                  └─ [40] Array Aggregator
                           │                                       └─ [33] Sheets: Search Rows Advanced
                           │                                            (select * where D <= E)
                           │                                            └─ [34] Gmail — Low Stock Alert
                           │
                           └─ < 0.7 (Low Confidence)
                                └─ [n] Sheets: Add a Row (Review_Queue)
                                     └─ [n] Gmail — Manual Review Alert
```

---

## Data Model

Three Google Sheets act as the database:

| Sheet | Purpose |
|-------|---------|
| `Master_Inventory` | Current stock levels per part number |
| `Transaction_Log` | Record of every processed document |
| `Review_Queue` | Low-confidence extractions awaiting human review |

### Master_Inventory

| Internal Part # | Vendor Interchange | Description | Qty On Hand | Reorder Point | Last Updated |
|----------------|-------------------|-------------|-------------|---------------|--------------|
| AAP-BRAKE-PAD-F150 | BMX-BP-F150-FR | Front Brake Pads (Ford F-150) | 44 | 10 | 10/26/2025 9:42 PM |
| AAP-OIL-FILTER-STD | FTS-OF-2840 | Standard Oil Filter | 272 | 75 | 10/26/2025 9:49 PM |
| AAP-COOLANT-1GAL | FTS-AF-1000 | Coolant/Antifreeze (1 Gallon) | 73 | 30 | 10/26/2025 9:49 PM |

Note the **Vendor Interchange** column — packing slips arrive with vendor part numbers (BMX-BP-F150-FR). The workflow resolves them to internal AAP numbers before updating stock.

### Transaction_Log

| Timestamp | Doc Type | Part # | Qty Change | Prev Bal | New Bal | Confidence |
|-----------|----------|--------|------------|----------|---------|------------|
| 10/26 21:37 | pick_ticket | AAP-BRAKE-PAD-F150 | -10 | 24 | 14 | 1.0 |
| 10/26 21:38 | packing_slip | FTS-AF-1000 | +50 | 25 | 75 | 1.0 |
| 10/26 21:42 | packing_slip | BMX-BP-F150-FR | +30 | 14 | 44 | 1.0 |

Pick tickets = negative qty (outbound). Packing slips = positive qty (inbound).

### Review_Queue

| Timestamp | Document Name | Issue | Status |
|-----------|--------------|-------|--------|
| 10/26 21:54 | ZPIC2-IndustrialFasteners_PackingSlip_Sept21.png | LOW CONFIDENCE EXTRACTION - Score: 0.6 | Pending Review |

Row is highlighted red in Google Sheets. Low-confidence items never touch inventory — they wait for human review.

---

## Test Document Design

The workflow was tested against a realistic set of documents — not just happy-path cases.

**Pick tickets** — unified internal format from Automation Auto Parts (AAP). Clean, consistent layout.

**Packing slips** — multiple vendors with different layouts, fonts, and formatting:
- BrakeMax Distributors (professional formatted)
- Everlast Batteries (different header style, teal/gold branding)
- Industrial Fasteners Inc. — **deliberately degraded**: faded text, low contrast, hard to read. Used to validate the low-confidence path actually works.

The Industrial Fasteners document is what you see in Review_Queue with score 0.6. That wasn't an accident — it was a test case.

---

## Gemini Prompt

The actual prompt used in the Google Gemini AI module:

```
You are an inventory management AI analyzing shipping documents for Automation Auto Parts.

TASK: Extract structured data from the document below.

RULES:
1. Identify if this is a "packing_slip" (incoming delivery from vendor) or "pick_ticket" (outgoing shipment to customer)
2. Extract vendor/customer name
3. Extract document date in YYYY-MM-DD format
4. Extract ALL item details with part numbers and quantities
5. Assign confidence scores (0.0-1.0) for each item based on field clarity:
   - 1.0 = perfectly clear
   - 0.7-0.9 = mostly clear, minor uncertainty
   - 0.4-0.6 = somewhat unclear
   - 0.0-0.3 = very unclear or missing data
6. If part numbers are unclear, quantities are ambiguous, or text is faded/damaged, set confidence < 0.7

CRITICAL OUTPUT FORMAT:
Your response must start IMMEDIATELY with {
Do NOT write "json" or any word before the opening brace
Do NOT use markdown, code blocks, or backticks
Your ENTIRE response = the JSON object only
First character: {    Last character: }    Nothing else

Required JSON structure:
{
  "document_type": "packing_slip or pick_ticket",
  "vendor_or_customer": "company name",
  "date": "YYYY-MM-DD",
  "items": [
    {
      "part_number": "exact part number from document",
      "description": "item description",
      "quantity": number,
      "confidence": 0.0-1.0
    }
  ],
  "overall_confidence": 0.0-1.0
}

DOCUMENT TEXT:
[38. Text]
```

The `overall_confidence` field drives the Router 8 decision. Items array is processed one-at-a-time by the Iterator module downstream.

---

## Email Alerts

There are two separate email alerts in the system, triggered by completely different conditions.

### 1. Manual Review Alert (low confidence path)
**Trigger:** `overall_confidence < 0.7`
**When:** Immediately, before any inventory is touched
**Recipients:** Warehouse manager
**Subject:** `MANUAL REVIEW REQUIRED: [document name]`
**Body:** Document name, confidence score, instruction to check Review_Queue sheet and process manually

The document is flagged in Review_Queue and the workflow ends — no inventory changes are made.

### 2. Low Stock Alert (pick ticket path only)
**Trigger:** After pick ticket inventory deductions, any part where `Qty On Hand ≤ Reorder Point`
**When:** After inventory is updated
**Recipients:** Warehouse manager
**Subject:** `⚠ LOW STOCK ALERT - Parts Below Reorder Point: [part number]`
**Body:** Part number, current stock count, reorder threshold, instruction to place reorder

The query that finds eligible parts: `select * where D <= E` (D = Qty On Hand, E = Reorder Point) on Master_Inventory. The Array Aggregator module collects all flagged parts from the run first — one email per document, not one per part.

**Note:** Low stock alerts only fire on pick tickets (outbound). Packing slips add inventory — they don't trigger reorder checks.

---

## Technical Challenges

These were the real bugs — the ones that looked like they should work until they didn't.

### 1. JSON parsing failures
The first version used Make's built-in document extractor. Gemini returned prose mixed with JSON — sometimes adding explanation text before or after the JSON block. The JSON parser choked on it.

**Fix:** Rewrote the Gemini prompt to explicitly say "Return ONLY valid JSON. No explanation, no markdown." Added strict output structure with an example. Pass rate went from ~60% to ~97%.

### 2. Document extractor breaks the entire confidence system
Testing with deliberately degraded, hard-to-read documents returned confidence scores of 1.0 every time. Make's AI Content Extractor runs OCR *before* passing anything to Gemini — it converts the image to clean text first. By the time Gemini sees it, the visual quality information is already gone. You can't make a hard-to-read document in document mode because no matter what it looks like, the computer reads it perfectly.

**Fix:** Switched to image mode. The workflow now passes the raw image directly to Gemini Vision. Gemini sees the same pixels a human would — a faded document looks faded, and gets a low confidence score. This is the only way the quality control routing path actually works.

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
