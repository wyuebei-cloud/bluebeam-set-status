# bluebeam-set-status

Hermes-agent skill for batch-setting Bluebeam markup review status via PyMuPDF.

```bash
hermes skills install "https://github.com/wyuebei-cloud/bluebeam-set-status/raw/refs/heads/main/SKILL.md"
```

## What it does

Opens any PDF with Bluebeam markups, sets review status (Accepted / Rejected / Canceled / Completed / None) by writing hidden reply annotations that Bluebeam's Markups List recognizes natively.

### Supported statuses

| Status | Meaning |
|--------|---------|
| Accepted | Review passed |
| Rejected | Review rejected |
| Canceled | Review canceled |
| Completed | Work finished |
| None | Clear status |

## How it works

One hidden Text annotation per markup:

```
/IRT → original markup  (links them)
/StateModel (Review)
/State (Completed)      (or Accepted, Rejected, Canceled, None)
/F 30                   (Hidden + Print)
```

No Bluebeam API, no COM, no `/BSIStatus`. Just PyMuPDF.

## Requirements

- Python 3.8+
- `pip install pymupdf`

## License

MIT
