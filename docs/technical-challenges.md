# Technical Challenges

Three bugs that each looked like they should work until they didn't.

## 1. JSON Parsing Failures

**The symptom:** The Make.com JSON parse module was failing intermittently. Not always — just enough to break the workflow without an obvious pattern.

**The cause:** Make's built-in AI Content Extractor prompted Gemini to return structured data, but Gemini kept adding conversational text before or after the JSON block. Something like:

```
Here is the extracted data from the document:

{"document_type": "pick_ticket", "part_number": "AB-1042", ...}

Let me know if you need anything else.
```

The JSON parser expects raw JSON. It saw a string and failed.

**The fix:** Took control of the Gemini prompt directly. Added "Return ONLY valid JSON. No explanation, no markdown." and included a hardcoded example of the exact expected format. Pass rate went from ~60% to ~97%.

**Lesson:** When using LLMs for structured output, be explicit to the point of being patronizing. Show it exactly what you want.

---

## 2. Confidence Scoring Unreliable on Scanned Documents

**The symptom:** The confidence score on the same document varied between runs. A scan that came back 0.91 one run would return 0.63 the next. The 0.7 threshold was routing documents inconsistently.

**The cause:** Make's AI Content Extractor pre-processes files before sending them to Gemini — it applies its own document parsing layer. On grayscale warehouse scans, that pre-processing was introducing noise.

**The fix:** Switched to image mode. Instead of treating the file as a "document" through Make's extractor, the workflow now passes the file as a raw image directly to the Gemini Vision API. Gemini assesses the image without Make's pre-processing layer. Confidence scores became stable and consistent.

**Lesson:** When a module abstracts something you care about, sometimes you have to go one level below it.

---

## 3. Duplicate Email Alerts

**The symptom:** A single packing slip could trigger 8 separate low-stock alert emails — one per line item that fell below reorder threshold.

**The cause:** The Iterator module unpacks the array of extracted items and processes them one at a time. The Gmail module was inside the iterator loop. Each iteration that hit the low-stock condition sent its own email.

**The fix:** Added an Array Aggregator module between the Iterator and the Gmail step. The aggregator collects all flagged items across the full document run into a single array. Then one Gmail module sends one email listing all flagged parts together.

**Lesson:** In Make.com, an iterator loop processes each item independently. If you want to act on a set, aggregate first.
