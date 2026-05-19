# Roampal Mobile Architecture

## Overview

Roampal Mobile extends the Google AI Edge Gallery with a persistent memory layer that implements the Roampal Core sidecar pattern entirely on-device. Every conversation is stored, embedded, scored, and retrieved locally -- no server required.

This document describes the intended architecture based on verified facts from the upstream Google AI Edge Gallery codebase and the Roampal Core reference implementation. Unverified or estimated values are explicitly marked as such.

---

## Verified Facts: Upstream Codebase

### Google AI Edge Gallery (Base)

| Property | Verified Value |
|----------|----------------|
| License | Apache 2.0 |
| Language | Kotlin (~40% of files; remainder XML, JSON, resources) |
| Stars | 23,095 |
| Runtime | LiteRT LM (`com.google.ai.edge.litertlm`) |
| minSdk | 31 (Android 12) |
| applicationId | `com.google.aiedge.gallery` |
| System prompt support | Yes (`systemInstruction` in `ConversationConfig`) |
| Hardware detection | Yes (`minDeviceMemoryInGb`, `estimatedPeakMemoryInBytes`) |
| iOS | Listed in README (App Store badge, iOS 17+). No iOS source code in this repo. |
| MCP SDK | `mcp.kotlin.sdk` dependency present. Usage not yet verified. |

### Models in Official Allowlist

| Model | File Size | Peak Memory | Config |
|-------|-----------|-------------|--------|
| Gemma-3n-E2B-it-int4 | 2.9 GB | 5.9 GB | topK=64, topP=0.95, temp=1.0, maxTokens=4096, accelerators=cpu,gpu |
| Gemma-3n-E4B-it-int4 | 4.1 GB | 7.0 GB | Same as above |
| Gemma3-1B-IT q4 | 529 MB | 2.0 GB | topK=64, topP=0.95, temp=1.0, maxTokens=1024, accelerators=gpu,cpu |
| Qwen2.5-1.5B-Instruct q8 | 1.5 GB | 2.5 GB | topK=40, topP=0.95, temp=1.0, maxTokens=1024, accelerators=cpu |

**Note**: The allowlist currently contains **Gemma-3n** models, not Gemma-4. Gemma-4 checkpoints exist on HuggingFace but are not yet included in the Gallery's official allowlist.

### Roampal Core (Reference Implementation)

| Property | Verified Value |
|----------|----------------|
| Embedding model | `paraphrase-multilingual-mpnet-base-v2` (ONNX O4, 768d) |
| Reranker | `cross-encoder/mmarco-mMiniLMv2-L12-H384-v1` (ONNX O4) |
| Tag extraction | **LLM-only** (`tag_service.py:534`: "No regex fallback") |
| Wilson score | **Metadata only** (`search_service.py:137-138`: "Raw CE score as final ranking. No Wilson blend.") |
| Retrieval | TagCascade: tag overlap counting + cosine fill + cross-encoder rerank |
| Vector store | ChromaDB (local embedded) |
| Scoring | Outcome-based deltas: worked +0.2, failed -0.3, partial +0.05 |

---

## Memory Layer Stack (Proposed)

| Component | Technology | Verified Size | Source |
|-----------|-----------|---------------|--------|
| **Embedding** | model2vec potion-base-4M (ONNX) | 14.4 MB | HF API verified |
| **Reranker** | cross-encoder MiniLM-L6 int8 ARM64 (ONNX) | 22.1 MB | HF API verified |
| **Tokenizer** | HuggingFace tokenizers (JSON) | 684 KB | HF API verified |
| **Vector store** | SQLite (built-in) | 0 MB | Android system |
| **Total assets** | | ~37 MB | |

**Unknown**: Whether `Sentence-Embeddings-Android` library directly supports cross-encoder inference. The library README documents sentence embedding models only. Cross-encoder support may require direct ONNX Runtime usage or a different integration path.

**Unknown**: Actual inference latency for model2vec and cross-encoder on target ARM64 devices. No benchmarks have been run.

---

## Memory Storage (SQLite Schema)

Proposed schema based on Roampal Core's data model, adapted for SQLite:

