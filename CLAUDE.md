# small-cmr-infra — Guida all'orchestratore

Repo infrastrutturale. Contiene il gateway nginx e il docker-compose orchestratore.

## Struttura

```
small-cmr-infra/
├── docker-compose.yml       # Orchestratore completo (tutti i servizi)
├── nginx/
│   └── nginx.conf           # Gateway: routing URL → servizi
├── .env.example             # Template variabili d'ambiente
└── .gitignore               # Esclude .env
```

## Routing nginx

| Path | Servizio |
|------|----------|
| `/api/users/me` (exact) | password:8004 |
| `/api/auth/*` | base:8001 |
| `/api/users/*` | base:8001 |
| `/api/ditte/*` | base:8001 |
| `/api/commesse/*` | ore:8002 |
| `/api/ore/*` | ore:8002 |
| `/api/bookings/*` | richieste:8003 |
| `/health/base` | base:8001/health |
| `/health/ore` | ore:8002/health |
| `/health/richieste` | richieste:8003/health |
| `/health/password` | password:8004/health |

## Avvio

```bash
# Solo DB (sviluppo locale)
docker compose up db -d

# Tutti i servizi + ore + richieste
docker compose --profile ore --profile richieste up -d

# Tutti i moduli
docker compose --profile ore --profile richieste --profile password up -d

# Con stack di logging (produzione)
docker compose --profile ore --profile richieste --profile password --profile prod up -d
```

## Profili Docker Compose

| Profilo | Servizi |
|---------|---------|
| _(nessuno)_ | db + gateway + base |
| `ore` | + ore |
| `richieste` | + richieste |
| `password` | + password |
| `prod` | + loki + promtail + grafana |

## Setup iniziale

1. `cp .env.example .env`
2. Compilare tutti i valori nel `.env`
3. Avviare i servizi desiderati

Vedere `SETUP-REPOS.md` nella cartella `small-cmr/` per le istruzioni complete sul deploy.
