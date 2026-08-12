# FoodFlow — Fullstack (Non-Containerized)

FoodFlow is a food-delivery platform built as 8 independent Python/FastAPI
microservices behind a single API Gateway, backed by one shared MongoDB
database and a Redis instance for live delivery tracking, with a plain
HTML/JS frontend. Everything runs as plain local processes on `localhost` —
no Docker, no Podman, no container runtime required.

```
Frontend (localhost:3000)
      │
      ▼
API Gateway (localhost:8000)
      │
  ┌───┼─────────┬──────────┬───────────┬──────────┬──────────┬─────────────┬──────────┐
  ▼   ▼         ▼          ▼           ▼          ▼          ▼             ▼
user  restaurant order    delivery    payment    review    notification  offer
:8001 :8002     :8003     :8004       :8005      :8006     :8007         :8008
  │      │         │          │           │          │           │            │
  └──────┴─────────┴──────────┴───────────┴──────────┴───────────┴────────────┘
                 MongoDB (:27017)                Redis (:6379)
```

All 8 services + the API Gateway + the frontend run as separate local
processes (one `uvicorn` process per service, `python -m http.server` for
the frontend), talking to each other over `localhost` and the ports above.

## Prerequisites (install once)

1. **Python 3.11+**
2. **MongoDB** — install MongoDB Community Server locally, or use a free
   MongoDB Atlas cluster and paste its connection string into `.env`.
   https://www.mongodb.com/try/download/community
3. **Redis** — OPTIONAL. Only used by delivery-service for the live
   location-tracking WebSocket. Every other feature works fine without it.
   Install locally (`sudo apt install redis-server` / Memurai on Windows /
   Redis via WSL) if you want live tracking.

## First-time setup

1. Extract this project anywhere.
2. *(Optional)* Copy `.env.example` to `.env` in the project root and edit
   `JWT_SECRET` before using this anywhere but a lab — the checked-in
   default is a placeholder. `main.py` does this copy for you
   automatically on first run if you skip this step.
3. *(Optional)* Seed the database schema/indexes once MongoDB is running:
   ```bash
   mongosh < database/mongo_init.js
   ```
   Not required — collections and indexes are created automatically on
   first use, this just seeds them up front.

## Running it

From the project root, run the single entry point:

```bash
python main.py
```

This will, in order:

1. Install everything in the consolidated `requirements.txt` into your
   current Python interpreter (skip with `--no-install` on later runs).
2. Copy `.env.example` to `.env` if you haven't already.
3. Check that MongoDB is reachable on `localhost:27017` (warns if not —
   start it with `mongod` in another terminal first if needed).
4. Start all 8 microservices (ports 8001–8008) as `uvicorn` subprocesses,
   each with its own reload loop and crash isolation.
5. Start the API Gateway (port 8000).
6. Serve the frontend (port 3000) via `python -m http.server`.

Then open **http://localhost:3000** in your browser.

Press **Ctrl+C** once to stop every service cleanly.

Useful flags:

```bash
python main.py --no-install     # skip the dependency install step
python main.py --install-only   # only install deps, don't start anything
python main.py --background     # start everything detached, then return the terminal to you
python main.py --stop           # stop a --background run
```

### Running without keeping a terminal open

By default, `main.py` is the parent process of every service - closing the
terminal (or closing VS Code, which closes its integrated terminal with it)
kills everything, since there's no container runtime keeping things alive
independently of your shell.

To keep FoodFlow running after you close the terminal/editor:

```bash
python main.py --background
```

This installs deps (if needed), verifies the project layout, starts every
service detached from the current console, writes its PID to `foodflow.pid`,
and hands the terminal straight back to you. Output goes to
`foodflow-background.log` instead of the console. Stop it later, from any
terminal, with:

```bash
python main.py --stop
```

### What happens if you edit or delete a file while it's running

- **Delete/rename a file before starting** - `python main.py` runs a layout
  check first and refuses to start, listing exactly which files/folders are
  missing, instead of a confusing partial startup.
