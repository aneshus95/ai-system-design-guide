# Case Study: Document Intelligence Pipeline

## The Problem

A legal tech company needs to process **50,000 contracts per month**, extracting key terms (parties, dates, obligations, termination clauses) and loading them into a searchable database.

**Constraints given in the interview:**
- Documents range from 2 to 200 pages
- Mix of scanned PDFs and native digital
- Multi-language (English, German, French, Spanish)
- Extraction accuracy: 95%+ on key fields
- Cost target: under $0.50 per document

---

## The Interview Question

> "Design a pipeline that takes a 100-page contract PDF and extracts structured data like parties, effective date, termination conditions, and payment terms into JSON."

---

## Solution Architecture

```mermaid
flowchart TB
    subgraph Intake["Document Intake"]
        PDF[Contract PDF] --> CLASSIFY{Native or Scanned?}
        CLASSIFY -->|Native| PARSE[PyMuPDF Parser]
        CLASSIFY -->|Scanned| OCR[Vision-LLM OCR<br/>Gemini 3 Flash]
    end

    subgraph Structure["Structure Recovery"]
        PARSE --> MARKDOWN[Markdown Conversion]
        OCR --> MARKDOWN
        MARKDOWN --> SECTION[Section Detection<br/>Headers, Clauses]
    end

    subgraph Extract["Extraction Layer"]
        SECTION --> PARALLEL{{"Parallel Extractors"}}
        PARALLEL --> E1[Parties Extractor]
        PARALLEL --> E2[Dates Extractor]
        PARALLEL --> E3[Obligations Extractor]
        PARALLEL --> E4[Termination Extractor]
    end

    subgraph Validate["Validation"]
        E1 --> MERGE[Merge Results]
        E2 --> MERGE
        E3 --> MERGE
        E4 --> MERGE
        MERGE --> VALIDATE[Cross-Field Validation]
        VALIDATE --> OUTPUT[Structured JSON]
    end
```

---

## Key Design Decisions

### 1. Vision-LLM for OCR Instead of Traditional OCR

**Answer:** Scanned contracts often have stamps, handwritten annotations, and complex layouts (tables, multi-column). Traditional OCR (Tesseract) produces garbled output. Gemini 3 Flash "sees" the layout and produces clean Markdown with tables preserved. Cost is higher but accuracy gain is worth it.

| Method | 100-page Scanned Contract | Accuracy | Cost |
|--------|---------------------------|----------|------|
| Tesseract | Noisy, broken tables | 60% | $0.02 |
| AWS Textract | Better, still struggles with layout | 75% | $0.15 |
| Gemini 3 Flash | Clean Markdown, tables intact | 92% | $0.35 |

### 2. Parallel Extractors vs Single-Pass

**Answer:** A single prompt asking for all fields produces worse results than specialized extractors. Each extractor has a focused prompt and schema:

```python
parties_schema = {
    "type": "object",
    "properties": {
        "party_a": {"type": "object", "properties": {
            "name": {"type": "string"},
            "role": {"type": "string"},
            "address": {"type": "string"}
        }},
        "party_b": {"type": "object", "properties": {...}}
    }
}

# Each extractor runs in parallel
async def extract_all(document: str):
    results = await asyncio.gather(
        extract_parties(document, parties_schema),
        extract_dates(document, dates_schema),
        extract_obligations(document, obligations_schema),
        extract_termination(document, termination_schema)
    )
    return merge_results(results)
```

### 3. Cross-Field Validation

**Answer:** Extraction errors often reveal themselves through inconsistencies:
- If `effective_date` is after `termination_date`, something is wrong
- If `party_a` name appears in `obligations` but spelled differently, flag for review
- If `payment_amount` is extracted but `payment_frequency` is null, incomplete

---

## Handling 200-Page Documents

The context window challenge:

```mermaid
flowchart LR
    subgraph Chunking["Smart Chunking"]
        DOC[200-page Contract] --> DETECT[Section Detector]
        DETECT --> SECTIONS[Logical Sections<br/>Recitals, Terms, Exhibits]
    end

    subgraph Process["Selective Processing"]
        SECTIONS --> FILTER{Relevant Section?}
        FILTER -->|Yes| EXTRACT[Extract Fields]
        FILTER -->|No| SKIP[Skip / Store Reference]
    end

    subgraph Merge["Result Assembly"]
        EXTRACT --> RESULTS[Partial Results]
        SKIP --> REFS[Section References]
        RESULTS --> FINAL[Final JSON]
        REFS --> FINAL
    end
```

**Key insight:** Not all 200 pages contain extractable fields. Exhibits (attached original documents) are stored as references, not processed. The "Terms and Conditions" section is often 80% of the document but contains most key fields.

---

## Multilingual Handling

German contracts use different structures than English ones. We maintain language-specific extractors:

```python
EXTRACTORS = {
    "en": {
        "parties": EnglishPartiesExtractor(),
        "dates": StandardDatesExtractor(),
        "termination": EnglishTerminationExtractor()
    },
    "de": {
        "parties": GermanPartiesExtractor(),  # Handles "GmbH", "AG" patterns
        "dates": GermanDatesExtractor(),       # DD.MM.YYYY format
        "termination": GermanTerminationExtractor()  # "Kündigung" patterns
    }
}
```

---

## Cost Breakdown

| Stage | Cost per 100-page Doc |
|-------|----------------------|
| OCR (Gemini 3 Flash, if scanned) | $0.18 |
| Section detection (GPT-4o-mini) | $0.03 |
| Field extraction (4 parallel, GPT-4o-mini) | $0.12 |
| Validation | $0.02 |
| **Total (scanned)** | **$0.35** |
| **Total (native PDF)** | **$0.17** |

