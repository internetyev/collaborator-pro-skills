# collaborator-pro-skills

Standalone Claude Code skills that use **only the Collaborator.pro Public API**.
No dependencies on other tools, CLIs, or data layers — each skill is a single self-contained `SKILL.md` prompt.

## Skills

- [`find-link-donors`](./find-link-donors/) — find link-building donor sites via `GET /api/public/creator/list` based on user criteria (DR, traffic, language, country, site age, price, site type, etc.). Outputs paginated Markdown tables and a final CSV.
  - [`SKILL.en.md`](./find-link-donors/SKILL.en.md) — English version
  - [`SKILL.uk.md`](./find-link-donors/SKILL.uk.md) — Ukrainian version
  - `SKILL.md` — symlink pointing to the active version (defaults to English). To switch to Ukrainian, re-link:
    ```bash
    cd find-link-donors && ln -sf SKILL.uk.md SKILL.md
    ```

## Setup

Each skill expects `COLLABORATOR_API` to be set in `/Users/andrei/PY/.env`:

```
COLLABORATOR_API=your_collaborator_api_key_here
```

If `/creator/list` returns `401`/`403`, request activation of the **Creator API** from Collaborator support — the endpoint is gated by default.

## Docs

- API spec (OpenAPI 3.0): https://collaborator.pro/es/api/public/default/schema
- Docs UI: https://collaborator.pro/api/public/default/docs
