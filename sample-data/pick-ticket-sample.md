# Sample Pick Ticket Format

**Company:** Automation Auto Parts (fictional — modeled on real warehouse operations)

---

```
AUTOMATION AUTO PARTS
Pick Ticket #: PT-2025-0847
Date: January 15, 2025

Customer: Metro Fleet Services
Order #: ORD-10293

Items to Pick:
-----------------------------------------
Part #    | Description           | Qty
-----------------------------------------
AB-1042   | Air Brake Valve       | 2
FS-0310   | Fuel Sensor Kit       | 1
-----------------------------------------

Picked by: ______________________
Time: ______________________
```

---

The automation extracts: `part_number`, `quantity`, and `document_type: "pick_ticket"` from documents in this format. Documents may be handwritten, printed, or low-quality scans — Gemini Vision handles all three.
