# Safarz Flask + SQL backend

This folder contains the Flask API for Safarz. It uses SQLAlchemy, supports PostgreSQL through `DATABASE_URL`, and defaults to a local SQLite database at `backend/data/safarz.db` for development. The first run creates the tables and seeds the extracted stops and routes. Bus recommendation images are selected from the supplied `attached_assets/` folder and exposed through `/assets/<filename>`.

## Run locally

From the repository root:

```bash
python -m venv .venv
.venv\\Scripts\\Activate.ps1
pip install -r backend/requirements.txt
python -m backend.app
```

The API runs at `http://localhost:5000`. Copy `.env.example` to `.env` or set environment variables in your shell. Set `DATABASE_URL` to a PostgreSQL connection string before starting it in deployment. Set `SEED_DATABASE=0` when the target database already contains its own data.

## Optional Groq AI ranking and chatbot

Route search always queries the SQL records imported from the guide images first. To let the free Groq API rank those candidates and answer `/api/chat`, set `GROQ_API_KEY`; ranking is enabled automatically (`GROQ_RANKING=1` by default). The default model is `llama-3.1-8b-instant`; override it with `GROQ_MODEL` if needed. Groq receives only route IDs, ordered stops, direct/connecting status, and distance. It cannot add buses or stops. If the key is absent, ranking is disabled, the service is unavailable, or the response is invalid, ranking and chat safely fall back to database results.

The guide images do not contain reliable GPS coordinates. By default, the service geocodes the selected endpoints with free OpenStreetMap Nominatim and calculates road distance/geometry with OSRM. Results are cached in memory. Set `GEOCODING_ENABLED=0` to disable network lookups; the UI then displays “Distance unavailable”. Public services are rate-limited, so production should use a provider key and persistent cache.

To rebuild local SQLite data from the extracted JSON, stop any running Flask process, then run:

```powershell
$env:RESET_DATABASE="1"
python -m backend.app
```

For Groq, copy `.env.example` to `.env`, add your Groq key, and restart Flask. The backend loads `.env` automatically; never commit that file.

Keep that terminal open while using the app. Use `Ctrl+C` to stop the server; `python -m backend.app` is intentionally long-running.

Run the backend checks with:

```bash
python -m unittest backend.test_app -v
```

The legacy `server.js` remains available for the original Node preview. Use `backend.app` when running the Flask service.

## Image extraction

`backend/data/extracted_routes.json` contains the route-guide transcription from the raw JPEG pages. It imports the named mini-bus, coach, other-bus, red-bus, EV, and BRT entries from the supplied pages. Each route stores its source image filename. The guide does not provide reliable coordinates, fares, or schedules, so those fields are left as defaults or null rather than guessed. Add reviewed records to the JSON file and rerun against a fresh database, or write a migration for an existing database.

The imported route data is used by `/api/stops` and `/api/routes`; recommendation images use `/assets/<filename>`. Route responses include `category`, `distanceKm`, `coordinates`, `recommended`, and `imageUrl` in addition to the existing frontend contract. Because the guide images do not provide GPS coordinates, `distanceKm` is `null` until verified coordinates are added; the app never invents kilometer values. Results include direct routes plus valid one-transfer alternatives and prefer direct routes, then route order.

## Routes

### `GET /api/stops?q=...`

Returns:

```json
{
  "stops": [
    {
      "id": "stop-id",
      "name": "Gulshan Chowrangi",
      "nameUrdu": "گلشن چورنگی",
      "latitude": 24.91,
      "longitude": 67.08
    }
  ]
}
```

### `GET /api/routes?from=...&to=...`

Returns the route contract in the root `README.md`. This lets the frontend display direct buses, EV features, fares, transfer count, stop sequences, and map geometry without knowing SQL details.

## Secrets

Use Replit Secrets or environment variables for `DATABASE_URL` and `MAPS_API_KEY`. Never put either value in `index.html`, `README.md`, a commit, or a chat message.

## AI assistant

`POST /api/chat` is grounded in `data/extracted_routes.json` through the seeded route database. It detects stop names in natural-language questions, retrieves direct/one-change routes, and returns route facts plus a natural-language answer.

Set `GROQ_API_KEY` to enable the optional free-tier Groq model. The app has a deterministic database fallback when no key is present.
