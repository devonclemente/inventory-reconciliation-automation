# Screenshots

Visual artifacts of the live Make.com workflow.

## What to capture

### 1. Full workflow overview
Zoom out in Make.com to show the entire scenario — even if modules are small. The goal is to show the scale (40+ modules) and the overall flow direction.

**Filename:** `workflow-full.png`

### 2. Document intake + Gemini step
The Google Drive watch module → download → AI Content Extractor → Gemini Vision API chain. Shows the AI processing entry point.

**Filename:** `workflow-intake-gemini.png`

### 3. Confidence router
The router module that splits high-confidence (≥0.7) vs low-confidence (<0.7) paths. This is the quality control decision point.

**Filename:** `workflow-confidence-router.png`

### 4. Google Sheets update
The iterator + router that splits pick_ticket vs packing_slip and updates the correct sheet.

**Filename:** `workflow-sheets-update.png`

### 5. Email alert
The Gmail module and/or a sample of the low-stock alert email.

**Filename:** `workflow-email-alert.png`

---

Add screenshots here and reference them in the root README.md.