Average (60% native, 40% scanned): **$0.24 per document** (under $0.50 target)

---

## Interview Follow-Up Questions

**Q: What if the extraction confidence is low?**

A: We output a confidence score per field. Fields below 0.8 are flagged for human review. The UI shows a "review queue" where humans validate only uncertain fields, not entire documents. This reduces human effort to an average of 30 seconds per document.

**Q: How do you handle contracts with non-standard layouts?**

A: We maintain a "layout library" of known contract templates. The section detector first tries to match against known templates. If no match, it falls back to heuristic detection (looking for numbered sections, ALL CAPS headers, etc.). Unknown layouts are flagged and added to the library after human review.

**Q: What about contracts where key terms are defined in exhibits?**

A: We detect cross-references ("as defined in Exhibit A") and resolve them. The extraction prompt includes relevant exhibit content when the main document references it. This prevents "null" extractions when the answer is in an attachment.

---

## Key Takeaways for Interviews

1. **Vision-LLMs beat traditional OCR** for complex layouts (tables, annotations)
2. **Parallel specialized extractors outperform single-pass** for structured extraction
3. **Cross-field validation catches extraction errors** before they reach the database
4. **Not all pages need processing**: detect relevant sections, skip exhibits

---

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Document Intelligence** | Using AI to automatically extract structured information (names, dates, clauses) from unstructured documents | Replaces manual contract review; turns PDFs into queryable database records at scale |
| **OCR (Optical Character Recognition)** | Technology that converts scanned images of text into machine-readable characters | Required for any scanned or photographed contract where the text is not already digital |
| **Vision-LLM OCR** | Using a multimodal large language model to extract text from images instead of traditional OCR software | Handles complex layouts, tables, stamps, and handwritten annotations that traditional OCR garbles |
| **Tesseract** | An open-source traditional OCR engine | Baseline comparison; achieves ~60% accuracy on complex scanned contracts due to layout sensitivity |
| **AWS Textract** | Amazon's managed OCR and document analysis service | Better than Tesseract but still struggles with non-standard layouts; ~75% accuracy at moderate cost |
| **PyMuPDF** | A Python library for parsing native digital PDF files | Fast, free text extraction for digital PDFs that does not require any model inference |
| **Markdown Conversion** | Transforming the extracted document content into Markdown format with headers, tables, and lists | Produces a clean, structured text representation that downstream extractors can reliably parse |
| **Section Detection** | Identifying logical sections (Recitals, Definitions, Obligations, Exhibits) within a document | Allows selective processing—only relevant sections are sent to extractors, reducing cost and noise |
| **Parallel Extractors** | Running separate specialized extraction calls simultaneously for each field type (parties, dates, obligations) | Outperforms single-pass extraction by giving each extractor a focused schema and prompt |
| **Parties Extractor** | A specialized model call that extracts only the contracting parties from a document | Avoids the distraction of other fields and produces cleaner output with a focused schema |
| **Cross-Field Validation** | Checking extracted fields against each other for logical consistency (e.g., effective date must precede termination date) | Catches extraction errors that look plausible in isolation but contradict other fields in the same document |
| **Confidence Score** | A 0–1 value the model attaches to each extracted field indicating how certain it is | Fields below 0.8 are flagged for human review, reducing human effort to checking only uncertain fields |
| **JSON Schema** | A formal specification describing the expected structure, types, and fields of a JSON output | Constrains the LLM to produce machine-parseable output instead of free-form text |
| **Context Window** | The maximum text length a model can read in one request (measured in tokens) | Limits how much of a 200-page document can be processed at once; drives the need for section-based chunking |
| **Smart Chunking** | Splitting a document at logical section boundaries rather than fixed character counts | Keeps related clauses together and prevents sections from being arbitrarily split across extraction calls |
| **Exhibit** | An attachment to the main contract body (e.g., a schedule, appendix, or original referenced document) | Often bulk of document pages but rarely contains primary key fields; stored as reference rather than processed |
| **Cross-Reference Resolution** | Detecting "as defined in Exhibit A" style references and including the referenced content in the extraction context | Prevents null extractions when the answer to a query is located in an attachment rather than the main body |
| **Layout Library** | A database of known contract templates used by the section detector to match documents to a familiar structure | Speeds up section detection for common contract types and improves accuracy over pure heuristic detection |
| **Heuristic Detection** | Falling back to pattern-based rules (numbered sections, ALL CAPS headers) when no template matches | Handles non-standard layouts gracefully without failing completely |
| **Multilingual Extractor** | A language-specific extraction model tuned to that language's structural conventions and terminology | Handles German GmbH/AG entity patterns, DD.MM.YYYY date formats, and "Kündigung" termination clauses correctly |
| **Field Extraction** | The act of identifying and isolating the value of a specific field (e.g., effective date) from document text | The core task of the pipeline; each extractor specializes in one field type for maximum accuracy |
| **Native PDF** | A PDF where the text is embedded digitally rather than scanned as an image | Processed cheaply with PyMuPDF; no OCR step required, reducing per-document cost from $0.35 to $0.17 |
| **Review Queue** | An interface where human reviewers validate only the low-confidence extracted fields | Focuses human effort on uncertain extractions rather than full document re-reads, averaging 30 seconds per doc |

*Related chapters: [OCR and Layout](../10-document-processing/01-ocr-and-layout.md), [Structured Generation](../05-prompting-and-context/06-structured-generation.md)*
