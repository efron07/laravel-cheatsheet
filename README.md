# Laravel Cheat Sheet — PharmaCare Edition

Laravel from the beginning: plain-language definitions, commands and working code,
all built around one example project — a pharmacy system called PharmaCare.

**Live site:** https://efron07.github.io/laravel-cheatsheet/

## Edit

Pages live in `docs/`, one file per section. Edit any `.md` file and push to `main` —
GitHub Actions rebuilds and republishes the site in about a minute.
You can also click the pencil icon on any page of the live site to edit it on GitHub.

## Preview locally

```bash
python3 -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
mkdocs serve        # open http://127.0.0.1:8000 — reloads as you save
```

## Add a new page

1. Create `docs/19-my-topic.md` starting with `# 19. My topic`.
2. Add it under `nav:` in `mkdocs.yml`.
3. Push.
