
### 1.2 How the ISI text is 100% accurate

**What "100%" means here.** Every character stored in `isi_structure.json` (headings, item text,
run-in labels) is a character of the uploaded file itself, copied by Python. No model writes text
that is stored. This is enforced by a check on every node, not just by design: a node whose text
is not exactly its source characters fails the run's review (`text_not_source_exact`, an error).

The models only decide **structure**: where a section starts, which sentences form one item, which
text is a label, a heading or page furniture. A structure problem the code can detect becomes a
flag; it is never written silently.

```mermaid
flowchart TB
    F["Uploaded ISI file<br/>(PDF, or the original Word file)"]

    subgraph PY1["1. Source model (Python, no model)"]
        SM["isi_source_model.build<br/>one canonical string of the file's own characters<br/>per character: page, box, bold / italic / underline / sup, zone"]
    end

    subgraph LC["2. LlamaExtract (model): structure only"]
        LE["sections, items, levels, run-in labels, roles, categories<br/>+ each item's text as the model read it"]
    end

    subgraph PY2["3. Binder (Python): isi_binder.build_structure"]
        FIND["find each model string in the source<br/>exact, then quote/dash/NFKC-folded, then fuzzy >= 85<br/>on word boundaries, never inside another item's text"]
        STORE["store the SOURCE slice (segments = source offsets)<br/>the model string is discarded"]
        EDIT["record how the model string differed (snap.llm_edits)"]
        REP["deterministic repairs<br/>absorb run-in label, complete the line, merge a page-break split"]
        FIND --> STORE --> EDIT --> REP
    end

    subgraph ADJ["4. Adjudicator (gpt-5.2): picks an owner, writes no text"]
        GAP["text no item covers:<br/>append to previous, prepend to next, new item, or furniture"]
        DUP["text two items claim:<br/>furniture always loses; gpt-5.2 picks A or B between two content items"]
    end

    subgraph CHK["5. Checks over the whole file (Python)"]
        C1["text_not_source_exact: stored text == source slices"]
        C2["source_text_unassigned: every printed body character has an owner"]
        C3["source_text_assigned_twice: no character has two owners"]
        C4["item_without_boxes: every content item can be highlighted"]
    end

    F --> SM
    F --> LE
    SM --> FIND
    LE --> FIND
    REP --> GAP
    REP --> DUP
    GAP --> CHK
    DUP --> CHK
    CHK -->|no flags| S1["clean"]
    CHK -->|only corrections| S2["clean_with_corrections<br/>the source overrode the model; text exact"]
    CHK -->|any error| S3["needs_review<br/>structure still written; flags carry the boxes"]
```

**Why a model mistake cannot become approved wording.** A model string is used only to *find* a
span of the file. Whatever the model changed in that string, the stored text is the span:

| Model returned | Stored (the file's characters) | Recorded as |
|---|---|---|
| "…performed by the **prescber** prior to…" | "…performed by the **prescriber** prior to…" | fuzzy match, `item_corrected_from_source` |
| "10 to 14 days" (model "fixed" a typo) | "10 to14 days" (the file's typo, which the asset must match) | edit in `snap.llm_edits` |
| label "Hypersensitivity Reactions:" returned apart from the text | "Hypersensitivity Reactions: Hypersensitivity reactions, including…" | `label_absorbed` |
| "…clinical judgment should" (p1) + "be considered based on…" (p2) | one item over pages 1–2, the running footer skipped | `page_break_merged` |
| a sentence the model skipped | added to the item on the same line, or given an owner by gpt-5.2 | `line_completed` / `adjudicated` |
| text the file does not contain (e.g. an icon description) | nothing | `item_dropped_not_printed` once every printed character has an owner, else `item_not_in_source` (error) |

**How each layer contributes.**

| Layer | Deterministic? | Can it change stored characters? | If it goes wrong |
|---|---|---|---|
| Source model (text layer / Word XML) | yes | it *is* the characters | see limits below |
| LlamaExtract | no | no, only moves boundaries | the binder corrects it or a flag is raised |
| Binder repairs | yes | no, only extends or joins source spans | recorded as a correction |
| gpt-5.2 adjudicator | no | no, only picks an owner | `adjudication_failed` leaves the error flag → `needs_review` |
| Checks | yes | no | the run is `needs_review`, never silently wrong |

On the 2026-09-27 test run (§9) all 12 documents came out `clean` or `clean_with_corrections`,
with no errors.

**Limits of the guarantee.**

- It holds for the **text**. The structure (which item a sentence belongs to, the section category,
  the heading level) comes from models. The checks catch missing, doubled and invented text; they
  cannot tell whether a correctly copied sentence was put under the right sub-heading.
- The source is only as good as the file's text. A scanned PDF with no text layer has no source
  characters: every item fails to bind and the run is `needs_review`. A legacy `.doc` or a Word
  file python-docx cannot read uses the text layer of LlamaParse's render instead of the Word XML.
- Bullets, whitespace and line-end hyphens are normalised by fixed rules (the `isi_source_model`
  docstring); a glyph those rules misclassify is kept or dropped the same way every time.
- `needs_review` does not block anything today. The structure is still used; the frontend and the
  asset pipeline must read `isi_review_status` to act on it.

---
