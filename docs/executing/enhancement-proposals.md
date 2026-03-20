# Photon Enhancement Proposals

> Prepared 2026-03-19 after a thorough review of the full codebase (~12.4K lines, 226 tests, 10 benchmarks).

---

## Context

Photon is already a well-architected, production-grade image processing pipeline. The sequential pipeline is clean, async-first, and well-tested. The tagging subsystem is sophisticated (three-pool relevance tracking, progressive encoding, hierarchy dedup). The codebase is disciplined — good error types, sensible defaults, and clean module boundaries.

The proposals below are not bug fixes or polish. They are **capability expansions** that leverage Photon's existing infrastructure to unlock new use cases and close the gap between what Photon *computes* and what users can *do* with it.

---

## Proposal 1: Semantic Search (`photon search`)

### The Gap

Photon's stated vision is: *"a user should be able to type in 'fashion' and then a photo of red sneakers should show up."* Today, Photon computes everything needed to make this work — 768-dim SigLIP image embeddings and a SigLIP text encoder — but stops at JSON output. The user must build their own vector search infrastructure to actually *find* anything.

### The Idea

Add a `photon search` command that performs text-to-image retrieval against previously processed output:

```bash
# Index a processed JSONL file (builds a local vector index)
photon index results.jsonl --index-path ./photos.idx

# Search by natural language
photon search "red sneakers on white background" --index-path ./photos.idx --top-k 10

# Search by image similarity (image-to-image)
photon search --image reference.jpg --index-path ./photos.idx --top-k 10

# Search by tag
photon search --tag "sunset" --index-path ./photos.idx
```

### Why This Is Feasible

Photon already has every building block:

1. **Image embeddings** — `EmbeddingEngine` produces 768-dim L2-normalized vectors (dot product = cosine similarity).
2. **Text encoder** — `SigLipTextEncoder` in the tagging subsystem encodes text to the same 768-dim space. It's currently used to encode vocabulary terms, but works with arbitrary text.
3. **SigLIP cross-modal alignment** — The whole point of SigLIP is that image and text embeddings live in the same space. `dot(text_embed, image_embed)` gives semantic similarity.
4. **Confidence scoring** — The learned sigmoid scaling (`117.33 * cosine - 12.93`) already converts raw cosines to calibrated [0, 1] confidence scores.

What's missing is only:
- A lightweight **vector index** (flat brute-force for <100K images; HNSW via `instant-distance` or `hnsw` crate for scale)
- A thin **CLI subcommand** that loads the index, encodes the query, and returns top-K results
- An `index` subcommand that reads JSONL embeddings into the index format

### Scope Estimate

- **Index format**: Flat file of `(file_path, embedding)` pairs, memory-mapped. For <100K images, brute-force cosine scan is <50ms.
- **Text encoding**: Reuse `SigLipTextEncoder` directly. Already handles tokenization, ONNX inference, L2 normalization.
- **Image-to-image**: Run query image through `EmbeddingEngine`, then same cosine search.
- **Output**: Ranked list of `{ path, score, tags }` — reuse existing `OutputWriter`.

### Impact

This would make Photon a **complete image retrieval system** — not just a processing pipeline. Users could process a photo library once and then search it instantly, without any external infrastructure. This directly fulfills the original vision document.

---

## Proposal 2: Near-Duplicate & Similarity Detection (`photon dedup`)

### The Gap

Photon computes three independent similarity signals per image — BLAKE3 content hash, perceptual hash (pHash), and 768-dim embedding vector — but never uses them for comparison. Users processing large libraries have no way to find duplicates, near-duplicates, or visually similar images.

### The Idea

Add a `photon dedup` command that finds duplicate and near-duplicate images:

```bash
# Find exact duplicates (BLAKE3 hash match)
photon dedup results.jsonl --mode exact

# Find near-duplicates (perceptual hash distance < threshold)
photon dedup results.jsonl --mode perceptual --threshold 10

# Find visually similar images (embedding cosine > threshold)
photon dedup results.jsonl --mode semantic --threshold 0.85

# Combined: cascade from fast to slow (hash → pHash → embedding)
photon dedup results.jsonl --mode cascade

# Output: groups of similar images with similarity scores
```

### Why This Is Feasible

All three similarity primitives already exist:

| Signal | Already Computed | Comparison Cost | Finds |
|--------|-----------------|----------------|-------|
| BLAKE3 hash | `ProcessedImage::content_hash` | O(1) string equality | Exact byte-identical copies |
| Perceptual hash | `ProcessedImage::perceptual_hash` | O(1) Hamming distance | Resized, recompressed, slightly cropped copies |
| Embedding vector | `ProcessedImage::embedding` | O(768) dot product | Visually similar but different images |

The **cascade mode** is the compelling feature: first group by BLAKE3 (instant), then compare pHash within non-matching groups (fast), then compare embeddings for remaining (accurate). This gives users a single command that handles everything from exact copies to "same subject, different angle."

