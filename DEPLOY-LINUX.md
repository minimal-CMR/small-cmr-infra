# Deploy CMR su server Linux

Guida operativa per portare lo stack `minimal-CMR` da zero su un server Linux con Docker.

Per la preparazione del server (Proxmox, VM Ubuntu, Docker, Tailscale) c'è già la guida `home-server-guide.md`. Questo documento riprende **dal momento in cui Docker e Git sono installati e funzionanti** sulla macchina di destinazione.

---

## 1. Prerequisiti sul server

```bash
docker --version           # >= 24.x
docker compose version     # plugin v2.x (sintassi: `docker compose`, non `docker-compose`)
git --version              # qualsiasi versione recente
openssl version            # per generare secret
```

L'utente con cui esegui i comandi deve essere nel gruppo `docker` (`sudo usermod -aG docker $USER`, poi logout/login).

---

## 2. Layout consigliato

I 5 repo vanno tenuti come **siblings** in una cartella comune. È il layout su cui sono allineati i `build.context` del docker-compose dopo il fix di sezione 4.

```
~/cmr/
├── small-cmr-base/
├── small-cmr-infra/
├── small-cmr-ore/
├── small-cmr-password/
└── small-cmr-richieste/
```

```bash
mkdir -p ~/cmr && cd ~/cmr
```

---

## 3. Clone dei repo

Tramite HTTPS (chiunque, repo pubblici):

```bash
cd ~/cmr
for r in small-cmr-base small-cmr-infra small-cmr-ore small-cmr-password small-cmr-richieste; do
  git clone https://github.com/minimal-CMR/$r.git
done
```

Tramite SSH (se hai una chiave registrata su GitHub):

```bash
for r in small-cmr-base small-cmr-infra small-cmr-ore small-cmr-password small-cmr-richieste; do
  git clone git@github.com:minimal-CMR/$r.git
done
```

---

## 4. Fix dei `build.context` in `docker-compose.yml`

⚠️ **Importante**. Il file `small-cmr-infra/docker-compose.yml` ha i build context puntati con `../../small-cmr-XXX/backend`. Con il layout siblings della sezione 2 serve **un solo** `../`. Da `~/cmr/small-cmr-infra/`:

```bash
sed -i 's|context: \.\./\.\./small-cmr-|context: ../small-cmr-|g' docker-compose.yml
```

Verifica:

```bash
grep "context:" docker-compose.yml
# context: ../small-cmr-base/backend
# context: ../small-cmr-ore/backend
# context: ../small-cmr-richieste/backend
# context: ../small-cmr-password/backend
```

(Da committare nel repo come PR separata se la patch ti sta bene anche in sviluppo.)

---

## 5. Generazione dei secret

```bash
# Chiave JWT (uguale per tutti i servizi)
openssl rand -hex 32
# Secret inter-service (uguale tra ore e richieste)
openssl rand -hex 32
# Password root MySQL
openssl rand -base64 24
# Password app MySQL
openssl rand -base64 24
```

Tieni questi valori in un password manager — `enforce_admins: false` sui repo non protegge il `.env` dal disco del server.

---

## 6. Configurazione `.env`

```bash
cd ~/cmr/small-cmr-infra
cp .env.example .env
nano .env
```

Variabili da compilare:

| Variabile | Valore | Note |
|---|---|---|
| `MYSQL_ROOT_PASSWORD` | (da openssl) | Solo per i container; mai esporre la porta 3306 |
| `MYSQL_PASSWORD` | (da openssl) | Usata dall'utente app `appuser` |
| `SECRET_KEY` | (da openssl) | **Identica in tutti i backend** — è già propagata nel compose |
| `SERVICE_SECRET` | (da openssl) | Auth tra `richieste` ↔ `ore` |
| `ALGORITHM` | `HS256` | Default OK |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `480` | 8 ore; alzare a piacere |
| `ADMIN_EMAIL` | la tua email | Account admin seed creato al primo avvio del `base` |
| `ADMIN_PASSWORD` | password forte | **Cambiala dopo il primo login** |
| `ADMIN_NOME` / `ADMIN_COGNOME` / `ADMIN_AZIENDA` | come preferisci | Solo metadati |
| `CORS_ORIGINS` | `http://<ip-tailscale>,http://<host>` | Lista CSV degli origin frontend |
| `GRAFANA_PASSWORD` | password forte | Solo se attivi il profilo `prod` |

