# Agricorps RAG — Contabo Deployment

**Live site:** https://agricorps.rapidsai.com  
**Server:** `217.77.10.180` (Ubuntu 22.04)  
**Project directory:** `/root/RapidsAI-Portfolio/Agricorps-RAG`

## 1. Clone repository

```bash
ssh root@217.77.10.180
mkdir -p /root/RapidsAI-Portfolio
cd /root/RapidsAI-Portfolio
git clone https://github.com/aidevxengineer-coder/Agricorps-RAG.git
cd Agricorps-RAG
```

## 2. Dependencies and storage

Docker and Docker Compose are required. On this VPS, Docker was already installed and Compose v2 was added.

```bash
apt update && apt install -y docker-compose-v2
docker network create rapidsai-portfolio  # one-time setup; currently not used by this stack
mkdir -p chroma_db uploads user_uploads pdfs
```

## 3. Docker files

The deployment uses these files created in the repository root:

- `Dockerfile.backend` — Python 3.11, FastAPI/Uvicorn, RAG packages, Tesseract OCR.
- `Dockerfile.frontend` — builds React/Vite with Node 22 and serves `dist/` using Nginx.
- `nginx.frontend.conf` — proxies `/api/` and `/agent/` to `http://backend:8001`; serves React routes.
- `compose.yaml` — `backend` (internal port 8001) and `frontend` (host `127.0.0.1:8085`); isolated `agricorps-rag-internal` network. The backend mounts `./:/app` to retain SQLite, ChromaDB and uploads.

**Important:** The Compose stack uses its own `agricorps-rag-internal` network, not the separately created `rapidsai-portfolio` network.

## 4. Environment variables

```bash
touch .env
chmod 600 .env
# First-time setup only:
echo "JWT_SECRET=$(openssl rand -hex 32)" >> .env
nano .env
```

Add required API credentials (e.g., `GROQ_API_KEY`) and other application configuration. **Do not commit or share `.env`.** Preserve the existing `JWT_SECRET` on subsequent deployments.

## 5. Build and start

```bash
docker compose config --quiet
docker compose build
docker compose up -d
docker compose ps
curl -I http://127.0.0.1:8085
```

Expected frontend response: `HTTP 200`.

## 6. Domain and HTTPS

In HostNext DNS, create an **A record**:

| Host | Type | Value |
|---|---|---|
| `agricorps` | A | `217.77.10.180` |

Host Nginx site: `/etc/nginx/sites-available/agricorps.rapidsai.com.conf` (symlinked to `sites-enabled`). Proxies `agricorps.rapidsai.com` to `http://127.0.0.1:8085`.

```bash
nginx -t && systemctl reload nginx
dig +short agricorps.rapidsai.com
certbot --nginx -d agricorps.rapidsai.com
curl -I https://agricorps.rapidsai.com
```

Certbot configured HTTPS and automatic certificate renewal. Do not replace the host's existing Nginx configuration or other production sites.

## 7. Linux OCR fix

The repository contains a Windows `tesseract.exe`. In `pdf_ingestor.py`, set the executable path to Linux Tesseract:

```python
_DEFAULT_TESSERACT_PATH = "/usr/bin/tesseract"
if os.path.exists(_DEFAULT_TESSERACT_PATH):
    pytesseract.pytesseract.tesseract_cmd = _DEFAULT_TESSERACT_PATH
```

This fixes `Permission denied: '/app/tesseract.exe'` during PDF indexing. The Docker image installs Linux Tesseract.

## 8. Index the knowledge base

The initial ChromaDB collection was empty, so run:

```bash
docker compose exec backend python main.py --index
```

For indexing that can continue after disconnecting SSH (do not run two indexers at once):

```bash
nohup docker compose exec -T backend python main.py --index > indexing.log 2>&1 < /dev/null &
tail -f indexing.log  # Ctrl+C stops viewing logs, not indexing
```

After successful indexing:

```bash
docker compose restart backend
docker compose exec backend python -c "import vector_store; print('Documents:', vector_store.collection_size())"
```

**Status:** Indexing was started and the OCR path was corrected; final indexing completion was not confirmed in the deployment conversation.

## 9. Verify and maintain

```bash
docker compose ps
docker compose logs backend --tail=50
docker compose logs frontend --tail=50
docker compose exec backend python -c "import urllib.request; r=urllib.request.urlopen('http://127.0.0.1:8001/openapi.json'); print(r.status, r.headers.get('Content-Type'))"
```

> `/api/docs` returned 404; FastAPI's `/openapi.json` returned 200 **inside the backend container**. A public `/docs` or `/openapi.json` returning 200 alone may be the React fallback, not FastAPI docs.

After GitHub updates:

```bash
cd /root/RapidsAI-Portfolio/Agricorps-RAG
git pull
docker compose up -d --build
docker compose ps
```

After changing `.env`:

```bash
docker compose up -d --no-deps --force-recreate backend
```

**Caution:** This deployment bind-mounts the entire repository into `/app`; back up SQLite files, `chroma_db/`, and upload directories before code changes. The server has limited RAM, so indexing can be resource-intensive.