### Output Format

```jsonl
{"group_id": 1, "type": "exact", "images": [{"path": "a.jpg", "hash": "abc..."}, {"path": "b.jpg", "hash": "abc..."}]}
{"group_id": 2, "type": "perceptual", "similarity": 0.94, "images": [{"path": "c.jpg"}, {"path": "d.jpg"}]}
{"group_id": 3, "type": "semantic", "similarity": 0.87, "images": [{"path": "e.jpg"}, {"path": "f.jpg"}, {"path": "g.jpg"}]}
```

### Impact

Deduplication is one of the most requested features in any photo management tool. Photon already does all the expensive computation — this proposal just adds the comparison layer. The cascade approach (fast exact check → medium perceptual check → expensive semantic check) is both correct and performant.

---

## Proposal 3: Watch Mode (`photon watch`)

### The Gap

Photon is currently a one-shot batch tool. For teams with continuous incoming assets (e.g., marketing teams uploading to shared drives, e-commerce product photography pipelines), users must manually re-run Photon or set up external cron jobs.

### The Idea

Add a `photon watch` command that monitors directories and processes new/changed images automatically:

```bash
# Watch a directory, process new images, append to output
photon watch ./incoming-photos --output results.jsonl --quality fast

# Watch with LLM enrichment
photon watch ./product-shots --output products.jsonl --llm anthropic

# Watch multiple directories
photon watch ./team-uploads ./client-assets --output combined.jsonl
```

### Implementation

- Use `notify` crate (cross-platform filesystem events) to detect new/modified files
- Deduplicate events with a debounce window (avoid processing partially-written files)
- Reuse existing `ImageProcessor` and `OutputWriter` — the pipeline is already async
- Track processed files via content hash to avoid re-processing renamed/moved files
- Append to JSONL output (natural for streaming)
- Graceful shutdown on SIGINT with in-progress flush

### Impact

This transforms Photon from a batch tool into a **continuous processing daemon** — suitable for integration into production asset pipelines. Combined with Proposal 1 (search), teams could have a self-updating searchable image library.

---

## Proposal 4: Output Field Selection & Compression

### The Gap

Photon's JSON output includes everything: embeddings (768 floats = ~3KB per image), thumbnails (base64 WebP = ~2KB), EXIF, tags, hashes. For many use cases, users only need a subset. There's no way to control output size or select specific fields.

### The Idea

```bash
# Only output tags and hashes (no embedding, no thumbnail, no EXIF)
photon process ./photos --fields tags,hash --output compact.jsonl

# Only embeddings (for feeding into external vector DB)
photon process ./photos --fields path,embedding --output vectors.jsonl

# Everything except thumbnails
photon process ./photos --exclude thumbnail --output no-thumbs.jsonl
```

### Implementation

Add a `FieldSelector` that wraps `ProcessedImage` serialization:

```rust
pub struct FieldSelector {
    include_embedding: bool,
    include_thumbnail: bool,
    include_exif: bool,
    include_tags: bool,
    include_hashes: bool,
    include_description: bool,
}
```

Apply the selector in `OutputWriter::write_record()` before serialization. Fields excluded by the selector are set to `None` (already `Option<T>` with `skip_serializing_if`), so no structural changes to `ProcessedImage` are needed.

**Pipeline optimization**: If a field is excluded, skip the corresponding pipeline stage entirely. `--fields tags,hash` should skip thumbnail generation and produce no embedding (unless tagging requires it).

### Impact

- **Smaller output**: Excluding embeddings and thumbnails reduces output by ~80% per image
- **Faster processing**: Skipping unnecessary stages saves CPU time
- **Better integration**: Users piping into specific backends (vector DB, metadata store) get exactly the shape they need

---

## Proposal 5: Structured LLM Extraction (Beyond Free-Text Descriptions)

### The Gap

Photon's current LLM integration generates free-text descriptions (`"A golden retriever sits on a sandy beach at sunset"`). This is useful but limited — the description can't be queried, filtered, or structured without NLP post-processing. The prompt is generic and doesn't adapt to image content.

### The Idea

Add structured extraction modes alongside free-text descriptions:

```bash
# Current behavior (free-text)
photon process ./photos --llm anthropic --llm-mode describe

# Structured extraction (JSON output with typed fields)
photon process ./photos --llm anthropic --llm-mode extract

# Custom schema (user-defined extraction template)
photon process ./photos --llm anthropic --llm-mode extract --llm-schema schema.json
```

**Default structured output:**

```json
{
  "enrichment": {
    "description": "A golden retriever sits on a sandy beach at sunset",
    "objects": ["dog", "beach", "ocean", "sand"],
    "colors": ["golden", "blue", "orange", "beige"],
    "scene": "outdoor",
    "mood": "calm",
    "text_content": null,
    "people_count": 0,
    "composition": "centered subject, rule of thirds horizon"
  }
}
```