```sql
CREATE TABLE memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    doc_id TEXT UNIQUE NOT NULL,
    collection TEXT NOT NULL,              -- working | history | patterns | memory_bank
    content TEXT NOT NULL,                 -- Primary text field
    text TEXT,                             -- Fallback/alternative text field (Roampal Core stores both)
    embedding BLOB NOT NULL,               -- 256 float32s
    noun_tags TEXT,                        -- JSON array
    memory_type TEXT,                      -- summary | fact | exchange_summary
    source TEXT,
    raw_score REAL DEFAULT 0.5,
    wilson_score REAL DEFAULT 0.5,
    success_count REAL DEFAULT 0.0,        -- Float: worked=1.0, partial=0.5
    uses INTEGER DEFAULT 0,
    outcome_history TEXT,                  -- JSON array
    created_at INTEGER NOT NULL,           -- Unix timestamp
    status TEXT DEFAULT 'active',
    importance REAL DEFAULT 0.7,
    confidence REAL DEFAULT 0.7,
    conversation_id TEXT,
    metadata TEXT                          -- Escape hatch for extensibility
);

CREATE TABLE tag_index (
    tag TEXT NOT NULL,
    doc_id TEXT NOT NULL,
    collection TEXT NOT NULL,
    PRIMARY KEY (tag, doc_id)
);

CREATE TABLE completion_state (
    conversation_id TEXT PRIMARY KEY,
    completed INTEGER DEFAULT 0,
    scoring_required INTEGER DEFAULT 0,
    scored_this_turn INTEGER DEFAULT 0,
    first_message_seen INTEGER DEFAULT 0,
    timestamp INTEGER NOT NULL
);
```

---

## Retrieval Flow (Based on Roampal Core)

### Step 1: Query Encoding
Encode the user's message using the embedding model.

### Step 2: Tag Routing
Extract noun tags from the query and match against the known tag index. Count tag overlaps per memory.

**Note**: Roampal Core uses LLM-only tag extraction (no regex fallback). On mobile, this approach is too slow. A NEW method is required -- this is not from Roampal Core:
- Heuristic extraction (faster, lower quality -- a mobile-specific addition)
- Storing tags at write-time and skipping extraction at read-time
- Tiny ONNX classifier for noun phrase extraction

### Step 3: Candidate Pool
Build a pool of candidate memories:
- Fill from highest tag-overlap tier down
- Sort by cosine distance within each tier
- Fill remaining slots with pure cosine similarity search

### Step 4: Cross-Encoder Rerank
Score each query-document pair with the cross-encoder. Sort by raw cross-encoder score.

**Verified**: Wilson score is NOT blended into the ranking. It is stored as metadata for display only.

### Step 5: Two-Lane Merge
- Lane 1: Top summaries/context memories (excluding facts)
- Lane 2: Top facts
- Merge and format for injection

### Step 6: Prompt Injection
Prepend formatted memories to the system prompt via LiteRT LM's `systemInstruction` parameter.

---

## Scoring Flow (Based on Roampal Core)

### After LLM Response (STOP hook)
1. Store the exchange in the `working` collection
2. Embed and store any extracted facts
3. Mark the conversation as completed in `completion_state`

### On Next User Message (Implicit Feedback)
1. Retrieve the previous exchange for this conversation
2. Classify the followup as outcome: worked / failed / partial / unknown
3. Update the Wilson score and raw score for each memory retrieved in the previous turn
4. Apply promotion/demotion rules

### Score Deltas (Verified from `outcome_service.py`)
| Outcome | Delta |
|---------|-------|
| worked | +0.2 |
| partial | +0.05 |
| unknown | -0.05 |
| failed | -0.3 |

---

## Memory Lifecycle (Based on Roampal Core)

```
    working (24h TTL)
    ↓ score >= 0.7 AND uses >= 2
history (30d TTL)
    ↓ score >= 0.9 AND uses >= 3 AND success_count >= 5
patterns (permanent)

    working: score < 0.2 OR age > 24h → DELETE
    history: score < 0.4 → demote or delete
    patterns: score < 0.4 → demote to history

**Note**: `success_count` resets to 0 on promotion to history. A memory must re-prove itself in the new tier before it can reach patterns.
```

---

## Model Tiering by Hardware

The Gallery app already filters models by `minDeviceMemoryInGb` and shows memory warnings. We extend this logic:

