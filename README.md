# TVRestreamer-docs

**English** · [Български](README.bg.md)

Product documentation for **TvRestream**.

This repo is used as the `docs/` git submodule of [kocetod/TVRestreamer](https://github.com/kocetod/TVRestreamer).

| | Link |
|---|---|
| Index (English) | [en/00-README.md](en/00-README.md) |
| Index (Bulgarian) | [bg/00-README.md](bg/00-README.md) |
| Diagrams | [diagrams/](diagrams/) |
| UI screens | [ui/screens/](ui/screens/) |

Kept in the main repo:

- `TVRestreamer_Code/api/openapi.yaml` — REST API contract
- `release-notes/` — release notes
- `index.html` — portal, generated with `python3 tools/build_docs_site.py`
- `bg|en/12-development-plan.md` is generated with `python3 Context/plan/build_plan_doc.py`
- pip install markdown
- python3 tools/build_docs_site.py

Contributing: see [CONTRIBUTING.md §5](https://github.com/kocetod/TVRestreamer/blob/main/CONTRIBUTING.md) in the main repo.