**Custom schema example** (`schema.json`):

```json
{
  "product_name": "string",
  "product_category": "string",
  "background_type": "studio | lifestyle | outdoor",
  "brand_visible": "boolean",
  "text_on_image": "string | null"
}
```

### Implementation

- Modify `LlmRequest::describe_image()` to accept a mode parameter
- For `extract` mode, use a structured JSON prompt with the schema as instruction
- For `custom` mode, inject user's schema into the prompt and validate the LLM's JSON response
- Leverage existing tag-aware context (include zero-shot tags in extraction prompt for grounding)
- Add response validation: parse LLM JSON, warn on missing fields, retry on malformed output

### Why This Matters

The vision doc's use case is e-commerce/marketing: *"identify products in photos and tag them with product names, prices, and other relevant information."* Free-text descriptions can't do this. Structured extraction with custom schemas can. A marketing team could define their own schema (`product_line`, `campaign`, `season`, `target_audience`) and get queryable structured data from every image.

### Prompt Quality Improvement

The current prompt is generic:
> "Describe this image concisely in 1-3 sentences. Focus on the main subject, setting, and mood."

This should be improved even without structured extraction:
- **Tag-aware guidance**: If zero-shot tags include "product photography", steer toward product description. If tags include "landscape", steer toward scene description.
- **Confidence-weighted context**: Include high-confidence tags (>0.5) but not low-confidence noise
- **Few-shot examples**: Include 1-2 examples of good descriptions to anchor output quality

---

## Priority Ranking

| # | Proposal | Effort | Impact | Leverage |
|---|----------|--------|--------|----------|
| 1 | **Semantic Search** | Medium | Very High | Fulfills core vision; reuses text encoder + embeddings |
| 2 | **Near-Duplicate Detection** | Low | High | All primitives exist; just comparison logic |
| 4 | **Output Field Selection** | Low | Medium | Simple serialization filter; enables pipeline optimization |
| 5 | **Structured LLM Extraction** | Medium | High | Unlocks enterprise/e-commerce use case from vision doc |
| 3 | **Watch Mode** | Medium | Medium | Transforms batch tool into continuous pipeline |

Proposals 1 and 2 are recommended as the first priorities — they leverage existing computed data (embeddings, hashes) that currently goes unused after output, and they directly address the stated vision of making images "easily searchable and retrievable."

---

## Side Notes: Quick Wins From the Review

These are smaller improvements noticed during the review that don't warrant full proposals but are worth tracking:

1. **Silent lock poisoning** — `processor.rs` silently skips tagging when `RwLock` is poisoned. Should return an error or at minimum log a warning.
2. **Hardcoded timeouts** — `embed_timeout_ms` and `sweep_interval` are not exposed in config. Easy to add.
3. **Test gaps** — `ProgressiveEncoder`, `SigLipTextEncoder`, and `Enricher` have zero unit tests despite being critical paths. These are the 3 most complex untested modules.
4. **LLM prompt quality** — Current prompt is too generic. Even without Proposal 5, adding tag-aware guidance and few-shot examples would meaningfully improve description quality.
5. **No dry-run mode** — Users can't preview what would be processed without actually running (relevant for LLM cost estimation).
6. **No resume on batch failure** — If a 10K-image batch fails at image 5K, `--skip-existing` helps but requires the output file to be intact. A checkpoint mechanism would be more robust.

---

## Open Questions

1. **Search index format**: Should Photon use a purpose-built format (flat mmap'd vectors) or integrate an existing library like `usearch`, `instant-distance`, or `hnswlib`? The choice affects the scale ceiling — flat scan is fine for <100K images but HNSW is needed beyond that.

2. **Search as library vs. CLI-only**: Should `photon-core` expose search APIs (e.g., `SearchIndex::query()`) so downstream Rust applications can embed search, or is this CLI-only? This affects whether the index format needs to be stable/versioned.

3. **Watch mode daemon lifecycle**: Should `photon watch` be a foreground process (simple) or support backgrounding with a PID file (systemd/launchd integration)? The latter is more useful for production but significantly more complex.

4. **Structured extraction schema validation**: How strict should schema validation be? If the LLM returns extra fields or slightly different types, should Photon warn, error, or silently accept? Different users will have different tolerance.

5. **Dedup output actions**: Should `photon dedup` only *report* duplicates, or also offer `--action delete-duplicates` / `--action symlink`? Destructive actions are risky but highly requested. A `--dry-run` flag could mitigate.

6. **Field selection vs. pipeline skipping**: If a user requests `--fields tags,hash`, should Photon skip embedding entirely? Tagging depends on embedding, so `--fields tags` implicitly requires embedding even though it's excluded from output. The dependency graph needs to be documented.
