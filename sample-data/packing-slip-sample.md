# Sample Packing Slip

Source company: **BrakeMax Distributors, LLC** (fictional vendor — ships inbound stock to Automation Auto Parts)

---

```
BRAKEMAX DISTRIBUTORS, LLC
8925 Commerce Drive, Detroit, MI 48201
Phone: (313) 555-8200  |  sales@brakemax.com

DELIVERY PACKING SLIP
Document No: BMX-PS-91847          Processed: September 18, 2025

SHIPPING ADDRESS                   ORDER DETAILS
Automation Auto Parts              Customer PO: PO-AAP-2025-0918
321 Automation Blvd                BrakeMax Order: BMX-918447
New York, NY 10001                 Sales Rep: Maria Rodriguez
ATTN: Warehouse Manager            Payment Terms: Net 30
                                   Account #: AAP-3982

SHIPMENT INFORMATION
Ship Date: September 18, 2025  |  Ship Via: UPS Ground
Tracking: 1Z847291AB9284710
Number of Boxes: 3  |  Total Weight: 142 lbs  |  Freight: Prepaid

PART NUMBER       DESCRIPTION                           QTY ORD  QTY SHPD
--------------------------------------------------------------------------
BMX-BP-F150-FR    Premium Ceramic Brake Pad Set - Front    30       30
BMX-BR-F150-FR    Performance Brake Rotor - Front (Pair)   15       15

Total Line Items: 2
Total Units Shipped: 45
ORDER STATUS: COMPLETE
```

---

## What the automation extracts

From a packing slip, Gemini extracts each line item using the **vendor's part number**:

```json
{
  "document_type": "packing_slip",
  "part_number": "BMX-BP-F150-FR",
  "quantity": 30,
  "confidence": 0.95
}
```

## Vendor interchange lookup

Packing slips use **vendor part numbers** (BMX-BP-F150-FR), not internal AAP numbers. The workflow looks up the vendor part number in the `Vendor_Interchange` column of Master_Inventory to find the internal AAP part number (AAP-BRAKE-PAD-F150) before updating stock.

| Vendor Part # | AAP Internal Part # |
|---------------|---------------------|
| BMX-BP-F150-FR | AAP-BRAKE-PAD-F150 |
| BMX-BR-F150-FR | AAP-BRAKE-ROTOR-F150 |

Packing slips add quantity (positive `qty_change`) — incoming stock received.
