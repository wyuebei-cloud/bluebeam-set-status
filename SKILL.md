---
name: bluebeam-set-status
title: Bluebeam Batch Set Review Status
description: |
  Batch-set Bluebeam markup status (Review model) via PyMuPDF.
  Given a "reference PDF" (where corrections were already made) and a
  "target PDF" (finished version that lost status), this skill matches
  annotations by position+content and adds proper hidden reply annotations
  so Bluebeam's Markups List shows the correct status with timestamps.
triggers:
  - User has a finished PDF and wants to propagate completed/corrected status from a reference PDF
  - User wants to batch-set Bluebeam markup status programmatically
  - User asks about Bluebeam status automation without Bluebeam's own API
author: Hermes Community
---
# Bluebeam Batch Set Review Status

## Available States (Bluebeam Built-in Review Model)

| State | Value | Works without config |
|-------|-------|---------------------|
| Accepted | `(Accepted)` | ✅ No extra setup needed |
| Rejected | `(Rejected)` | ✅ |
| Canceled | `(Canceled)` | ✅ |
| Completed | `(Completed)` | ✅ |
| None | `(None)` | ✅ |

These use Bluebeam's native **Review** state model (`/StateModel (Review)`).
No `/BSIStatus` document-level config required.

> **Custom states (e.g. "Corrected") cannot be written via PyMuPDF.**
> Bluebeam's `/BSIStatus` custom state model is stored as a PDF array;
> PyMuPDF can write matching serialized content but Bluebeam still rejects it.
> If you need "Corrected" display, set it once manually in Bluebeam first — that
> generates a valid `/BSIStatus` that subsequent automated writes can reference.

## Core Mechanism

Bluebeam reads markup status from a **hidden Text annotation** that:

1. Has `/IRT` (In Response To) pointing to the original markup's xref
2. Uses `/Subtype /Text` and `/F 30` (Hidden + Print + NoZoom + NoRotate)
3. Sets `/StateModel (Review)` and `/State (Completed)` (or any of the 5 states)

```python
import pymupdf

doc = pymupdf.open("target.pdf")
page = doc[page_num]

for annot in page.annots():
    rect = annot.rect
    # Create hidden reply annotation
    reply = page.add_text_annot(
        pymupdf.Rect(rect.x0, rect.y0, rect.x0 + 1, rect.y0 + 1), " "
    )
    reply.set_info(
        subject="Set to Completed",
        title="<username>",
        content="",
        creationDate="D:20260714111827-07'00'",
        modDate="D:20260714111827-07'00'"
    )
    reply.set_irt_xref(annot.xref)   # Link to original
    reply.set_flags(30)              # Hidden + Print + NoZoom + NoRotate
    reply.update()

    # Set state model and value
    nx = reply.xref
    doc.xref_set_key(nx, "StateModel", "(Review)")
    doc.xref_set_key(nx, "State", "(Completed)")
```

## Full Workflow: Propagate Status from Reference PDF

### Step 1: Parse reference PDF

Open the reference PDF (where user already marked items as corrected).
For each hidden Text annotation with `/State (Corrected)` or `/State (Completed)`:

```python
corrected_originals = {}   # xref_of_original -> creation_date
for page in ref_doc:
    for a in page.annots():
        state = ref_doc.xref_get_key(a.xref, "State")
        if state[1] in ("Corrected", "Completed"):
            irt = ref_doc.xref_get_key(a.xref, "IRT")
            if irt[0] == "xref":
                irt_xref = int(irt[1].split()[0])
                import re
                raw = ref_doc.xref_object(a.xref)
                m = re.search(r'/CreationDate \(([^)]*)\)', raw)
                corrected_originals[irt_xref] = m.group(1) if m else ""
```

### Step 2: Build position map from reference PDF

```python
ref_map = {}
for pn, page in enumerate(ref_doc, 1):
    ref_map[pn] = []
    for a in page.annots():
        r = a.rect
        ref_map[pn].append({
            "xref": a.xref,
            "cont": a.info.get("content", "").strip(),
            "author": a.info.get("title", ""),
            "x0": r.x0, "y0": r.y0,
        })
```

### Step 3: Match and apply to target PDF

```python
target = pymupdf.open("finished.pdf")
count = 0

for pn, page in enumerate(target, 1):
    if pn not in ref_map:
        continue
    for a in page.annots():
        r = a.rect
        c = a.info.get("content", "").strip()

        for ref in ref_map[pn]:
            if ref["author"] != "adelacruz" or ref["cont"] != c:
                continue
            if abs(ref["x0"] - r.x0) + abs(ref["y0"] - r.y0) > 80:
                continue
            if ref["xref"] in corrected_originals:
                date = corrected_originals[ref["xref"]]
                na = page.add_text_annot(
                    pymupdf.Rect(r.x0, r.y0, r.x0 + 1, r.y0 + 1), " "
                )
                na.set_info(subject="Set to Completed", title="<username>",
                            content="", creationDate=date, modDate=date)
                na.set_irt_xref(a.xref)
                na.set_flags(30)
                na.update()
                nx = na.xref
                target.xref_set_key(nx, "StateModel", "(Review)")
                target.xref_set_key(nx, "State", "(Completed)")
                count += 1
                break

target.save("output.pdf", deflate=True)
target.close()
```

## Pitfalls

- **Position matching threshold**: Use 80px as the distance threshold for matching
  annotations between documents. Adjust if pages have different scales.
- **Author filter**: Only process annotations by the reviewer
  (e.g. `adelacruz`). Skip the user's own annotations.
- **Content matching**: Both `content` (FreeText text) AND position must match.
  Content-free Cloud annotations match by position only.
- **Timestamps**: Always copy the original `CreationDate` from the reference PDF's
  reply annotation. Without it, Bluebeam shows `1/1/0001`.
- **Save format**: Use `deflate=True` for compression. If the target has been
  incrementally saved, save to a temp path first then `os.replace()`.
- **Do NOT delete annotations before modifying**: `page.delete_annot()` on
  annotations with `/IRT` relationships can orphan them. Work on a clean copy
  of the target PDF instead.
- **Do not add `/BSIStatus` to catalog**: PyMuPDF-created BSIStatus arrays are
  structurally correct but Bluebeam rejects them. Only use the built-in
  Review model states.

## Verification

```python
doc = pymupdf.open("output.pdf")
verified = sum(
    1 for page in doc
    for a in page.annots()
    if doc.xref_get_key(a.xref, "State")[1] == "Completed"
)
print(f"Verified: {verified}")
```

Open in Bluebeam and check the Markups List Status column.