- **Edit a file while services are already running** - each service runs
  with `--reload`, so a saved change restarts *that* service. If the edit
  breaks it (syntax error, bad import, etc.), `main.py` automatically
  retries that one service up to 3 times with a short delay. The other
  services are never touched. If it's still broken after 3 tries, `main.py`
  leaves just that service down and prints which one, so you can fix the
  file and run `python main.py --no-install` again to bring it back -
  one bad edit doesn't take the whole stack offline.

### Ports

| Service               | Port  |
|------------------------|------|
| Frontend               | 3000 |
| API Gateway            | 8000 |
| user-service           | 8001 |
| restaurant-service     | 8002 |
| order-service          | 8003 |
| delivery-service       | 8004 |
| payment-service        | 8005 |
| review-service         | 8006 |
| notification-service   | 8007 |
| offer-service          | 8008 |
| MongoDB                | 27017 |
| Redis                  | 6379 |

## Using the app

1. Register an account and pick a role: **customer**, **restaurant
   owner**, **delivery partner**, or **admin**.
2. As a **restaurant owner**: go to "My restaurant", create it (optionally
   with a photo), add menu items, upload/change your restaurant's photo
   any time, and see every order placed for your restaurant with the
   customer's name, delivery partner, and location.
3. As a **customer**: browse restaurants (owner-uploaded photos show on
   the cards), add items to your cart with a little fly-to-cart
   animation, enter a delivery address (or use "Use my location" /
   the automatic GPS prompt), and place your order — payment is
   processed automatically (mocked, ~95% success rate) and you can
   track status and leave a review once delivered.
4. As a **delivery partner**: register your vehicle, then see two
   tables — "Available orders" (any unassigned order, with a **Pick
   up** button to self-assign it) and "My deliveries" (your current +
   past deliveries, each with the customer's name and delivery
   location, and a **Mark delivered** button while active).
5. As an **admin**: view live order/revenue stats, manage discount
   offers, see every registered delivery partner and who's available,
   dispatch a specific (or automatic) partner to an order, and browse
   an all-orders table with customer, restaurant, delivery partner, and
   location for every order in the system.

## Troubleshooting

- **"Can't reach the API Gateway"** in the frontend — make sure
  `python main.py` is still running and didn't print a startup error for
  the gateway or any service.
- **A service exits immediately** — almost always a missing/invalid env
  var, or MongoDB not reachable yet. Check the terminal output for that
  service (each one prints its own logs inline).
- **`bcrypt`/`passlib` import or hashing errors** — make sure the pinned
  `bcrypt==4.0.1` actually installed (passlib 1.7.4 breaks on
  `bcrypt>=4.1`); re-run `python main.py` (installs by default) or
  `pip install -r requirements.txt` by hand.
- **Port already in use** — another process is already bound to one of
  the ports above. Stop it, or stop any leftover `python`/`uvicorn`
  processes from a previous run, then re-run `python main.py`.
- **Mongo data looks stale/wrong after schema changes** — drop the
  `foodflow` database in `mongosh` and re-run
  `mongosh < database/mongo_init.js`, then restart.

## Project layout

```
foodflow/
├── main.py                    # single entry point: installs deps, starts every service + gateway + frontend
├── requirements.txt           # consolidated deps used by every service
├── shared/                    # config, MongoDB connection, JWT auth (used by all services)
├── api-gateway/                # single entry point, routes to each service, port 8000
├── services/
│   ├── user-service/           # register/login/profile, port 8001
│   ├── restaurant-service/     # restaurants + menu, port 8002
│   ├── order-service/          # order lifecycle, port 8003
│   ├── delivery-service/       # partner assignment + tracking, port 8004
│   ├── payment-service/        # mocked payment gateway, port 8005
│   ├── review-service/         # ratings & reviews, port 8006
│   ├── notification-service/   # mocked email/sms/push, port 8007
│   └── offer-service/          # coupon codes, port 8008
├── frontend/
│   └── index.html              # the whole UI — no build step, no npm
└── database/
    └── mongo_init.js           # optional one-time schema/index seed script (run with mongosh)
```
