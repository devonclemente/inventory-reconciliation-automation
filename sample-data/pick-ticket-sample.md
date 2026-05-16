# Sample Pick Ticket

Source company: **Automation Auto Parts** (fictional — modeled on real warehouse operations)

---

```
AUTOMATION AUTO PARTS
321 Automation Blvd | New York, NY 10001 | (212) 555-9000

PICK TICKET
Order #: PT-2025-0923-001          Date: September 23, 2025
                                   Priority: STANDARD

CUSTOMER INFORMATION               SHIPPING ADDRESS
Company: ABC Auto Repair           ABC Auto Repair
Contact: Mike Anderson             458 Mechanics Row
Phone: (212) 555-3400              Brooklyn, NY 11201
Account #: CUST-1847               Delivery: Customer Pickup

AAP PART NUMBER       DESCRIPTION                  LOCATION  QTY
--------------------------------------------------------------------
AAP-BRAKE-PAD-F150    Front Brake Pads (F-150)     B-14-C     10
AAP-OIL-FILTER-STD    Standard Oil Filter          A-08-F     25
AAP-COOLANT-1GAL      Coolant/Antifreeze (1 Gal)   C-02-A     15

Total Line Items: 3
Total Units to Pick: 50
PICK STATUS: PENDING

PICKED BY ____________  VERIFIED BY ____________  SHIPPED BY ____________
```

---

## What the automation extracts

From a pick ticket, Gemini extracts each line item as:

```json
{
  "document_type": "pick_ticket",
  "part_number": "AAP-BRAKE-PAD-F150",
  "quantity": 10,
  "confidence": 0.97
}
```

Pick tickets use **internal AAP part numbers** (AAP-BRAKE-PAD-F150). The workflow deducts quantity from Master_Inventory directly — no interchange lookup needed.

Each line item is processed as a separate transaction in Transaction_Log with a negative `qty_change` (deduction).
