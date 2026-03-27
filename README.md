# Assignment 2, Recommender System, Data Mining

## Python environment 

Virtual enviroment uv

From the project root:
```bash
uv venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
uv pip install -r requirements.txt
```

Then open `notebooks/recommender_system.ipynb` and run all cells please and .

## Folder layout

| Folder | Contents |
|--------|----------|
| `notebooks/` | `recommender_system.ipynb` — run this for report results 
| `report/` | `report.tex`, `report.pdf`, `report.md`, `report_assets/` (**PDF figures:** `data_overview.png`, `recommendation_stats.png`) |
| `docs/` | Course assignment brief (PDF) |

## Report PDF

From the `report/` directory:

```bash
cd report
pdflatex report.tex
pdflatex report.tex
```
