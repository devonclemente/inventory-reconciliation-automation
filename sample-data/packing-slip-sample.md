# Sample Packing Slip Format

**Company:** Automation Auto Parts (fictional — modeled on real warehouse operations)

---

```
AUTOMATION AUTO PARTS
Packing Slip #: PS-2025-1124
Date: January 15, 2025

Ship To: Metro Fleet Services
         412 Industrial Blvd
         Newark, NJ 07101

Items Shipped:
-----------------------------------------
Part #    | Description           | Qty
-----------------------------------------
AB-1042   | Air Brake Valve       | 2
FS-0310   | Fuel Sensor Kit       | 1
HL-2210   | Headlamp Assembly     | 4
-----------------------------------------

Shipped by: ______________________
Carrier: UPS Ground
Tracking: 1Z999AA10123456784
```

---

The automation extracts: `part_number`, `quantity`, and `document_type: "packing_slip"` from documents in this format. Packing slips trigger outbound inventory deductions in the Master_Inventory sheet.
