# oneinchapps — Infrastruktur

Traefik-Reverse-Proxy-Setup für die oneinchapps-Produktwebsite (`oneinchapps.com`) und
deren Produkt-Subdomains (z. B. `headspace.oneinchapps.com`).

Dieses Repo enthält **nur die Infrastruktur** (Traefik + Routing). Die eigentlichen
Produkt-Apps (Landingpage, Headspace, ...) leben in eigenen Repos und werden hier
später als eigene Docker-Services eingebunden. Aktuell laufen an ihrer Stelle
minimale Platzhalter-Container (`nginx:alpine` + statisches HTML unter
[placeholders/](placeholders/)), damit das Routing sofort testbar ist.

## Struktur

```
docker-compose.yml       Basis: Netzwerk, Traefik, Platzhalter-Services
docker-compose.dev.yml   Override für lokale Entwicklung (HTTP, *.localhost, Dashboard offen)
docker-compose.prod.yml  Override für Produktion (HTTPS via Let's Encrypt, www-Redirect)
traefik/
  traefik.dev.yml        Statische Traefik-Config für dev
  traefik.prod.yml       Statische Traefik-Config für prod
  dynamic/                Dynamische Config (Middlewares), von Traefik live neu geladen
placeholders/
  landingpage/            Platzhalter für oneinchapps.com
  headspace/              Platzhalter für headspace.oneinchapps.com
letsencrypt/              ACME-Zertifikatsspeicher (nur prod, nicht in git)
```

## Lokale Entwicklung

Kein `/etc/hosts`-Eintrag nötig: `*.localhost` wird von Browsern und macOS
automatisch auf `127.0.0.1` aufgelöst.

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

Danach erreichbar:

- http://oneinchapps.localhost — Landingpage-Platzhalter
- http://headspace.oneinchapps.localhost — Headspace-Platzhalter
- http://localhost:8080 — Traefik-Dashboard (ungesichert, nur lokal)

Stoppen:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml down
```

**Hinweis (Docker Desktop für Mac):** `docker-compose.dev.yml` mountet den Docker-Socket
von `${HOME}/.docker/run/docker.sock` statt `/var/run/docker.sock` — Letzteres ist auf
macOS nur ein Symlink auf dem Host und lässt sich nicht 1:1 in den Container mounten.
Auf einem Linux-Server (Prod) ist `/var/run/docker.sock` ein echter Socket, daher bleibt
`docker-compose.prod.yml` beim Standardpfad.

## Produktion

Voraussetzungen:

1. Ein Server mit Docker, öffentlich unter einer IP erreichbar (Ports 80/443 offen).
2. DNS A-Records, die auf diesen Server zeigen:
   - `oneinchapps.com`
   - `www.oneinchapps.com`
   - `headspace.oneinchapps.com`
3. Eine `.env.prod` (aus dem Template kopieren und anpassen):

   ```bash
   cp .env.prod.example .env.prod
   ```

4. ACME-Storage vorbereiten (einmalig, korrekte Rechte für Let's Encrypt):

   ```bash
   touch letsencrypt/acme.json
   chmod 600 letsencrypt/acme.json
   ```

Start:

```bash
docker compose --env-file .env.prod -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Dabei passiert:

- HTTP (Port 80) wird permanent auf HTTPS umgeleitet.
- Traefik holt automatisch Let's Encrypt-Zertifikate per HTTP-01-Challenge.
- `www.oneinchapps.com` wird permanent auf `oneinchapps.com` umgeleitet.
- `headspace.oneinchapps.com` bekommt ein eigenes Zertifikat und Routing.

**Hinweis:** Das Traefik-Dashboard ist in `docker-compose.prod.yml` bewusst
*nicht* nach außen exponiert (kein Router/Port dafür). Falls gewünscht, muss es
später explizit mit Basic-Auth-Middleware und eigenem Router abgesichert werden.

## Neue Subdomain / neues Produkt hinzufügen

1. Neuen Service in `docker-compose.yml` mit eigenen Traefik-Labels ergänzen
   (`Host(\`produktname.${DOMAIN}\`)`, analog zu `landingpage`/`headspace`).
2. Für Prod-TLS in `docker-compose.prod.yml` die passenden
   `entrypoints`/`tls.certresolver`-Labels für den neuen Service ergänzen.
3. DNS-Record für die neue Subdomain anlegen.