| Device RAM | LLM Options | Memory Layer (Estimated) | Total Peak (Estimated) |
|------------|-------------|--------------------------|------------------------|
| 12GB+ | Gemma-3n-E4B (7.0GB peak) | ~0.1GB | ~7.1GB |
| 8-12GB | Gemma-3n-E2B (5.9GB peak) | ~0.1GB | ~6.0GB |
| 6-8GB | Gemma3-1B (2.0GB peak) | ~0.1GB | ~2.1GB |
| 4-6GB | Qwen2.5-1.5B (2.5GB peak) | ~0.1GB | ~2.6GB |

**Note**: Memory layer RAM usage is an estimate -- actual usage depends on SQLite cache, embedding buffers, and concurrency. The E4B model alone peaks at 7GB. Embedding and reranker run sequentially (not simultaneously with the LLM), which reduces peak pressure.

---

## File Structure (Proposed New Modules)

```
Android/src/app/src/main/java/com/roampal/mobile/
├── memory/
│   ├── MemoryService.kt
│   ├── embedding/
│   │   └── Model2VecEmbedder.kt
│   ├── reranker/
│   │   └── CrossEncoder.kt
│   ├── storage/
│   │   ├── MemoryDatabase.kt
│   │   └── MemoryDao.kt
│   ├── retrieval/
│   │   └── TagCascadeRetriever.kt
│   ├── scoring/
│   │   ├── WilsonScore.kt
│   │   └── ScoringService.kt
│   └── models/
│       └── Memory.kt
└── hooks/
    ├── GetContextHook.kt
    └── StopHook.kt

assets/models/
├── model2vec/
│   ├── model.onnx
│   └── tokenizer.json
└── cross_encoder/
    ├── model_qint8_arm64.onnx
    └── tokenizer.json
```

---

## Integration Points (Known)

### 1. Pre-LLM Hook
Insert before `model.runtimeHelper.runInference()` in `LlmChatViewModel.generateResponse()`.

**Unknown**: Whether resetting the conversation with a new `systemInstruction` on every message has side effects on the LLM's internal state or performance.

### 2. Post-LLM Hook
Insert in the result listener's `onDone()` callback.

### 3. Followup Scoring
Insert at the start of the next `generateResponse()` call.

---

## Dependencies (Proposed)

```kotlin
// In Android/src/app/build.gradle.kts

dependencies {
    // Existing Gallery dependencies...
    
    // Roampal Memory Layer
    // NOTE: sentence-embeddings supports embedding models.
    // Cross-encoder support is UNVERIFIED -- may need direct ONNX Runtime.
    implementation("io.gitlab.shubham0204:sentence-embeddings:v6.1")
    implementation("io.gitlab.shubham0204:model2vec:v6")
    
    // JSON parsing
    implementation("com.google.code.gson:gson:2.10.1")
}
```

---

## Open Questions / Unknowns

1. **Tag extraction on mobile**: Roampal Core uses LLM calls. On mobile, we need a method that doesn't require a second LLM inference. Options: heuristic, tiny classifier, or skip extraction and rely on cosine-only.

2. **Cross-encoder integration**: Whether `Sentence-Embeddings-Android` supports cross-encoder models or if we need direct ONNX Runtime Java API.

3. **System prompt reset cost**: Resetting `ConversationConfig.systemInstruction` before every message may add initialization overhead. Needs measurement.

4. **SQLite vector search**: SQLite has no native cosine similarity. Options: custom UDF, brute-force in Kotlin (fine for <1000 memories), or `sqlite-vss` extension.

5. **Memory cleanup scheduling**: When to run promotion/demotion jobs? App backgrounding? Periodic WorkManager? On-demand during retrieval?

6. **iOS architecture**: This repo has no iOS code. A separate Swift implementation would be needed.

7. **Gemma-4 models**: Not in the official allowlist. Adding them requires updating `model_allowlist.json` with community LiteRT conversions (e.g., `litert-community/gemma-4-E2B-it-litert-lm`).

---

## References

- [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery) (upstream)
- [Roampal Core](https://github.com/roampal/roampal-core) (reference implementation)
- [model2vec](https://huggingface.co/minishlab/potion-base-4M) (embedding model)
- [cross-encoder MiniLM-L6](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L6-v2) (reranker)
- [Sentence-Embeddings-Android](https://github.com/shubham0204/Sentence-Embeddings-Android)
