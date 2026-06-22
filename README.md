# Secure Event Ticketing Platform (Sample DevSecOps Project)

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.
Prikazuje cijeli tok: lokalni razvoj kroz Compose i produkcijski deployment kroz Kubernetes manifeste.

## Arhitektura

- `frontend` - web UI za pregled evenata i kupnju karata
- `api` - REST API za evente, narudzbe i health provjere
- `worker` - pozadinska obrada queue poruka
- `postgres` - trajna pohrana narudzbi
- `redis` - queue/cache sloj

## Lokalni razvoj (Docker Compose)

Preduvjeti: Docker i Docker Compose.

### Pokretanje (startup)

```bash
cp .env.example .env          # jednom, prilagodi vrijednosti
docker compose up -d --build  # build + pokretanje cijelog stacka
```

Servisi:

- Frontend UI: `http://localhost:3000`
- API: `http://localhost:8080`

### Praćenje i status

```bash
docker compose ps             # status servisa
docker compose logs -f        # logovi
```

### Zaustavljanje (shutdown)

```bash
docker compose down           # zaustavi i ukloni kontejnere (podaci baze ostaju)
docker compose down -v        # dodatno ukloni volume (BRISE podatke baze)
```

### Brza validacija funkcionalnosti

1. Health API:
   ```bash
   curl http://localhost:8080/healthz
   curl http://localhost:8080/readyz
   ```
2. Dohvati evente:
   ```bash
   curl http://localhost:8080/events
   ```
3. Posalji narudzbu:
   ```bash
   curl -X POST http://localhost:8080/tickets/purchase \
     -H "Content-Type: application/json" \
     -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
   ```
4. Provjeri obradene narudzbe:
   ```bash
   curl http://localhost:8080/tickets/orders
   ```
5. UI:
   - Otvori `http://localhost:3000`

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik
- Secret + ConfigMap odvojena konfiguracija
- Liveness/Readiness probe
- Resource requests/limits
- ServiceAccount + RBAC
- NetworkPolicy segmentacija
- Trivy skeniranje slika u CI pipelineu

Izvještaji skeniranja slika (Trivy, generirani u CI-u) nalaze se u `docs/security/`:

- `trivy-api-report.txt`
- `trivy-frontend-report.txt`
- `trivy-worker-report.txt`

## Produkcijski deployment i troubleshooting

- Upute za produkcijski deployment (Kubernetes): [`infra/k8s/README.md`](infra/k8s/README.md)
- Runbook za troubleshooting: sekcija
  **Incident runbook** u istom dokumentu.
