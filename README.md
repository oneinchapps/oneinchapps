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
docker-compose.prod.yml  Override für Produktion (HTTPS via Datei-Zertifikat, www-Redirect)
traefik/
  traefik.dev.yml        Statische Traefik-Config für dev
  traefik.prod.yml       Statische Traefik-Config für prod
  dynamic/
    middlewares.yml       Beispiel-Middlewares (compress, security-headers)
    tls.yml                Verweist auf certs/fullchain.pem + certs/privkey.pem
placeholders/
  landingpage/            Platzhalter für oneinchapps.com
  headspace/              Platzhalter für headspace.oneinchapps.com
certs/                    Zertifikat + Private Key für prod (nicht in git)
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

## Produktion (IONOS VPS, eigenes Wildcard-Zertifikat)

`docker-compose.prod.yml` nutzt **kein** Let's Encrypt/ACME, sondern ein
gekauftes Zertifikat als Datei (z. B. IONOS-SSL-Paket). Traefik liest es über
`traefik/dynamic/tls.yml` aus `certs/fullchain.pem` + `certs/privkey.pem` und
setzt es als Default-Zertifikat für den `websecure`-Entrypoint — damit sind
`oneinchapps.com`, `www.oneinchapps.com`, `headspace.oneinchapps.com` und
alle künftigen `*.oneinchapps.com`-Subdomains automatisch abgedeckt, solange
das Zertifikat ein Wildcard ist.

### 1. DNS einrichten

Im DNS-Management der Domain (IONOS-Kundencenter oder wo `oneinchapps.com`
registriert ist) A-Records (und falls verfügbar AAAA für IPv6) auf die
VPS-IP anlegen:

- `oneinchapps.com`
- `www.oneinchapps.com`
- `headspace.oneinchapps.com`
- optional gleich `*.oneinchapps.com` als Wildcard-Record, damit künftige
  Produkt-Subdomains ohne weiteren DNS-Eintrag funktionieren

DNS-Propagation kann bis zu ein paar Stunden dauern — vor dem nächsten
Schritt mit `dig oneinchapps.com` bzw. `dig headspace.oneinchapps.com`
prüfen, ob die IP schon ankommt.

### 2. Firewall auf dem VPS

Ports 80 und 443 müssen von außen erreichbar sein — sowohl in der
IONOS-Firewall/Security-Group (falls im Cloud Panel konfiguriert) als auch
in einer lokalen Firewall auf dem Server (z. B. `ufw allow 80,443/tcp`).

### 3. Repo + Zertifikat auf den Server bringen

Da dieses Repo bislang nur lokal existiert (kein Git-Remote), am einfachsten
per `rsync` übertragen (aus dem lokalen Repo-Verzeichnis heraus, `.git`
weglassen):

```bash
rsync -avz --exclude '.git' ./ user@<vps-ip>:/opt/oneinchapps/
```

Die drei Dateien aus dem IONOS-SSL-Paket (Zertifikat, Private Key,
CA-Bundle/Intermediate-Zertifikate) auf den Server kopieren, z. B. nach
`/opt/oneinchapps/certs/`:

```bash
scp domain.crt privkey.key ca-bundle.crt user@<vps-ip>:/opt/oneinchapps/certs/
```

Auf dem Server Zertifikat + CA-Bundle zu einer Fullchain zusammenführen
(Reihenfolge: erst das eigene Zertifikat, danach die Intermediates) und
Rechte setzen:

```bash
ssh user@<vps-ip>
cd /opt/oneinchapps/certs
cat domain.crt ca-bundle.crt > fullchain.pem
mv privkey.key privkey.pem
chmod 600 privkey.pem
chmod 644 fullchain.pem
rm domain.crt ca-bundle.crt
```

`certs/` ist in `.gitignore` — Zertifikat und Key landen nie im Git-Repo.

### 4. `.env.prod` anlegen

```bash
cd /opt/oneinchapps
cp .env.prod.example .env.prod
# DOMAIN=oneinchapps.com steht schon drin, bei Bedarf anpassen
```

### 5. Starten

```bash
docker compose --env-file .env.prod -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Dabei passiert:

- HTTP (Port 80) wird permanent auf HTTPS umgeleitet.
- `websecure` (Port 443) terminiert TLS mit dem Zertifikat aus `certs/`.
- `www.oneinchapps.com` wird permanent auf `oneinchapps.com` umgeleitet.
- `headspace.oneinchapps.com` läuft über dasselbe Wildcard-Zertifikat.

### 6. Prüfen

```bash
curl -vI https://oneinchapps.com/ 2>&1 | grep -i "subject\|HTTP/"
curl -vI https://headspace.oneinchapps.com/ 2>&1 | grep -i "subject\|HTTP/"
```

**Hinweis Zertifikats-Laufzeit:** Anders als bei Let's Encrypt gibt es hier
*keine* automatische Erneuerung. Gekaufte Zertifikate laufen meist 1 Jahr —
rechtzeitig vor Ablauf im IONOS-Kundencenter erneuern und die neuen Dateien
per Schritt 3 austauschen (`docker compose ... restart traefik` danach).

**Hinweis Dashboard:** Das Traefik-Dashboard ist in `docker-compose.prod.yml`
bewusst *nicht* nach außen exponiert (kein Router/Port dafür). Falls
gewünscht, muss es später explizit mit Basic-Auth-Middleware und eigenem
Router abgesichert werden.

## Neue Subdomain / neues Produkt hinzufügen

1. Neuen Service in `docker-compose.yml` mit eigenen Traefik-Labels ergänzen
   (`Host(\`produktname.${DOMAIN}\`)`, analog zu `landingpage`/`headspace`).
2. Für Prod-TLS in `docker-compose.prod.yml` die passenden
   `entrypoints`/`tls`-Labels für den neuen Service ergänzen (siehe
   `headspace` als Vorlage) — dank Wildcard-Zertifikat reicht das, ohne
   dass ein neues Zertifikat besorgt werden muss.
3. DNS-Record für die neue Subdomain anlegen (entfällt, falls bereits ein
   `*.oneinchapps.com`-Wildcard-Record existiert).
