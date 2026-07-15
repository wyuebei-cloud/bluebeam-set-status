---
name: bluebeam-set-status
title: Bluebeam Batch Set Review Status
description: |
  Hermes-agent skill for batch-setting Bluebeam markup review status
  (Accepted, Rejected, Canceled, Completed, None) via PyMuPDF.
  Opens any PDF, finds markups by type/author/color/content, and
  writes hidden reply annotations that Bluebeam's Markups List
  recognizes as official review status.
triggers:
  - User says "set these markups to Completed/Accepted/Rejected"
  - User wants to batch-update Bluebeam review status without opening each markup manually
  - User asks "can you change the Bluebeam status on this PDF"
author: Hermes Community
---
# Bluebeam Batch Set Review Status

## How it works

Bluebeam reads a markup's review status from a **hidden Text annotation** that references the original markup via `/IRT` (In Response To). Four API calls:

```python
# 1. Create hidden reply → links to original markup via /IRT
reply = page.add_text_annot(
    pymupdf.Rect(markup.rect.x0, markup.rect.y0, markup.rect.x0+1, markup.rect.y0+1), " "
)
reply.set_irt_xref(markup.xref)           # /IRT: "this is a status for that markup"
reply.set_flags(30)                       # /F: Hidden + Print (no visible sticky note)
reply.update()
# 2. Set the status
doc.xref_set_key(reply.xref, "State", "(Completed)")  # State
#    StateModel is always (Review)
```

Also set timestamps via `reply.set_info(creationDate=..., modDate=...)`.

Available states: `Accepted`, `Rejected`, `Canceled`, `Completed`, `None`.

> Custom states (e.g. "Corrected") need a `/BSIStatus` catalog entry
> that PyMuPDF can write but Bluebeam doesn't recognize — stick to the 5 above.

**That's the entire mechanism.** The agent picks the markups (by author, color, type, date, page, content, or any combination) and applies this pattern.

## Pitfalls

- **Always set timestamps**. Without `creationDate`/`modDate` → `1/1/0001`.
- **Always set `/F 30`**. Without it → visible sticky note on the page.
- **Always set `/IRT`**. Without it → separate markup, not a status.
- **Don't write `/BSIStatus`** to catalog. PyMuPDF's array looks right but Bluebeam rejects it.
- Save with `deflate=True`; use temp path + `os.replace()` if file was incrementally saved.
