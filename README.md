# Dallas Agent Workshop (Notebook-first)

This repo is designed for a ~50-person hands-on meetup.

## Quick start

### 1) Create venv + install
```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2) Configure OpenRouter
Copy env template and fill in your key:
```bash
cp .env.example .env
```

Set:
- `OPENROUTER_API_KEY`
- `OPENROUTER_MODEL` (default: `arcee-ai/trinity-large-preview:free`)

### 3) Run notebook
```bash
jupyter lab
```

Open: `workshop.ipynb`

## Preflight Check
```bash
python test_model.py
```

Expected output:

```
MODEL WORKING
```

Troubleshooting:
- Regenerate your OpenRouter key if you see a 401 error.
- Restart your notebook kernel after updating `.env`.

## Notes
- Agent runs locally on your laptop.
- Model calls go to OpenRouter.
- The Python execution tool is **meetup-grade** (timeouts + some blocked imports), **not** a hardened sandbox.
