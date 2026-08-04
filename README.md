# climate-index-bus

A unified data layer for weather- and water-index instruments. One schema, one
query API, many underlying sources (NOAA station data, ERA5 reanalysis,
satellite precipitation, water-trading registries) — so anyone building a
weather/water derivative, parametric insurance product, or index-tracking tool
doesn't have to write a new adapter for every data source.

This is the foundational piece from the [gap analysis on water & weather
derivatives](../water-weather-derivatives-gaps-and-platforms.md): pricing
libraries, basis-risk tooling, and index-construction platforms all need clean,
normalized climate data. This project is that plumbing.

## Why this exists

Weather/water derivative contracts settle against wildly different data
sources with no common schema: CME temperature contracts use NOAA station
data, water indices use private registries like Waterlitix, parametric
insurance products increasingly use satellite precipitation (CHIRPS, IMERG)
or reanalysis (ERA5). Every project that wants to work across more than one
of these ends up writing bespoke ingestion code. `climate-index-bus`
normalizes them into one record shape and one query interface.

## Status

**Early scaffold.** NOAA GHCN-Daily connector is implemented and functional
(no API key required — reads NOAA's public archive directly). ERA5 and
satellite precipitation connectors are stubbed with the same interface,
ready to be filled in. Storage defaults to SQLite for local dev; swap in
Postgres/TimescaleDB via `DATABASE_URL` for production.

## Architecture

```
                    ┌─────────────────┐
   NOAA GHCN  ─────▶│                 │
                     │   Connectors    │  each implements BaseConnector.fetch()
   ERA5 (stub) ─────▶│  (normalize to  │  and returns list[ClimateRecord]
                     │  ClimateRecord) │
   CHIRPS (stub) ───▶│                 │
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Store (SQLAlchemy)  │  SQLite (dev) / Postgres+Timescale (prod)
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │   FastAPI layer  │  /records, /stations, /ingest
                     └─────────────────┘
```

Every record, regardless of source, normalizes to the same shape
(`ClimateRecord` in `schema.py`):

| field | meaning |
|---|---|
| `source` | e.g. `"noaa_ghcn"`, `"era5"` |
| `station_id` | station or grid-cell identifier |
| `variable` | e.g. `"TMAX"`, `"TMIN"`, `"PRCP"` |
| `date` | ISO date |
| `value` | numeric value |
| `unit` | e.g. `"degC"`, `"mm"` |
| `lat` / `lon` | location, when known |
| `confidence` | source-reported quality flag, if any |

## Quickstart

```bash
pip install -e ".[dev]"

# Ingest a NOAA station (no API key needed — pulls from NOAA's public archive)
python -m climate_index_bus.cli ingest noaa --station USW00023174 --start 2024-01-01 --end 2024-12-31
# USW00023174 = Los Angeles Intl Airport, one of CME's benchmark weather cities

# Run the API
uvicorn climate_index_bus.api:app --reload

# Query it
curl "http://localhost:8000/records?station_id=USW00023174&variable=TMAX&start=2024-06-01&end=2024-06-30"
```

## Benchmark stations

To make outputs directly comparable/backtestable against real CME weather
contracts, `stations.py` ships a starter list mapping CME's benchmark weather
cities to NOAA GHCN station IDs (Chicago, NYC, Atlanta, LA, etc.). Use
`python -m climate_index_bus.cli list-stations` to see them.

## Roadmap

- [x] `ClimateRecord` normalized schema
- [x] NOAA GHCN-Daily connector (public archive, no auth)
- [x] SQLite/Postgres storage layer
- [x] FastAPI query layer
- [x] CLI for ingestion
- [ ] ERA5 connector (needs free Copernicus CDS API key — stubbed, see `connectors/era5.py`)
- [ ] CHIRPS/IMERG satellite precipitation connector
- [ ] Degree-day derived-variable endpoint (HDD/CDD computed on the fly from TMAX/TMIN)
- [ ] Water-registry connector (pluggable — schema differs by state/region, starting with a generic CSV importer)
- [ ] Docker Compose with TimescaleDB for production-scale ingestion

## Contributing a connector

Every connector implements `BaseConnector.fetch(**params) -> list[ClimateRecord]`.
See `connectors/noaa.py` for a working example and `connectors/era5.py` for the
stub interface to fill in. PRs adding new sources (regional water registries in
particular) are the highest-value contribution right now.

## License

MIT — see `LICENSE`.
