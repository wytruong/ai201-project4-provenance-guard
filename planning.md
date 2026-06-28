# Provenance Guard — Planning Document

## Architecture Narrative

A submitted piece of text enters the system at POST /submit. It is passed through two independent detection signals: an LLM classifier (Groq) and a stylometric heuristics function. Each signal returns a score between 0.0 and 1.0. A confidence scorer combines them into a single score using a weighted average (LLM 60%, stylometrics 40%). That score maps to one of three transparency labels, which is returned to the caller along with the result. Every submission is written to a structured audit log. Appeals enter at POST /appeal, update the original log entry's status to "under_review", and append the creator's reasoning.

## Architecture

SUBMISSION FLOW
POST /submit

|
v

[Signal 1: Groq LLM]  -->  llm_score (0.0–1.0)

[Signal 2: Stylometrics]  -->  stylo_score (0.0–1.0)
|
v

[Confidence Scorer]

|  confidence = (llm_score * 0.6) + (stylo_score * 0.4)
|
v

[Label Generator]

|  >= 0.75  --> "Likely AI-generated"

|  <= 0.40  --> "Likely human-written"

|  else     --> "Uncertain"

|
v

[Audit Log]  <-- writes: content_id, creator_id, timestamp,

|              attribution, confidence, llm_score, stylo_score, status
v

Response: { content_id, attribution, confidence, label, llm_score, stylo_score }
APPEAL FLOW
POST /appeal

|
v

[Look up content_id in audit log]

|

[Update status --> "under_review"]

[Append appeal_reasoning + appeal_timestamp]

|
v

Response: { message, content_id, status }

## Detection Signals

### Signal 1: LLM Classification (Groq)
- What it measures: Semantic and stylistic coherence. Asks whether the text reads as AI-generated based on tone, structure, and phrasing patterns.
- Output: Float between 0.0 (definitely human) and 1.0 (definitely AI).
- Blind spot: Can flag formal human writing as AI. Can be fooled by lightly edited AI output.

### Signal 2: Stylometric Heuristics
- What it measures: Statistical structural properties including sentence length variance, type-token ratio, and punctuation density.
- Output: Float between 0.0 and 1.0. Low variance + low TTR + low punctuation = higher AI probability.
- Blind spot: Short texts lack enough signal. Formal human writing looks structurally similar to AI.

## Uncertainty Representation

- Confidence >= 0.75: Likely AI-generated
- Confidence <= 0.40: Likely human-written
- Confidence 0.41-0.74: Uncertain

A score of 0.6 means both signals lean toward AI but neither is confident enough to make a strong call. The threshold is intentionally conservative on the AI side because false positives are worse than false negatives on a creative platform.

## Transparency Label Variants

High-confidence AI:
"Likely AI-generated. This content was flagged as probably created by an AI writing tool. Confidence: High."

High-confidence human:
"Likely human-written. This content appears to have been written by a person. Confidence: High."

Uncertain:
"Uncertain. Our system could not confidently determine whether this content was written by a human or AI. The author may appeal if this label is incorrect."

## Appeals Workflow

- Any creator can submit an appeal via POST /appeal
- They provide their content_id and creator_reasoning
- The system updates the entry status to "under_review" and appends the reasoning and timestamp to the audit log
- A human reviewer would see the original classification, both signal scores, and the creator's reasoning

## Anticipated Edge Cases

1. Non-native English speakers: Simpler vocabulary and shorter, more uniform sentences may score high on stylometrics even though the content is human-written.
2. Heavily edited AI output: If a human significantly rewrites AI-generated text, the LLM signal may not catch it and stylometrics will reflect the human edits.

## AI Tool Plan

### M3 - Submission endpoint and first signal
- Provide: detection signals section + architecture diagram
- Ask for: Flask app skeleton + signal_llm function
- Verify: call the function directly with test inputs before wiring into the endpoint

### M4 - Second signal and confidence scoring
- Provide: detection signals section + uncertainty representation + diagram
- Ask for: signal_stylometric function + scoring logic
- Verify: scores vary meaningfully between clearly AI and clearly human text

### M5 - Production layer
- Provide: label variants + appeals workflow + diagram
- Ask for: generate_label function + POST /appeal endpoint
- Verify: all three label variants are reachable, appeal updates status correctly