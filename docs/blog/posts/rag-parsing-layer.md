---
date: 2026-09-02
authors:
  - mdfarooque
categories:
  - RAG
  - LLMs
  - Data Engineering
description: Retrieval quality is decided at parse time. A practitioner's deep-dive into the parsing and ingestion layer of RAG, with the tools and trade-offs I use in production.
---

# Garbage In, Garbage Retrieved: Why Parsing Is the RAG Layer You're Underfunding

I have spent nine years building data and ML systems, and the last few of those buried in LLM and RAG pipelines on GCP and AWS. Here is the pattern I see at almost every enterprise: teams spend weeks A/B testing embedding models, tuning rerankers, and arguing about vector databases — and about 20 minutes on the code that turns a PDF into text. Then they wonder why the answers are wrong.

The uncomfortable truth: in most failing RAG systems I've audited, retrieval quality was capped long before the embedding model ever ran. It was capped at parse time. This article is a practitioner's deep-dive into the parsing/ingestion layer — the what, the why, and the how, with the tools and trade-offs I actually use in production.

<!-- more -->

!!! info "Originally published on LinkedIn"
    This article first appeared on [my LinkedIn](https://www.linkedin.com/pulse/garbage-retrieved-why-parsing-rag-layer-youre-md-farooque-eneaf/) on September 2, 2026.

## The Water-Treatment Analogy

Think of a RAG pipeline as a municipal water system. The LLM is the tap in your kitchen — shiny, visible, the thing everyone judges. Embeddings and retrieval are the pipes. But parsing is the **water-treatment plant**. If contaminated water enters the plant and treatment fails, no amount of polished chrome on the tap makes the water safe to drink. You just deliver contamination faster and with more confidence.

Parsing errors don't stay contained — they **propagate and amplify** downstream:

- A parser that jumbles two-column reading order produces sentences that splice across columns. That poisons **chunking** (semantic boundaries are now meaningless).
- Broken chunks produce **embeddings** that encode noise, not meaning. The vector sits in the wrong neighborhood of latent space.
- Retrieval returns the wrong chunk, or the right chunk mangled beyond use.
- The LLM, dutifully grounded, generates a confident answer from garbage — a hallucination you *caused* at ingestion.

This is why I call parsing the highest-leverage, most underinvested stage of RAG. A 5% improvement in parse fidelity often beats a 5% "better" embedding model, because parse errors are unrecoverable — downstream stages cannot reconstruct information that was destroyed at the source.

## The WHAT: Parsing Is Not "Extract The Text"

Naive parsing treats a document as a bag of characters. Real documents are **structured**: headings, reading order, tables, figures, footnotes, hierarchy, and metadata. The job of a parser is not merely to extract text — it is to **recover the structure and semantics a human reader would perceive**, and to preserve them into a representation your chunker and embedder can exploit.

Let me go modality by modality, because the failure modes are different for each.

## Plain Text, Markdown, HTML

The easy case — but easy to get subtly wrong. HTML is full of navigation, boilerplate, cookie banners, and scripts that dilute your embeddings with noise.

- trafilatura is my default for main-content extraction from web pages — it strips boilerplate well and preserves structure.
- BeautifulSoup for surgical, selector-based extraction when you know the DOM.
- html2text when you want to preserve semantic structure as Markdown.

**Why it matters:** Markdown is the lingua franca of modern RAG ingestion because it preserves heading hierarchy (#, ##), lists, and tables in a token-efficient, LLM-legible form. Preserving that structure lets you do hierarchy-aware chunking later. Flatten it to plain text and you've thrown away the document's skeleton.

## PDFs: Where Dreams Go To Die

PDF is not a document format — it's a *print* format. It describes glyph positions on a page, not logical structure. There is often no reliable notion of "reading order," "paragraph," or "column" in the file itself. This is why PDF parsing is the single hardest and most consequential problem in ingestion.

The hard cases:

- **Multi-column layouts** — naive extractors read straight across columns, splicing unrelated sentences.
- **Headers/footers/page numbers** — repeated boilerplate that pollutes every chunk.
- **Footnotes** — interrupt the main text flow if not separated.
- **Reading order** — figures, sidebars, and captions inserted mid-flow.
- **Scanned PDFs** — no text layer at all; you need OCR.

### The tool landscape

**Fast, native-text extractors (no ML):**

- **PyMuPDF (fitz)** — the speed king, 10–50× faster than pure-Python libraries, with span-level font/coordinate data. **Critical licensing caveat: PyMuPDF is AGPL-3.0** (it wraps the AGPL/commercial-dual-licensed MuPDF from Artifex). If you ship a closed-source product or SaaS, AGPL's copyleft obligations attach — you either open-source your stack or buy a commercial license from Artifex. This is a legal decision, not a technical one, and I've seen it derail projects late.
- **pdfplumber** — MIT-licensed, built on pdfminer.six, excellent for character-level layout inspection and native-PDF table extraction. Slower, but permissive and precise.
- **pypdf** — BSD-licensed, pure-Python, fine for trivial text extraction and page manipulation; weak on layout. (PyPDF2 was deprecated in 2022; all development moved to pypdf.)
- **pdfminer.six** — low-level layout analysis, MIT.

**Layout-aware document AI (ML pipelines and VLMs):**

- **Docling** (IBM Research, now under LF AI & Data governance) — MIT-licensed, self-contained, runs entirely locally. Uses DocLayNet layout analysis + TableFormer for tables, emits a typed DoclingDocument you can serialize to Markdown/JSON. It parses PDF, DOCX, PPTX, XLSX, HTML, images, and even WebVTT. My default open-source choice when structure and license matter.
- **Unstructured** — Apache-2.0 open-source core plus a commercial Serverless/Platform API (flat per-page pricing on Pay-As-You-Go, with a monthly free-page allotment; Business tier is sales-quoted); 40+ source connectors. Great unified loader; the best table/enrichment models live in the paid tier.
- **Marker** (Datalab) — converts PDF→Markdown/JSON, routing layout/OCR/equations through the Surya VLM. Fast and accurate; check its licensing for your use case.
- **MinerU** (OpenDataLab/Shanghai AI Lab) — pipeline + VLM backends, strong on complex and scientific layouts.
- **LlamaParse** (LlamaIndex) — hosted, credit-based (1,000 credits = $1.25; new accounts get 10,000 free credits/month). v2 uses tier presets: Cost-effective (3 credits/page ≈ $0.00375), Agentic (10 credits ≈ $0.0125), Agentic Plus (45 credits ≈ $0.056); a 1-credit Fast mode returns plain text only. Excellent on messy documents; a paid service.
- **Nougat** (Meta) — transformer OCR specialized for academic papers and math; niche but strong on equations.

**OCR engines (for scanned PDFs/images):**

- **Tesseract** — the open-source workhorse; accuracy degrades on complex layouts and colored backgrounds.
- **PaddleOCR** — strong multilingual, actively developed.
- **EasyOCR** — simple API, decent baseline.

**Layout/vision models under the hood:** LayoutLM/v2/v3 (joint text+layout+visual), DiT (document image transformer), and Surya (Datalab's Apache-2.0 multilingual OCR/layout/reading-order/table toolkit that now runs through a single VLM).

### Benchmark reality (read the caveats!)

The dominant benchmark is **OmniDocBench** (CVPR 2025, OpenDataLab, arXiv 2412.07626). **Warning: its scoring convention changed across versions**, and mixing them produces nonsense comparisons:

- **v1.0** reported **overall edit distance (lower is better)**, split English/Chinese.
- **v1.5/v1.6** switched to a composite **Overall score (0–100, higher is better)** = ((1−TextEdit)×100 + TableTEDS + FormulaCDM)/3, and dropped the EN/ZH split.

From the v1.0-era leaderboard (edit distance, lower = better, English): MinerU ~0.15, Marker (v1.7.1) 0.296, Unstructured (0.17.2) 0.586, Docling (2.14.0) 0.589. Docling and Unstructured score poorly on *raw edit distance* — but Docling isn't trying to win an OCR contest; it's trying to emit a stable, typed document object, and it carries the only MIT license + foundation governance of the major open-source three. Unstructured's table TEDS is effectively zero because it doesn't emit structured HTML/LaTeX tables — context, not a bug.

On the current composite scoring (higher = better), specialized VLMs now dominate. Primary-source v1.5 figures: **PaddleOCR-VL-1.5 achieves 94.5%** ("maintaining its position as the official top-ranked solution"; base PaddleOCR-VL = 92.86); **MinerU2.5 reports 90.67** ("outperforming the second-best model MonkeyOCR-pro-3B by 1.82"); **dots.ocr = 88.41**. The classic pipeline tools sit lower — MinerU's pipeline backend and Marker land in the high-70s to high-80s depending on version and subset.

On **olmOCR-Bench** (Ai2, English print OCR, 0–100 higher better), Ai2's own *reproduced* runs put **olmOCR 2 at 82.4, Marker 1.10.1 at 76.1, and MinerU 2.5.4 at 75.2** ("olmOCR 2 achieves 82.4 points… outperforms both specialized tools like Marker (76.1) and MinerU (75.8)"). Many higher VLM numbers (PaddleOCR-VL 80.0, dots.OCR 79.1) are *author-reported*, not independently reproduced. And the widely-cited "Docling 50.3" on olmOCR-Bench comes from **Datalab's own Marker 2 run — "All benchmark numbers come from Datalab's own runs, not independent third-party testing"** — so treat it with skepticism.

**The honest takeaway:** version drift is enormous (Marker moved from ~0.34 to 0.296 edit distance across releases, and 70.1 → 76.1 on olmOCR-Bench), corpora differ (OmniDocBench is bilingual with formulas/tables; olmOCR-Bench is English print OCR via binary unit tests), and self-reported numbers are everywhere. **Benchmark on *your* documents.**

### My PDF decision rule

- **Born-digital, clean, simple layout, permissive license required:** pdfplumber or PyMuPDF (mind AGPL).
- **Born-digital, complex layout, need structure + MIT:** **Docling**. It's my default.
- **Mixed/messy corpus, want a managed service, budget exists:** LlamaParse or Unstructured API.
- **Scanned/photographed, dense tables and figures, retrieval quality is paramount:** a VLM-based approach (MinerU 2.5, PaddleOCR-VL) or skip text extraction entirely and use page-image retrieval (ColPali — see below).

```python
# Docling: PDF -> clean Markdown, structure preserved
from docling.document_converter import DocumentConverter

converter = DocumentConverter()
result = converter.convert("report.pdf")          # PDF path or URL
markdown = result.document.export_to_markdown()    # headings, tables, reading order preserved
```

```python
# PyMuPDF: layout-aware text with coordinates (mind the AGPL license)
import fitz  # PyMuPDF

doc = fitz.open("report.pdf")
for page in doc:
    blocks = page.get_text("blocks")               # (x0, y0, x1, y1, text, block_no, block_type)
    blocks.sort(key=lambda b: (b[1], b[0]))         # sort by y, then x -> approximate reading order
    text = "\n".join(b[4] for b in blocks)
```

## Tables: The Silent Killer of RAG

Tables break embeddings. A dense sentence-transformer flattens a 2D grid into a 1D token sequence, and the row/column relationships — the entire meaning of the table — evaporate. "North America Q2 $18.7M" only means something if you preserve that it's the intersection of a row and a column.

**Strategies that actually work:**

1. **Serialize to Markdown or HTML** before embedding, preserving explicit column headers. A chunk like *"Table 3: Quarterly Revenue by Region — Q2: North America $18.7M, Europe $11.2M"* is far more retrievable than raw CSV, because queries are natural language.
2. **Table summary + raw table.** Embed an LLM-generated natural-language summary for retrieval; store the raw structured table for the generation step. This is the highest-recall pattern I use for financial docs.
3. **Row-wise vs. whole-table chunks.** Small lookup tables → one chunk. Large tables → row-wise chunks that repeat the header, so each row is self-contained.
4. **LLM/VLM table-to-text** for gnarly merged-cell tables that structural parsers mangle.

**Tools:** Camelot (lattice mode for bordered tables, stream mode for whitespace-aligned — outputs pandas/CSV/JSON/Markdown, requires Ghostscript); Tabula (Java-based); pdfplumber's table extraction (borderless, reliable fallback); Docling's TableFormer; Unstructured's table extraction; img2table (image+OCR). Camelot/pdfplumber need a text layer — scanned tables need OCR or a VLM first. On OmniDocBench's table TEDS metric, the accuracy gap between naive extractors and structure-aware tools is large, so this is worth measuring explicitly.

## Images, Figures, Charts, Diagrams

A figure your parser drops is information your RAG system can never retrieve. Three approaches, in ascending sophistication:

1. **Caption at index time.** Run each image through a VLM (GPT-4o, Gemini, Claude, Qwen-VL, LLaVA, Florence-2) to generate a description; embed the description with your text embedder. Cheap, simple, works with any text vector store.
2. **OCR-on-images** for screenshots and infographics with embedded text.
3. **Multimodal / page-image embeddings.** CLIP and SigLIP map images and text into a shared space — but they're trained on *natural* images and underperform on dense text pages. For document pages, use **ColPali / ColQwen** (late-interaction, ColBERT-style multi-vector embeddings over page images, built on PaliGemma/Qwen-VL, benchmarked on ViDoRe). ColPali treats each page as an image and skips OCR entirely, preserving spatial relationships that flattening to text destroys.

**When to use what:** For text-heavy documents where structure matters, caption-based indexing is cheap and effective. For visually dense material — financial reports with multi-line tables, slide decks, scanned forms — page-image retrieval (ColQwen2.5) wins, because on scanned corpora OCR errors in a text pipeline drag hybrid scores *below* ColPali-only. Practitioners report the largest text-vs-visual gaps on dense financial PDFs. The cost is GPU memory and storage: multi-vector page embeddings are much heavier than single dense vectors.

## Office Documents

The unglamorous majority of enterprise knowledge lives in Office files, including legacy binary formats.

- **DOCX:** python-docx (paragraphs, styles), docx2python (nested structure, headers/footers), mammoth (semantic HTML→Markdown conversion — my pick when you want heading structure).
- **PPTX:** python-pptx — iterate shapes and text frames; remember speaker notes.
- **XLSX:** openpyxl for cell-level access, pandas for tabular loads. Treat spreadsheets as tables (see above), not free text.
- **Email:** .eml via Python's stdlib email; .msg (Outlook) via extract-msg. Preserve headers (from/to/date/subject) as metadata.
- **Unified loaders:** **Docling** and **Unstructured** both ingest DOCX/PPTX/XLSX/HTML/email through one API and normalize to a common representation — this is where they earn their keep, replacing a zoo of format-specific libraries. (Watch for legacy .doc/.xls/.ppt in old SharePoints — they need OLE2-capable loaders.)

## Audio & Video Transcripts

Meetings, calls, and webinars are a goldmine most pipelines ignore.

- **Whisper / WhisperX** — WhisperX adds VAD pre-segmentation, batched inference, and **word-level timestamps** via wav2vec2 forced alignment, plus **speaker diarization** through pyannote. Its README claims **70× realtime transcription with large-v2 in under 8GB of GPU memory**. **Deepgram** and **AssemblyAI** are strong managed alternatives with built-in diarization.
- **VTT/SRT parsing:** these files carry timestamps and (sometimes) speaker labels. Preserve them.
- **Timestamp-preserving, speaker-aware chunking:** chunk on speaker turns or topic boundaries, not fixed windows, and store start/end/speaker as metadata so answers can cite "at 14:32, the CFO said…". A known WhisperX gotcha: it assigns speaker labels at chunk boundaries, so a single label can bleed across a turn when multiple speakers overlap — validate diarization on your own audio.

```python
# Parse a VTT transcript into timestamped, speaker-aware chunks
import webvtt

segments = []
for caption in webvtt.read("meeting.vtt"):
    segments.append({
        "start": caption.start,        # "00:14:32.000"
        "end": caption.end,
        "text": caption.text.strip(),
    })
# Merge consecutive segments by speaker/topic into retrieval chunks,
# keeping start/end as metadata for citation.
```

## Code Repositories

Splitting code on character count shreds functions mid-body and destroys the logic the model needs. Use **AST-aware chunking**: parse with **tree-sitter** (via tree-sitter-language-pack, which bundles 306+ grammars behind one API), walk the tree, and split on real boundaries — functions, classes, methods. Merge tiny nodes; break oversized ones. LangChain's RecursiveCharacterTextSplitter.from_language() is a keyword-based approximation (it scans for `\ndef`/`\nclass`) — fine for clean Python, but it gets fooled by def inside multiline strings; true tree-sitter AST splitting is the production answer. Attach scope, imports, and signatures as chunk context to boost embedding quality.

## Web & Crawling

- **Firecrawl** — purpose-built for RAG; recursively crawls a site and returns clean Markdown per page, with headless-Chromium JS rendering. Managed, with a free tier.
- **Crawl4AI** — open-source, self-hosted, LLM-aware; the best free alternative when you want to own the pipeline (no API costs, no rate limits).
- **Jina Reader** — URL→Markdown, great for quick public-URL experiments.
- **Scrapy** for large custom crawls; **Playwright** when you need a real browser (login, clicks, rendering).
- **JS-rendered pages:** static fetchers get an empty shell. You need headless-browser rendering (Firecrawl's Chromium, Crawl4AI, or Playwright) to execute JavaScript before extraction. This is the #1 silent web-ingestion failure.

## Structured Sources: Know When NOT To Embed

This is where most teams overreach. **Not everything belongs in a vector database.**

- **Relational databases (SQLAlchemy):** For analytical/aggregation questions ("total Q3 revenue by region"), embedding rows is the *wrong tool* — you cannot sum vectors. Use **text-to-SQL**, or better, embed the **schema** and generate queries against it (embedding a few-shot bank of question→SQL pairs has been shown to lift accuracy meaningfully). But be warned: text-to-SQL (and text-to-Cypher) is not robust enough for production on complex schemas — as Neo4j's own engineers put it, "it turns out in practice that it isn't reliable and not robust enough for production use." The robust pattern is to narrow scope and generate queries deterministically from function inputs where possible. **Row serialization to text** ("Customer: Acme; Plan: Enterprise; MRR: $12k") is appropriate only for lookup/semantic search over a bounded set of records — not billions of rows, where chunking adds scalability problems and stuffs the context window with redundant, irrelevant rows.
- **Data warehouses (BigQuery/Snowflake):** query them; don't dump them into a vector store.
- **NoSQL/JSON (MongoDB):** serialize documents to readable text, preserving field names as context.
- **Knowledge graphs (Neo4j, GraphRAG):** when relationships and multi-hop reasoning matter, a graph beats top-k chunk retrieval. Microsoft's GraphRAG extracts entities/relationships with an LLM, builds a graph, detects communities (Leiden algorithm), and generates community summaries for query-focused synthesis. Hybrid vector+graph retrieval is the strongest pattern for "connect the dots across documents" questions.
- **SaaS connectors (Google Drive, Confluence, Notion, SharePoint, Slack):** use LlamaHub/LangChain/Unstructured connectors. **Critical: preserve ACLs.** Ingesting a Confluence space without carrying its permissions into your retrieval filter is a data-leak waiting to happen.
- **CDC / incremental sync:** at enterprise scale you cannot re-ingest everything nightly. Track change-data-capture, re-parse only what changed, and make ingestion **idempotent** (stable document IDs, content hashing to skip unchanged files).

```python
# Serialize a DB row to retrievable text (only for bounded lookup sets)
def row_to_text(row: dict) -> str:
    return "; ".join(f"{k}: {v}" for k, v in row.items() if v is not None)
# "customer: Acme Corp; plan: Enterprise; mrr: 12000; region: EMEA"
```

## Cross-Cutting Concerns

**Metadata extraction & preservation.** This is not optional. Every chunk should carry source, page, section hierarchy, timestamp, and ACL/permissions. Metadata powers filtering, citations, freshness, and access control — the difference between a demo and a product.

**Document hierarchy for parent-child chunking.** Preserve the heading tree so you can retrieve a small, precise child chunk but feed the LLM its parent section for context. You cannot do this if your parser flattened the structure.

**Deduplication & normalization.** Enterprise corpora are full of near-duplicates (v1, v2, final, final-FINAL). Hash and dedupe; normalize Unicode, whitespace, and encodings.

**Encoding & language.** Detect language (Docling does this in post-processing); handle non-UTF-8 gracefully; don't let mojibake into your embeddings.

**PII scrubbing.** **Microsoft Presidio** (MIT-licensed, 8,800+ GitHub stars, active 2026 releases) detects and anonymizes PII/PHI — names, cards, SSNs — and plugs in as a post-processor (it's integrated into LlamaIndex as a node post-processor). Scrub *before* text hits logs, embeddings, or the LLM context window. For regulated industries this is a hard gate. Pair it with a high-fidelity parser (Docling→Presidio) so PII split across broken layout lines isn't missed.

**Parse-quality evaluation.** You can't improve what you don't measure. Use **OmniDocBench** methodology on a sample of *your* docs: **text edit distance / similarity vs. ground truth**, and **table TEDS** (Tree-Edit-Distance Similarity) for tables. Build a small golden set of hand-corrected documents and score every parser change against it.

**Cost/latency at scale.** VLM-based parsing is accurate but expensive and slow; native extraction is cheap and fast but brittle. LlamaParse bills per credit; Unstructured's API is per-page. Route cheap documents to cheap parsers and reserve the expensive path for the hard 5–10%.

**Orchestration frameworks:** LangChain document loaders (broadest connector set — its community loaders cover Confluence, Arxiv, and dozens more), LlamaIndex readers / LlamaHub (160+ community connectors), Haystack converters (clean, strongly-typed pipelines), Docling pipelines, and the Unstructured platform. Pick based on whether you value breadth (LangChain), retrieval focus (LlamaIndex), or production clarity (Haystack).

## A Reference Ingestion Architecture

Here's the shape I build at enterprise scale. It is a **pipeline of specialized stages**, not one monolithic "loader":

**Route → Parse → Normalize → Enrich → Chunk → Embed → Store**

1. **Router.** Detect file type *and quality*. Born-digital vs. scanned. Simple vs. complex layout. Route each to the right parser — this is where cost and quality are won. A scanned PDF goes to a VLM/OCR path; a clean DOCX goes to a fast native loader.
2. **Parser.** The modality-specific tool from above. Emit a common structured representation (Markdown + JSON with coordinates/hierarchy).
3. **Normalizer.** Unicode/whitespace normalization, dedup, encoding fixes, language detection.
4. **Enricher.** Metadata extraction, table/image captioning, PII scrubbing, section-hierarchy tagging.
5. **Chunker.** Structure-aware: hierarchy-based for prose, AST for code, row-wise for tables, speaker/timestamp for transcripts.
6. **Embedder.** Text embeddings for most; multimodal (ColPali) for visually dense pages.
7. **Store.** Vector DB + metadata store, with ACLs enforced at query time.

### Decision matrix (start here, then benchmark)

| Modality | Default (permissive/OSS) | When quality is paramount |
| --- | --- | --- |
| Clean born-digital PDF | pdfplumber / PyMuPDF* | Docling |
| Complex/scanned PDF | Docling / MinerU | LlamaParse or VLM (MinerU 2.5, PaddleOCR-VL) |
| Tables | pdfplumber / Camelot | Docling TableFormer + summary-and-raw |
| Images/figures | VLM caption (Qwen-VL) | ColPali/ColQwen page-image retrieval |
| Office docs | python-docx/pptx, openpyxl | Docling / Unstructured unified |
| Audio/video | WhisperX | Deepgram / AssemblyAI |
| Code | tree-sitter AST | tree-sitter + contextual metadata |
| Web | Crawl4AI / Firecrawl | Firecrawl (JS rendering) |
| Relational DB | text-to-SQL / schema embedding | deterministic query generation |
| Multi-hop/relational | — | Neo4j / GraphRAG |

## The Takeaway

Retrieval quality is decided at parse time. Everything downstream — chunking, embeddings, reranking, the LLM — can only degrade the signal your parser extracts; none of them can recover what it destroyed. If you're serious about RAG, the parsing layer deserves the same rigor, the same benchmarks, and the same engineering investment you currently lavish on your embedding model and vector database.

Stop optimizing the tap. Fix the water-treatment plant.

**So here's my question for you:** how much of your RAG engineering time goes into parsing versus retrieval tuning? If it's less than a third, I'd bet your ceiling is lower than you think — and you can't see the ceiling because your evals don't measure parse fidelity.
