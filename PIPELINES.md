# CI/CD — Pipeline e branch protection

Documento di riferimento per la pipeline CI/CD dell'intera organization `minimal-CMR`.

---

## 1. Cosa fa la CI

Ogni repo dell'organization ha il proprio workflow GitHub Actions in `.github/workflows/ci.yml`. Si attiva su:

- `push` verso `main`
- `pull_request` verso `main`

Esecuzione: ~30 s per i backend, ~10 s per `small-cmr-infra` (runner `ubuntu-latest`).

| Repo                  | Job                                     | Cosa verifica                                 |
|-----------------------|-----------------------------------------|-----------------------------------------------|
| `small-cmr-base`      | `Backend tests (pytest)` + `Frontend build (vite)` | 37 test pytest + build Vite |
| `small-cmr-ore`       | `Backend tests (pytest)` + `Frontend build (vite)` | 19 test pytest + build Vite |
| `small-cmr-password`  | `Backend tests (pytest)` + `Frontend build (vite)` | 7 test pytest + build Vite |
| `small-cmr-richieste` | `Backend tests (pytest)` + `Frontend build (vite)` | 13 test pytest + build Vite |
| `small-cmr-infra`     | `Validate infra configs`                | YAML di `docker-compose.yml` + `nginx -t` |

---

## 2. Anatomia di un workflow backend

Esempio: `small-cmr-base/.github/workflows/ci.yml`.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
          cache-dependency-path: backend/requirements.txt
      - run: pip install -r requirements.txt
      - name: Run pytest
        env:
          LOG_DIR: ${{ runner.temp }}/logs
        run: python -m pytest -q

  frontend-build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run build
```

Dettagli da ricordare:

- **`python -m pytest`** (non `pytest`): aggiunge la cwd a `sys.path` su Linux, altrimenti `from database import engine` nel conftest non risolve.
- **`LOG_DIR: ${{ runner.temp }}/logs`**: in `audit.py` il default è `/app/logs` (path Docker). Senza override il runner non ha permessi di scrittura e i test falliscono al primo `TestClient(app)`.
- **`working-directory: backend|frontend`**: ogni job lavora nella sua cartella.
- **`cache: pip|npm`**: velocizza le run successive.

## 3. Anatomia del workflow infra

`small-cmr-infra/.github/workflows/ci.yml` non ha test, ma valida i config:

```yaml
- name: Validate docker-compose YAML
  run: python -c "import yaml; yaml.safe_load(open('docker-compose.yml'))"

- name: Validate nginx syntax
  run: |
    docker run --rm \
      --add-host base:127.0.0.1 \
      --add-host ore:127.0.0.1 \
      --add-host richieste:127.0.0.1 \
      --add-host password:127.0.0.1 \
      -v "${{ github.workspace }}/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro" \
      nginx:alpine nginx -t
```

Perché `--add-host`: `nginx -t` risolve gli `upstream <name>:<porta>` al boot. Senza i mock, il check fallisce su DNS.

`docker compose config` non si può usare in CI infra: i `build.context` puntano a repo siblings (`small-cmr-base`, ecc.) che non sono presenti nel checkout del solo infra.

---

## 4. Branch protection su `main`

Configurata via `gh api -X PUT repos/minimal-CMR/<repo>/branches/main/protection`. Regole attive su tutti i 5 repo:

| Regola | Valore |
|---|---|
| `required_status_checks.strict` | `true` (PR deve essere aggiornata su `main` prima del merge) |
| `required_status_checks.contexts` | Job CI del repo (vedi tabella sopra) |
| `required_pull_request_reviews.required_approving_review_count` | `0` (PR obbligatoria, review no) |
| `allow_force_pushes` | `false` |
| `allow_deletions` | `false` |
| `enforce_admins` | `false` (escape hatch per emergenze) |

Conseguenze pratiche:

- Push diretto su `main` → **bloccato per chiunque non sia admin**.
- PR mergeabile solo se tutti i job CI sono verdi.
- Force-push e delete del branch `main` → bloccati a livello globale.
- Come admin (owner dell'org) puoi fare un push diretto se davvero serve, ma è un'eccezione.

### Verificare lo stato

```bash
gh api repos/minimal-CMR/small-cmr-base/branches/main/protection | jq
```

### Modificare le regole

```bash
# Esempio: richiedere 1 review prima del merge
gh api -X PATCH repos/minimal-CMR/small-cmr-base/branches/main/protection/required_pull_request_reviews \
  -f required_approving_review_count=1
```

### Disattivare temporaneamente (emergenza)

```bash
gh api -X DELETE repos/minimal-CMR/small-cmr-base/branches/main/protection
# … fai il fix, poi riattiva ricreandola
```

---

## 5. Workflow di lavoro tipico

```bash
# 1. Crea branch
cd small-cmr-base
git checkout -b feat/nuova-feature

# 2. Modifica, committa
git add .
git commit -m "feat: descrizione"

# 3. Push del branch
git push -u origin feat/nuova-feature

# 4. Apri la PR
gh pr create --fill

# 5. Aspetta CI (visibile nella PR o con: )
gh pr checks

# 6. Merge (squash consigliato)
gh pr merge --squash --delete-branch
```

Se il CI rosso → vedi sezione 6.

---

## 6. Debugging di una CI fallita

### Localizzare la run

```bash
gh run list --repo minimal-CMR/small-cmr-base --limit 5
```

### Vedere solo i log dello step fallito

```bash
gh run view <RUN_ID> --repo minimal-CMR/small-cmr-base --log-failed
```

### Riprodurre in locale (backend)

Stesso comando del workflow:

```bash
cd small-cmr-base/backend
LOG_DIR=/tmp/logs python -m pytest -q
```

### Re-run di una run fallita

```bash
gh run rerun <RUN_ID> --repo minimal-CMR/small-cmr-base --failed
```

---

## 7. Tweak comuni

### Cambiare versione Python o Node

Modifica `python-version` o `node-version` nel workflow.

### Aggiungere un linter

Aggiungi uno step prima del test:

```yaml
- run: pip install ruff
- run: ruff check .
```

Per renderlo bloccante anche per il merge: aggiungi il nome job ai `contexts` nella branch protection.

### Skip CI (commit di solo refactoring readme, ecc.)

Aggiungi `[skip ci]` al messaggio commit. **Attenzione**: se hai required status checks, il merge resterà bloccato perché lo status check non avrà mai uno stato.

### Forzare il rilancio del CI senza nuovo commit

Da web (Actions → Re-run all jobs) oppure:

```bash
gh run rerun <RUN_ID> --repo minimal-CMR/<repo>
```

---

## 8. Aggiungere CI a un nuovo repo dell'org

1. Crea il workflow `.github/workflows/ci.yml` (copia da un repo simile).
2. Commit + push. Il primo run su `main` registra i job come "context" disponibile.
3. Abilita la branch protection con `gh api` (vedi `gh api ... | jq` esempi sopra). Usa i nomi job esatti come `contexts`.

---

## 9. Fix storici già applicati

Documentati qui per non ripeterli:

| Sintomo | Causa | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'database'` | `pytest` non aggiunge cwd a `sys.path` su Linux | `python -m pytest -q` |
| `FileNotFoundError: '/app/logs'` durante `TestClient(app).__enter__()` | `audit.py` default `/app/logs`, runner non scrive in `/app/` | `env: LOG_DIR: ${{ runner.temp }}/logs` |
| `Problems parsing JSON` su `gh api --input` | `Set-Content -Encoding utf8` su PS 5.1 scrive BOM | Usare `[System.IO.File]::WriteAllText` con `UTF8Encoding($false)` |
