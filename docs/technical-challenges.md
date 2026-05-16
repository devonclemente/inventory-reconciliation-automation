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

## 2. Document Extractor Breaks the Confidence System

**The symptom:** Testing with deliberately degraded, hard-to-read documents — faded text, low contrast, poor scan quality — was returning confidence scores of 1.0. Every document was "perfectly clear" regardless of actual readability.

**The cause:** This is a fundamental architectural issue, not a bug. Make's AI Content Extractor runs OCR on the file *before* passing anything to Gemini. It converts the image to clean text first. By the time Gemini sees it, the faded image is already clean characters — the visual quality information is gone. The computer could read it perfectly no matter what it looked like.

A confidence score based on clean extracted text is meaningless for quality control. If OCR succeeds, confidence is always high. The entire low-confidence routing path was useless.

**The fix:** Switched from document mode to image mode. Instead of letting Make pre-process the file, the workflow passes the raw image directly to Gemini Vision. Gemini now sees the same pixels a human would see. A faded, hard-to-read document looks faded and hard to read — and gets a low confidence score. A clean document gets a high one.

This is what makes the quality control path actually work. The Industrial Fasteners packing slip (intentionally degraded as a test case) scores 0.6 in image mode. In document mode it would have scored 1.0 and been processed silently with bad data.

**Lesson:** If your system needs to distinguish document quality, you must pass the visual information — not a cleaned-up version of it. The abstraction that makes things easier is the same abstraction that destroys the signal you need.

---

## 3. Duplicate Email Alerts

**The symptom:** A single packing slip could trigger 8 separate low-stock alert emails — one per line item that fell below reorder threshold.

**The cause:** The Iterator module unpacks the array of extracted items and processes them one at a time. The Gmail module was inside the iterator loop. Each iteration that hit the low-stock condition sent its own email.

**The fix:** Added an Array Aggregator module between the Iterator and the Gmail step. The aggregator collects all flagged items across the full document run into a single array. Then one Gmail module sends one email listing all flagged parts together.

**Lesson:** In Make.com, an iterator loop processes each item independently. If you want to act on a set, aggregate first.