Permessi consigliati:

```bash
chmod 600 .env
```

---

## 7. Primo avvio

Dalla cartella `~/cmr/small-cmr-infra/`:

```bash
# Build immagini + start (tutti i moduli applicativi, senza monitoring)
docker compose --profile ore --profile richieste --profile password up -d --build
```

Cosa succede al boot di ogni servizio backend:

1. `alembic upgrade head` (vedi `Dockerfile`) crea/aggiorna le tabelle.
2. `base` esegue il seed dell'utente admin.
3. `ore` esegue il seed delle sottocommesse `ASSENZE`.

Verifica:

```bash
docker compose ps
# tutti i container in stato "Up" o "healthy"

curl -s http://localhost/health/base
curl -s http://localhost/health/ore
curl -s http://localhost/health/richieste
curl -s http://localhost/health/password
# tutti devono rispondere {"status":"ok"} o simile
```

Login admin (test rapido):

```bash
curl -X POST http://localhost/api/auth/login \
  -d "username=$ADMIN_EMAIL&password=$ADMIN_PASSWORD"
# attesa: 200 + access_token
```

---

## 8. Frontend

Il `docker-compose.yml` orchestra **solo i backend**. I frontend sono Vite + module federation:

- `small-cmr-base/frontend` è la shell che monta gli altri come moduli remoti.
- `small-cmr-ore/frontend` e `small-cmr-richieste/frontend` espongono moduli federati.
- `small-cmr-password` non ha frontend (usa quello di `base`).

Per servirli in produzione, opzione minima senza modificare il compose:

```bash
# Per ciascun frontend:
cd ~/cmr/small-cmr-base/frontend
npm ci
npm run build
# output in dist/
```

Poi servi le `dist/` con un nginx aggiuntivo o sposta i file in `/var/www/<repo>`. Aggiungi una `location /` nel `nginx.conf` del gateway che serva i file statici di `base/dist`, e `location /modules/ore/`, `/modules/richieste/` con `alias` verso le rispettive `dist`.

Configurazione completa dei frontend in produzione è fuori dallo scope di questo doc — i build sono validati dalla CI ma il deploy va impostato a parte.

---

## 9. Aggiornamenti / redeploy

Workflow standard quando arriva un fix su `main`:

```bash
cd ~/cmr

# Pull dei repo modificati (o di tutti)
for r in small-cmr-base small-cmr-infra small-cmr-ore small-cmr-password small-cmr-richieste; do
  git -C $r pull --ff-only
done

# Solo se è cambiato il backend di un servizio specifico (più veloce)
cd small-cmr-infra
docker compose --profile ore --profile richieste --profile password up -d --build base
# (sostituisci 'base' con il servizio cambiato)

# Se ha toccato anche le migrations, fai prima un backup (sez. 11) e poi:
docker compose --profile ore --profile richieste --profile password up -d --build
```

Rolling restart senza rebuild (solo riavvio):

```bash
docker compose restart base
```

---

## 10. Logs

### File log strutturati (sui container backend)

Ogni servizio scrive in `/app/logs/`:

- `access.log` — richieste HTTP (metodo, path, status, latenza, user_id)
- `app.log` — eventi applicativi (startup, password change, audit)

Per persistirli su host, aggiungi un volume nel compose:

```yaml
base:
  volumes:
    - ./logs/base:/app/logs
```

(Già fatto via PR consigliata in `small-cmr-infra/docker-compose.yml`.)

### Log Docker

```bash
docker compose logs -f                    # tutti i servizi
docker compose logs -f base               # un servizio
docker compose logs --tail 100 base       # ultime 100 righe
```

### Stack monitoring (opzionale, profilo `prod`)

```bash
docker compose --profile ore --profile richieste --profile password --profile prod up -d
```

