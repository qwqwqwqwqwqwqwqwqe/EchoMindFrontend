# EchoMind Frontend

Vue 3 + Vite interface for the [EchoMind Python backend](https://github.com/qwqwqwqwqwqwqwqe/EchoMind). It also contains a switch for the separately maintained Java backend.

## What you can do

- Chat with the selected backend, revisit conversations, inspect agent/tool traces and view health/monitor data.
- Search and upload knowledge documents and run the backend's evaluation endpoint.
- Review generated routing examples: edit intent and primary/supporting-agent labels, record quality decisions, and export a JSONL audit backup.
- Review source passages alongside the actual indexed ChromaDB chunks: inspect source/version evidence, use a deterministic per-source sample, record issues, and export/import a snapshot-bound JSON audit.

Review records in the browser are working copies. Export them before clearing browser storage or changing datasets. The passage review is a **passage-level** decision covering its displayed chunks; it is not an independent vote for every chunk.

## Local development

Use Node.js 22 or newer. From this directory:

```powershell
npm ci
npm run dev
```

Open `http://127.0.0.1:5173/`. Vite proxies `/api/python` to `http://127.0.0.1:8000` and `/api/java` to `http://127.0.0.1:8080`. Start the relevant backend first. Build and test with `npm run build` and `npm test`.

## Review data

The app loads generated routing candidates from `public/routing-candidates.jsonl` and indexed passages from `public/passage-review.json`. These generated files contain local review data and are intentionally excluded from Git.

To recreate the passage snapshot, first index the official source documents in the sibling Python repository, then run from its root:

```powershell
.\env\Scripts\python.exe -m evaluation.dataset.export_passage_review
```

The exporter checks source hashes, passage IDs, chunk versions and counts before writing the frontend asset. A new index changes the snapshot identity, so old browser decisions cannot silently attach to new chunks. To review routing examples, generate or import a JSONL file in the routing review tab. The backend repository explains the experimental status and limits of its pilot data.

## Docker

The supplied `docker-compose.yml` builds a full Python + Java + frontend stack and uses Nginx for routing. It requires the sibling `EchoMind` and `EchoMindJava` directories and their local `.env` files. From this directory:

```powershell
docker compose --env-file ../EchoMind/.env up -d --build
```

The default standalone frontend port is 5174. Docker data volumes are separate from a host-local ChromaDB path; transfer or rebuild indexes explicitly when switching modes.
