# POC Contratos Dashboard

Web dashboard for Brazil's "Oferta de Capacidade" (PEG) portal's gas
transport **contracts** data — transport contracts and master transport
contracts across TBG, TAG, and NTS.

Rebuilds itself daily from the public API and republishes to
[poc2.gasbrazil.com](https://poc2.gasbrazil.com) via GitHub Actions + Pages.
Same architecture as [poc-dashboard](https://github.com/caissonpoint/poc-dashboard)
and [ons-dashboard](https://github.com/caissonpoint/ons-dashboard) (gzip+base64
JSON payload inflated client-side into a single static HTML file — no server,
no database).

## Source

- API: `https://www.ofertadecapacidade.com.br/v2/api/graphql` (GraphQL,
  operation `getContratos`, field `contratosCarregador`)
- Site: https://ofertadecapacidade.com.br/home/contratos
- `contratos_pipeline.py` covers two of the four contract categories shown on
  the site's UI:
  - **Contrato de Transporte** (`tipoContrato: PEDIDO` / `PEDIDO_LEILAO`,
    `statusContrato: ATIVO` / `CONCLUIDO`)
  - **Contrato Master** (`tipoContrato: CONTRATO MASTER DE TRANSPORTE`,
    `statusContrato: HABILITADO`)

  **Not yet covered:** "Contrato de Transporte Legado" and "Conexão de
  Acesso". These are not served by `contratosCarregador` — confirmed by
  exhaustively querying every value the `StatusContrato` enum accepts (only
  three: `ATIVO`, `CONCLUIDO`, `HABILITADO`, all consumed by the branches
  above) and by introspecting the full GraphQL schema (33 types, nothing
  named legado/conexão/acesso). That data comes from a different, unidentified
  endpoint. To close this gap, capture a DevTools "Copy as fetch" request for
  each of those two tabs on the live site (same method used for the original
  Contrato Master capture) and pass them along — see `data/api-research.md`
  equivalent in project memory for the full research trail.
- TSO id mapping: `1 = TBG` is confirmed directly from the API. `2 = TAG` and
  `3 = NTS` are assumed from the UI's checkbox order and are not yet
  independently confirmed — worth a quick spot-check against the live site
  before fully trusting those two labels.
- The source API has no CORS headers reachable from this sandbox's network,
  so live fetching is expected to run from GitHub Actions (matching the
  precedent from ons-dashboard/poc-dashboard) rather than client-side or from
  a dev machine with restricted egress.

## Files

- `contratos_pipeline.py` — `fetch` pulls raw JSON, `build` transforms it
  into `data/contratos.parquet` (tidy store, checked into git for history).
- `dashboard.py` — builds `docs/index.html`, the single-file dashboard.
- `make_mock.py` — synthetic raw data for local testing without hitting the
  live API.
- `.github/workflows/refresh.yml` — daily cron (11:00 UTC) + push +
  manual dispatch: fetch → build → commit → deploy to Pages.

## Local dev

```
pip install -r requirements.txt
python make_mock.py                  # or: python contratos_pipeline.py fetch (needs network access to the source)
python contratos_pipeline.py build
python dashboard.py
```
