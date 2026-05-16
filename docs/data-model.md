# Data Model

Three Google Sheets act as the database for the automation.

---

## Master_Inventory

Current stock levels. Updated in real time as documents are processed.

| Column | Type | Description |
|--------|------|-------------|
| Internal Part Numbers | string | AAP's internal SKU (e.g. AAP-BRAKE-PAD-F150) |
| Vendor Interchange | string | Vendor's part number (e.g. BMX-BP-F150-FR) — used for packing slip lookup |
| Description | string | Human-readable part description |
| Qty On Hand | integer | Current stock count — updated on every transaction |
| Reorder Point | integer | Triggers low-stock email alert when Qty On Hand falls below this |
| Last Updated | timestamp | When the row was last modified by the automation |

### Sample data

| Internal Part # | Vendor Interchange | Description | Qty On Hand | Reorder Point | Last Updated |
|----------------|-------------------|-------------|-------------|---------------|--------------|
| AAP-BRAKE-PAD-F150 | BMX-BP-F150-FR | Front Brake Pads (Ford F-150) | 44 | 10 | 10/26/2025 9:42 PM |
| AAP-OIL-FILTER-STD | FTS-OF-2840 | Standard Oil Filter | 272 | 75 | 10/26/2025 9:49 PM |
| AAP-COOLANT-1GAL | FTS-AF-1000 | Coolant/Antifreeze (1 Gallon) | 73 | 30 | 10/26/2025 9:49 PM |
| AAP-BRAKE-ROTOR-F150 | BMX-BR-F150-FR | Front Brake Rotors (Ford F-150) | 21 | 6 | 10/26/2025 9:42 PM |
| AAP-ALT-12V-150A | PSE-ALT-150A | 12V Alternator (150 Amp) | 32 | 5 | 10/26/2025 9:50 PM |
| AAP-STARTER-12V | PSE-STR-12V-HD | 12V Starter Motor | 22 | 4 | 10/26/2025 9:50 PM |

---

## Transaction_Log

Append-only record of every document processed by the automation.

| Column | Type | Description |
|--------|------|-------------|
| Timestamp | datetime | When the transaction was processed |
| Document Type | string | `pick_ticket` or `packing_slip` |
| Vendor/Customer | string | Company name from the document |
| Part Number | string | Internal AAP part number (after interchange lookup) |
| Description | string | Part description |
| Qty Change | integer | Negative = deduction (pick ticket), Positive = addition (packing slip) |
| Previous Balance | integer | Qty On Hand before this transaction |
| New Balance | integer | Qty On Hand after this transaction |
| Document Name | string | Source filename in Google Drive |
| Confidence Score | float | Gemini extraction confidence (0.0–1.0) |

### Sample data

| Timestamp | Doc Type | Vendor/Customer | Part # | Qty Change | Prev Bal | New Bal | Confidence |
|-----------|----------|----------------|--------|------------|----------|---------|------------|
| 26-10-2025 21:37:28 | pick_ticket | ABC Auto Repair | AAP-BRAKE-PAD-F150 | -10 | 24 | 14 | 1 |
| 26-10-2025 21:38:23 | packing_slip | FluidTech Supply Co | FTS-AF-1000 | 50 | 25 | 75 | 1 |
| 26-10-2025 21:42:00 | packing_slip | BrakeMax Distributors | BMX-BP-F150-FR | 30 | 14 | 44 | 1 |
| 26-10-2025 21:40:30 | pick_ticket | Express Auto Service | AAP-STARTER-12V | -4 | 11 | 7 | 1 |

---

## Review_Queue

Documents the automation could not process with sufficient confidence (score < 0.7). Requires human review.

| Column | Type | Description |
|--------|------|-------------|
| Time Stamp | datetime | When the low-confidence extraction occurred |
| Document_Name | string | Source filename in Google Drive |
| Issue | string | Description of the problem (e.g. "LOW CONFIDENCE EXTRACTION - Score: 0.6") |
| Status | string | `Pending Review` or `Resolved` |

Row is highlighted red in Google Sheets to make pending items immediately visible.

### Sample data

| Time Stamp | Document_Name | Issue | Status |
|------------|--------------|-------|--------|
| 26-10-2025 21:54:37 | ZPIC2-IndustrialFasteners_PackingSlip_Sept21.png | LOW CONFIDENCE EXTRACTION - Score: 0.6 | Pending Review |

When a document lands here, the automation simultaneously sends an email alert to the warehouse manager with the document name and confidence score.

---

## Key design notes

**Vendor interchange mapping:** Pick tickets use internal AAP part numbers directly. Packing slips use vendor part numbers — the workflow looks up `Vendor_Interchange` in Master_Inventory to resolve to the internal part number before recording the transaction.

**Sign convention:** Pick tickets (outbound) → negative `Qty Change`. Packing slips (inbound) → positive `Qty Change`. New Balance = Previous Balance + Qty Change.

**Confidence threshold:** 0.7. Above = auto-process into Transaction_Log + update Master_Inventory. Below = append to Review_Queue + send email alert. No partial processing on low-confidence documents.
