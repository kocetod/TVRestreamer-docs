# TVRestreamer-docs

Продуктова документация на **TvRestream** / Product documentation for **TvRestream**.

Това репо е git submodule (`docs/`) на [kocetod/TVRestreamer](https://github.com/kocetod/TVRestreamer).
This repo is used as the `docs/` git submodule of [kocetod/TVRestreamer](https://github.com/kocetod/TVRestreamer).

| | Български | English |
|---|---|---|
| Индекс / Index | [bg/00-README.md](bg/00-README.md) | [en/00-README.md](en/00-README.md) |
| Диаграми / Diagrams | [diagrams/](diagrams/) | [diagrams/](diagrams/) |
| UI скрийншоти / UI screens | [ui/screens/](ui/screens/) | [ui/screens/](ui/screens/) |

Извън това репо (в основното) / Kept in the main repo:

- `TVRestreamer_Code/api/openapi.yaml` — REST API договор / REST API contract
- `release-notes/` — бележки към изданията / release notes
- `index.html` — портал, генериран с `python3 tools/build_docs_site.py` / portal generator
- `bg|en/12-development-plan.md` се генерира с `python3 Context/plan/build_plan_doc.py` / is generated

Работа с промени / Contributing: виж [CONTRIBUTING.md §5](https://github.com/kocetod/TVRestreamer/blob/main/CONTRIBUTING.md) в основното репо.
