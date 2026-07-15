# bluebeam-set-status

Batch-set Bluebeam markup status (Review model) via PyMuPDF.

```bash
hermes skills install "https://github.com/wyuebei-cloud/bluebeam-set-status/raw/refs/heads/main/SKILL.md"
```

## What it does

Given a "reference PDF" (where corrections were already marked as Completed/Accepted/etc. in Bluebeam) and a "target PDF" (finished version that lost status), this skill matches annotations by position+content and adds proper hidden reply annotations so Bluebeam's **Markups List** shows the correct status with timestamps.

### Supported statuses (Bluebeam built-in Review model)

| Status | Description |
|--------|-------------|
| Accepted | Review passed |
| Rejected | Review rejected |
| Canceled | Review canceled |
| Completed | Work completed |
| None | No status |

## How it works

Bluebeam reads markup status from a hidden Text annotation that:

1. References the original markup via `/IRT` (In Response To)
2. Is hidden (`/F 30`)
3. Sets `/StateModel (Review)` and `/State (Completed)`

No Bluebeam API required — just PyMuPDF.

## Requirements

- Python 3.8+
- `pymupdf` (`pip install pymupdf`)

## Usage

```python
import pymupdf

doc = pymupdf.open("my_drawing.pdf")
page = doc[0]

for annot in page.annots():
    reply = page.add_text_annot(
        pymupdf.Rect(annot.rect.x0, annot.rect.y0, annot.rect.x0 + 1, annot.rect.y0 + 1),
        " "
    )
    reply.set_info(subject="Set to Completed", title="my_name",
                    content="", creationDate="D:20260715160000-07'00'",
                    modDate="D:20260715160000-07'00'")
    reply.set_irt_xref(annot.xref)
    reply.set_flags(30)
    reply.update()
    doc.xref_set_key(reply.xref, "StateModel", "(Review)")
    doc.xref_set_key(reply.xref, "State", "(Completed)")

doc.save("output.pdf", deflate=True)
```

For the full batch-propagation workflow (reference PDF → target PDF), load the skill in Hermes:

```bash
hermes skills install "https://github.com/wyuebei-cloud/bluebeam-set-status/raw/refs/heads/main/SKILL.md"
```

## Limitations

- **Custom states** (e.g. "Corrected") require a `/BSIStatus` document catalog entry that PyMuPDF can create structurally but Bluebeam does not recognize. Only the 5 built-in **Review** model states work reliably.
- Requires PyMuPDF; does not use Bluebeam's COM API (Revu.Launcher only exposes file-open, not full automation).

## License

MIT