Espone:

- Grafana → `http://<host>:3000` (login `admin` / `GRAFANA_PASSWORD`)
- Loki → `http://<host>:3100` (interno)
- Promtail raccoglie i log da `/var/log` e dai container

Datasource Loki da aggiungere in Grafana: `http://loki:3100`.

---

## 11. Backup MySQL

Script base — adatta i path e le variabili al tuo ambiente:

```bash
mkdir -p ~/cmr-backups

cat > ~/cmr-backup.sh <<'EOF'
#!/bin/bash
set -e
cd ~/cmr/small-cmr-infra
DATE=$(date +%Y%m%d_%H%M%S)
DEST=~/cmr-backups/cmr_${DATE}.sql.gz
# legge MYSQL_ROOT_PASSWORD dall'.env
source .env
docker compose exec -T db \
  mysqldump -u root -p"${MYSQL_ROOT_PASSWORD}" small_cmr \
  | gzip > "$DEST"
# ruota: tieni gli ultimi 14
ls -1t ~/cmr-backups/cmr_*.sql.gz | tail -n +15 | xargs -r rm
echo "Backup OK: $DEST"
EOF

chmod +x ~/cmr-backup.sh
```

Schedula con cron:

```bash
crontab -e
# Ogni notte alle 3:00
0 3 * * * /home/$USER/cmr-backup.sh >> /home/$USER/cmr-backups/cron.log 2>&1
```

Restore:

```bash
gunzip < ~/cmr-backups/cmr_20260613_030000.sql.gz \
  | docker compose exec -T db mysql -u root -p"$MYSQL_ROOT_PASSWORD" small_cmr
```

---

## 12. Operazioni comuni

### Reset password admin

Bypassando il flow normale (richiede accesso al DB):

```bash
docker compose exec base python - <<'PY'
from auth import hash_password
from database import SessionLocal
from models import User
db = SessionLocal()
u = db.query(User).filter_by(email="admin@azienda.com").first()
u.password_hash = hash_password("nuova-password")
db.commit()
print("OK")
PY
```

### Aprire shell nel container

```bash
docker compose exec base bash
docker compose exec db mysql -u appuser -p small_cmr
```

### Vedere consumo risorse

```bash
docker stats
```

### Ricostruire un solo servizio da zero

```bash
docker compose build --no-cache base
docker compose up -d base
```

---

## 13. Troubleshooting

| Sintomo | Causa probabile | Fix |
|---|---|---|
| `docker compose up` fallisce con "no such file or directory" sul `build.context` | Layout non siblings o sed di sezione 4 non eseguito | Vedi sez. 2 + 4 |
| `502 Bad Gateway` su `/api/auth/login` | `base` non avviato o crashato | `docker compose logs base` |
| `401` su tutti gli endpoint | `SECRET_KEY` diversa tra i servizi | Tutti i backend leggono dalla stessa `.env` (gestita dal compose) — verifica `docker compose config` |
| `richieste` risponde 503 sulle approvazioni | `ore` non raggiungibile | `docker compose ps ore` + logs |
| Admin login fallisce al primo avvio | Tabelle create ma seed non eseguito (DB già popolato) | `docker compose down -v` cancella i dati e ri-seedda (⚠️ perdi tutto) |
| Tutti i container in restart loop dopo update | Migration Alembic incompatibile | Restore da backup + risolvi in dev |
| `docker compose ps` mostra `db` non healthy | Volume corrotto o crash | `docker compose logs db`; ultimo restart `docker compose restart db` |

---

## 14. Checklist post-deploy

- [ ] `docker compose ps` → tutti `Up`/`healthy`
- [ ] `curl http://localhost/health/{base,ore,richieste,password}` → 200
- [ ] Login admin OK
- [ ] Cambio password admin eseguito
- [ ] `chmod 600 .env`
- [ ] Cron del backup attivo (`crontab -l`)
- [ ] Firewall blocca le porte non necessarie (`ufw status`); 3307 (DB) **non** deve essere aperta a internet
- [ ] Accesso al server solo via Tailscale o SSH con chiave
