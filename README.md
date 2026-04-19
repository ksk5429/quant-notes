# K-Fish Quant Notes

> Development notes, reviews, and progress log for the **K-Fish**
> swarm-intelligence prediction-market engine.
>
> Published at **https://ksk5429.github.io/quant-notes/**

This site is intentionally separate from the research-notes site (academic /
offshore-wind work) because the K-Fish project is a distinct agenda:
high-frequency algorithmic forecasting of prediction-market outcomes, with a
companion public risk-analyzer package.

## What's in it

- Both primary planning documents in full:
  - [K-Fish compounding infrastructure — a 90-day build guide](docs/blueprints/90-day-build-guide.md)
  - [Production-grade GitHub blueprint for a three-repo Python stack](docs/blueprints/three-repo-blueprint.md)
- The 10 architecture-decision records (ADRs) as of the current build
- A build log of what shipped when
- Live validation artefacts: Brier-parity proofs, Polymarket + UMA smoke
  results, HIP-4 watchdog state
- A review area for discussing the project with collaborators

## Local preview

```bash
cd quant-notes-site
pip install -r requirements.txt
mkdocs serve
# → http://127.0.0.1:8000
```

## Related repos

| Repo | Visibility | Purpose |
|---|---|---|
| [ksk5429/kfish](https://github.com/ksk5429/kfish) | private | Production uv-workspace monorepo (9-agent swarm, calibration spine, Telegram bot) |
| [ksk5429/polymarket-oracle-risk](https://github.com/ksk5429/polymarket-oracle-risk) | public, MIT | PyPI package for UMA oracle manipulation-risk scoring |
| [ksk5429/quant-notes](https://github.com/ksk5429/quant-notes) | public | **This site** |
| [ksk5429/research-notes](https://github.com/ksk5429/research-notes) | public | Separate site for offshore-wind PhD research |
| [ksk5429/quant](https://github.com/ksk5429/quant) | public | Earlier public scratch repo for K-Fish (frozen) |

## License

Content © 2026 Kyeong Sun Kim. All rights reserved.
