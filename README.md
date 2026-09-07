# Safarz / سفرز

Safarz is a multilingual Karachi bus finder for daily commuters and visitors. The included interface supports:

- English, Urdu, and Roman Urdu
- Urdu right-to-left layout
- Pickup by current location or by manually entered stop
- Destination search
- Direct routes and one-change route options
- Map-first route results
- EV accessibility details (AC and wheelchair access)
- Fare details from the route card
- Login modal for saved stops and route preferences
- Light and dark themes
- Safarz assistant for route and app-help questions

This folder is intentionally safe to download and share. It contains no API keys, passwords, database credentials, or user data.


## AI Safarz assistant

The assistant is now connected to the same `backend/data/extracted_routes.json` dataset used to seed the route database. It uses **grounded retrieval (RAG-style)** rather than inventing routes:

1. The JSON routes are loaded into SQLite/PostgreSQL.
2. The chat endpoint detects Karachi stop names from the user's message.
3. It retrieves direct and one-change routes from the imported route records.
4. Distance is calculated from available coordinates; when imported stops do not have verified coordinates, the backend can geocode selected waypoints with OpenStreetMap Nominatim and calculate a road route with OSRM.
5. If `GROQ_API_KEY` is configured, a free-tier Groq-hosted open model (`openai/gpt-oss-20b`) turns the retrieved facts into a natural-language answer. Without a key, the grounded database fallback still answers route questions.

Example questions:

- `Which bus goes from Gulshan Chowrangi to Saddar?`
- `What is the shortest route from Nagan Chorangi to Numaish?`
- `Which buses go to Saddar?`
- `Is there an electric bus from Malir to Numaish?`
- `Gulshan se Saddar kaunsi bus jati hai?`

### Enable the free AI model

Copy `.env.example` to `.env`, then add your Groq API key:

```text
GROQ_API_KEY=your_key_here
GROQ_MODEL=openai/gpt-oss-20b
```

Keep the key only in `.env`/your host's secret manager. Do not put it in `index.html` or commit it to GitHub.

### Run (recommended)

You **do not need to start Flask** for the included chatbot and route finder. The Node server now reads the 69-route JSON directly and serves the chat API on the same origin.

```bash
npm install
npm start
```

Then open `http://localhost:3000`.

If `GROQ_API_KEY` is present in `.env`, Groq is used only to phrase the already-retrieved route facts. If no key is present, the local JSON-grounded fallback still answers route questions.

**Distance note:** the supplied JSON has stop sequences but no reliable GPS coordinates, so the Node-only mode reports a clearly labelled stop-count distance estimate. It does not pretend that estimate is a GPS road distance.

## Run the UI locally

Requirements: Node.js 18+.

```bash
npm install
npm start
```

Open `http://localhost:3000`.

The UI works in demo mode without a database. Demo mode makes it possible to review the complete experience before connecting production data.

## Connect a SQL database

1. Copy `.env.example` to `.env`.
2. Set `DATABASE_URL` to your PostgreSQL connection string. On Replit, use the managed database's `DATABASE_URL` secret rather than committing it.
3. Apply the starter schema only if your existing database does not already have equivalent tables:

```bash
psql "$DATABASE_URL" -f db/schema.sql
```

4. Start the server:

```bash
npm start
```

The server exposes:

| Endpoint | Purpose |
| --- | --- |
| `GET /api/health` | Shows whether a database is configured |
| `GET /api/stops?q=gulshan` | Searches stops |
| `GET /api/routes?from=Gulshan-e-Iqbal&to=Saddar` | Returns matching buses and route legs |

### Existing backend database

You do not have to migrate to the included schema. If your tables already exist, update the SQL in `server.js` inside `searchStops()` and `searchRoutes()` to map your existing column names into the documented JSON response shape below. The UI only needs this response contract:

```json
{
  "from": { "id": "stop-1", "name": "Gulshan Chowrangi", "lat": 24.91, "lng": 67.08 },
  "to": { "id": "stop-2", "name": "Saddar", "lat": 24.86, "lng": 67.01 },
  "routes": [
    {
      "id": "route-ev-03",
      "routeCode": "EV-03",
      "busName": "Gulshan → Saddar",
      "vehicleType": "ev",
      "durationMinutes": 32,
      "frequencyMinutes": 12,
      "changes": 0,
      "fare": { "label": "Rs 80 / Rs 120", "kind": "fixed" },
      "features": ["ac", "wheelchair"],
      "stops": ["Gulshan Chowrangi", "Saddar"],
      "legs": []
    }
  ]
}
```

`vehicleType` should be `ev` or `fuel`. `features` can contain `ac` and `wheelchair`. For a transfer route, set `changes` to the number of bus changes and put each ride in `legs`.

## Connect a maps API

The current preview uses a lightweight map illustration so it has no external dependency. For production maps:

1. Choose a provider such as Mapbox or Google Maps.
2. Put the server-side credential in `MAPS_API_KEY` using Replit Secrets or your host's secret manager.
3. Use a browser-restricted key in `MAPS_PUBLIC_KEY` only when the provider's JavaScript SDK requires one.
4. Replace the map illustration in `index.html` with your chosen SDK and draw the route geometry returned by your backend.

Keep map keys restricted by domain, API, and quota. Do not paste a key into the repository or chat. A useful backend response addition is:

```json
{
  "geometry": {
    "type": "LineString",
    "coordinates": [[67.08, 24.91], [67.01, 24.86]]
  }
}
```

The server already exposes `GET /api/config` with the selected provider and a public key, if configured. It never returns `MAPS_API_KEY`.

## Frontend/backend wiring

The preview ships with demo route cards to keep the UI useful when no API is configured. When you are ready to use live route data, change the search handler in `index.html` to call:

```js
const response = await fetch(
  `/api/routes?from=${encodeURIComponent(from)}&to=${encodeURIComponent(to)}`
);
const routeData = await response.json();
```

Then map `routeData.routes` into the existing route-card layout. The exact backend contract is documented above so the UI and SQL service can evolve independently.

## Language notes

Translations are kept together in the `translations` object near the bottom of `index.html`. Add new keys to all three language objects. Urdu uses `dir="rtl"` and the Noto Naskh Arabic font; Roman Urdu stays left-to-right.

## Project structure

```text
.
├── .env.example       # safe environment template
├── .replit            # Replit run configuration
├── db/
│   └── schema.sql     # optional starter PostgreSQL schema
├── index.html         # responsive Safarz UI and translations
├── public/images/     # uploaded Karachi bus photography
├── package.json       # Node scripts and PostgreSQL client
├── README.md          # setup and integration guide
└── server.js          # static server plus SQL-backed API endpoints
```

## Production checklist

- Add authentication/session handling before saving user stops.
- Add geocoding and route geometry through your chosen maps provider.
- Add stop aliases in Urdu and Roman Urdu for better search.
- Add live service alerts and last-updated timestamps.
- Rate-limit public route and geocoding endpoints.
- Restrict database and map credentials to server-side environments.

## Run locally

Run `npm install` then `npm start`, and open `http://localhost:3000`.
