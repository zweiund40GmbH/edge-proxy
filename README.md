# Edge-Stack: Traefik und optionale Container-Updates

Eigenständiger Docker-Compose-Stack für labelbasiertes Reverse-Proxy-Routing, automatische TLS-Zertifikate und optionale Container-Updates. Die verwendeten Image-Versionen stehen in `compose.yml`.

## Zweck

Dieser Stack ist der zentrale HTTP-/HTTPS-Eingang eines Docker-Hosts. Traefik liest Routing-Labels aus den Anwendungscontainern, leitet Anfragen an deren interne Ports weiter und beschafft beziehungsweise erneuert TLS-Zertifikate automatisch.

Neue Anwendungen werden in ihrem eigenen Compose-Stack konfiguriert. Für gewöhnliche neue HTTP-Routen muss der Edge-Stack nicht angepasst werden. Containerwechsel und neue Container-IP-Adressen erkennt Traefik über Docker-Events.

Der Stack veröffentlicht TCP-Port 80 und 443. Anwendungen und ihre Routing-Regeln werden in separaten Compose-Projekten verwaltet. Ein Traefik-Dashboard wird in dieser Konfiguration nicht veröffentlicht.

## Komponenten und Netzwerke

| Bestandteil | Aufgabe |
| --- | --- |
| `traefik` | HTTP-/HTTPS-Routing, TLS-Terminierung, Zertifikate und Access-Logs |
| `socket-proxy` | Beschränkt Traefiks Zugriff auf die Docker-API auf lesende Anfragen |
| `watchtower` | Optional: ersetzt ausdrücklich freigegebene Container bei Image-Updates |
| `edge` | Externes gemeinsames Netzwerk für Traefik und veröffentlichte Anwendungsdienste |
| `docker-api` | Internes Netzwerk nur für Traefik und Socket-Proxy |
| `updater` | Separates Netzwerk mit ausgehendem Zugriff für Watchtower |
| `edge-certificates` | Persistentes Docker-Volume mit ACME-Konto und Zertifikaten |

Der Anfrageweg lautet: Client → Port 80/443 → Traefik → Anwendungscontainer im Netzwerk `edge`.

Traefik hat den festen Containernamen `edge-traefik`. Anwendungen, die die Client-Adresse aus `X-Forwarded-For` lesen, sollten nur diesem Absender vertrauen: Hostname `edge-traefik` (per Docker-DNS im Netzwerk `edge` auflösbar) statt einer IP-Adresse, die sich bei jedem Neustart ändern kann, und nicht das ganze Netzwerk `edge` – jeder Container darin könnte den Header sonst fälschen. Der Dienstname `traefik` ist dafür nicht geeignet, weil ihn jeder Stack im Netzwerk `edge` ebenfalls tragen kann. Der feste Name verhindert mehrere Traefik-Instanzen desselben Stacks auf einem Host.

Traefik greift über `socket-proxy:2375` auf die Docker-API zu. Der Socket-Proxy hat keinen veröffentlichten Host-Port und hängt nicht im Anwendungsnetz. `POST=0` beschränkt ihn auf GET/HEAD. Container-Metadaten können trotzdem sensible Informationen enthalten; dies ist keine vollständige Geheimnis-Isolation.

Watchtower benötigt zum Ersetzen von Containern Schreibzugriff und erhält deshalb den Docker-Socket direkt. Ein Socket-Mount mit `:ro` allein würde API-Schreiboperationen nicht verhindern.

Das externe Bridge-Netzwerk verbindet Compose-Projekte auf **demselben Docker-Host**. Es verbindet keine unterschiedlichen Server. Alle Dienste in `edge` können grundsätzlich miteinander kommunizieren. Interne Dienste wie Redis gehören normalerweise nur ins jeweilige Anwendungsnetz.

## Dateien und Voraussetzungen

Im Stack-Verzeichnis liegen `compose.yml`, `.env` und diese `README.md`. Alle folgenden Compose-Befehle werden dort ausgeführt.

Voraussetzungen:

- Docker Engine und Docker Compose mit Unterstützung für Profile.
- Freie TCP-Ports 80 und 443 auf dem Host.
- DNS-Einträge der veröffentlichten Domains zeigen auf den Server. Vorhandene AAAA-Einträge müssen ebenfalls korrekt erreichbar sein.
- Firewall/NAT erlauben eingehendes HTTP und HTTPS.
- Für Watchtower: eine vorhandene Docker-Registry-Konfiguration mit passenden Zugangsdaten.

Beispiel für `.env`:

```dotenv
LETSENCRYPT_EMAIL=admin@example.com
TRAEFIK_LOG_LEVEL=INFO
WATCHTOWER_POLL_INTERVAL=300
DOCKER_AUTH_CONFIG=/root/.docker/config.json
WATCHTOWER_NOTIFICATION_URL=
```

Die E-Mail-Adresse anpassen. Die Benachrichtigungs-URL ist optional, beispielsweise eine ntfy-URL. Zugangsdaten und private Benachrichtigungs-URLs nicht ins Git-Repository einchecken. Eine `.env.example` mit Platzhaltern kann dagegen versioniert werden.

## Verwendung in einem öffentlichen Repository

Versionieren lassen sich `compose.yml`, diese README und eine `.env.example` mit neutralen Platzhaltern. Lokale Konfiguration, Zugangsdaten und Backups bleiben außerhalb des Repositorys.

Beispiel für `.gitignore`:

```gitignore
.env
.env.*
!.env.example
backups/
acme.json
```

Die Docker-Registry-Konfiguration liegt außerhalb des Projektverzeichnisses. Zertifikate werden im Docker-Volume gespeichert. `.gitignore` entfernt keine bereits versionierten Dateien; vor der Veröffentlichung auch die Git-Historie auf versehentlich eingecheckte Zugangsdaten prüfen.

Alle Domains unter `example.com` in dieser Dokumentation sind Platzhalter und müssen durch eigene Domains ersetzt werden.

## Erststart

Gemeinsames Netzwerk einmalig anlegen, falls es noch nicht existiert:

```bash
docker network inspect edge
docker network create edge
```

Den zweiten Befehl nur ausführen, wenn das Netzwerk fehlt. Danach:

```bash
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=100 traefik socket-proxy
```

Die Ports 80 und 443 dürfen nicht bereits durch einen anderen Dienst belegt sein. Anwendungsnetzwerke und Routing-Labels werden im jeweiligen Anwendungs-Stack eingerichtet. Ohne passende Router antwortet Traefik mit 404; ein Catch-all-Backend ist nicht vorkonfiguriert.

`docker compose config --quiet` prüft die Compose-Struktur und Variablenauflösung. Es prüft weder Traefik-Regeln noch die Erreichbarkeit von Backends oder Zertifikatsausstellung.

## Eine Anwendung veröffentlichen

Im Anwendungs-Stack das externe Netzwerk deklarieren und den veröffentlichten Dienst damit verbinden. Beispiel als Ergänzung eines bestehenden Dienstes; Image, Volumes und weitere Einstellungen bleiben dort erhalten:

```yaml
services:
  dashboard:
    networks:
      - backend
      - edge
    labels:
      traefik.enable: "true"
      traefik.docker.network: edge

      traefik.http.routers.example-web.rule: "Host(`${DASHBOARD_HOST:-app.example.com}`)"
      traefik.http.routers.example-web.entrypoints: websecure
      traefik.http.routers.example-web.tls: "true"
      traefik.http.routers.example-web.tls.certresolver: le
      traefik.http.routers.example-web.priority: "100"
      traefik.http.routers.example-web.service: example-web

      traefik.http.services.example-web.loadbalancer.server.port: "8080"

      com.centurylinklabs.watchtower.enable: "${AUTOUPDATE_DASHBOARD:-false}"

networks:
  backend:
  edge:
    external: true
    name: edge
```

Wichtige Regeln:

- `traefik.enable=true` ist erforderlich, weil automatische Veröffentlichung standardmäßig deaktiviert ist.
- Router wählen Anfragen aus; Services definieren das Backend; Middlewares verändern Anfragen oder Antworten.
- Der Service-Port ist der **interne Container-Port**, nicht ein veröffentlichter Host-Port. Der Prozess muss auf einer im Container erreichbaren Adresse lauschen, nicht nur auf `127.0.0.1`.
- Für HTTP-Dienste hinter Traefik ist normalerweise kein `ports:` im Anwendungs-Stack nötig. Bestehende direkte Zugriffe vor dem Entfernen prüfen.
- Router-, Service- und Middleware-Namen über alle Stacks eindeutig vergeben, beispielsweise mit `project-a-` oder `project-b-` als Präfix.
- Bei mehreren passenden Regeln Prioritäten ausdrücklich setzen: höhere Zahl gewinnt. Ohne explizite Priorität entscheidet normalerweise die Regellänge.
- `${...}` wird von Compose ersetzt. Traefik liest die resultierenden Labels, nicht beliebige Environment-Variablen der Anwendung.
- Nach Änderungen an Labels oder `.env` im jeweiligen Anwendungs-Stack `docker compose up -d` ausführen. `restart` übernimmt keine geänderte Container-Konfiguration.

## HTTP, HTTPS und besondere Protokolle

Es gibt absichtlich keinen globalen HTTP→HTTPS-Redirect. Jede Anwendung entscheidet über eigene Router, ob HTTP erlaubt oder auf HTTPS umgeleitet wird. Ein HTTPS-Router erzeugt nicht automatisch einen HTTP-Router.

Für eine Anwendung, die HTTP auf HTTPS umleiten soll, zusätzlich folgende Labels an deren Dienst setzen:

```yaml
traefik.http.routers.example-web-http.rule: "Host(`${DASHBOARD_HOST:-app.example.com}`)"
traefik.http.routers.example-web-http.entrypoints: web
traefik.http.routers.example-web-http.service: example-web
traefik.http.routers.example-web-http.middlewares: example-web-https
traefik.http.middlewares.example-web-https.redirectscheme.scheme: https
traefik.http.middlewares.example-web-https.redirectscheme.permanent: "true"
```

Soll HTTP direkt funktionieren, einen separaten Router auf `web` ohne TLS und ohne Redirect-Middleware mit dem gewünschten Service verbinden.

WebSocket-Upgrades werden unterstützt. Für MQTT über WebSocket muss der Router auf den im Broker konfigurierten WebSocket-Port zeigen. Natives MQTT auf Port `1883` ist davon unabhängig: Dieser Edge-Stack veröffentlicht keinen MQTT-TCP-Eingang. Natives MQTT erfordert eine separate Portfreigabe oder zusätzliche TCP-Router und EntryPoints.

Für einen gRPC-Dienst, der intern unverschlüsseltes HTTP/2 auf Port 50051 spricht, den Service beispielsweise so konfigurieren:

```yaml
traefik.http.services.example-grpc.loadbalancer.server.port: "50051"
traefik.http.services.example-grpc.loadbalancer.server.scheme: h2c
```

Der zugehörige Router muss auf diesen Service verweisen und den gRPC-Pfad erhalten. Streaming/SSE und lange Verbindungen separat testen und Zeitlimits entsprechend den Anforderungen der Anwendung konfigurieren.

## Weitere Benutzungsbeispiele

### Vollständiger kleiner Test-Stack

Dieses Beispiel in einem separaten Verzeichnis als `compose.yml` speichern. `test.example.com` durch eine eigene Domain ersetzen, deren DNS auf den Docker-Host zeigt. Es veröffentlicht absichtlich einen Diagnose-Dienst; nach dem Test wieder entfernen.

```yaml
name: edge-test

services:
  whoami:
    image: traefik/whoami:v1.11.0
    restart: unless-stopped
    networks:
      - edge
    labels:
      traefik.enable: "true"
      traefik.docker.network: edge
      traefik.http.routers.edge-test.rule: "Host(`test.example.com`)"
      traefik.http.routers.edge-test.entrypoints: websecure
      traefik.http.routers.edge-test.tls: "true"
      traefik.http.routers.edge-test.tls.certresolver: le
      traefik.http.routers.edge-test.service: edge-test
      traefik.http.services.edge-test.loadbalancer.server.port: "80"
      com.centurylinklabs.watchtower.enable: "false"

networks:
  edge:
    external: true
    name: edge
```

```bash
docker compose up -d
curl -v https://test.example.com/
# Nach dem Test, im Verzeichnis dieses Test-Stacks:
docker compose down
```

Das erste Zertifikat kann einen Moment benötigen. Der Testdienst zeigt Request-Informationen an und ist keine produktive Anwendung.

### API und Frontend unter derselben Domain

Zusätzlich zum Dashboard-Beispiel bekommt der vorhandene `api`-Dienst das Netzwerk `edge` und diese Labels:

```yaml
traefik.enable: "true"
traefik.docker.network: edge
traefik.http.routers.example-api.rule: "Host(`app.example.com`) && PathPrefix(`/api`)"
traefik.http.routers.example-api.entrypoints: websecure
traefik.http.routers.example-api.tls: "true"
traefik.http.routers.example-api.tls.certresolver: le
traefik.http.routers.example-api.priority: "200"
traefik.http.routers.example-api.service: example-api
traefik.http.services.example-api.loadbalancer.server.port: "8080"
```

Ergebnis: `/api/...` geht an Port 8080 der API, andere Pfade an das Dashboard mit Priorität 100. Der Pfad bleibt vollständig erhalten. ``PathPrefix(`/api`)`` trifft auch `/apix`; für eine strikte Pfadgrenze stattdessen ``(Path(`/api`) || PathPrefix(`/api/`))`` verwenden.

Das Beispiel beschränkt die API auf eine konkrete Domain. Für einen Präfixvergleich ohne Beachtung der Groß-/Kleinschreibung kann stattdessen ``PathRegexp(`(?i)^/api`)`` verwendet werden.

### Einen Pfad-Präfix vor dem Backend entfernen

Wenn ein Dienst öffentlich unter `/tool/` erreichbar sein soll, intern aber `/` erwartet, am entsprechenden Router eine Middleware zuweisen:

```yaml
traefik.http.routers.mein-tool.middlewares: mein-tool-strip
traefik.http.middlewares.mein-tool-strip.stripprefix.prefixes: /tool
```

Damit wird beispielsweise `/tool/status` zu `/status`. Das allein repariert keine absoluten Links oder Weiterleitungen der Anwendung; diese muss gegebenenfalls ihre externe Basis-URL kennen. Den Präfix nicht entfernen, wenn das Backend selbst `/api/...` erwartet.

### Updates pro Anwendung per .env steuern

Im Anwendungsdienst:

```yaml
labels:
  com.centurylinklabs.watchtower.enable: "${AUTOUPDATE_API:-false}"
```

In der `.env` dieses Anwendungs-Stacks:

```dotenv
AUTOUPDATE_API=true
```

Danach im Anwendungs-Stack:

```bash
docker compose up -d api
```

Voraussetzung: Watchtower läuft im Edge-Stack mit dem Profil `updates`. Mit `AUTOUPDATE_API=false` und erneutem `up -d api` wird die Freigabe wieder entfernt.

## Automatische Zertifikate

Der Resolver heißt `le` und verwendet Let's Encrypt mit HTTP-01. Ein HTTPS-Router aktiviert ihn mit `tls.certresolver=le`; die Domain stammt normalerweise aus seiner `Host(...)`-Regel. Reine Pfad- oder Catch-all-Regeln liefern keine konkrete Zertifikatsdomain.

Traefik bedient die ACME-Challenge auf Port 80 selbst. Ein zusätzlicher ACME-Client ist für diese Router nicht erforderlich. Ausstellung und Erneuerung erfolgen automatisch, solange DNS, Erreichbarkeit und persistenter Speicher funktionieren.

`acme.json` liegt unter `/letsencrypt/acme.json` im Volume `edge-certificates` und enthält private Schlüssel. Traefik legt die Datei an; sie benötigt restriktive Rechte (0600). Das Volume bleibt bei Container-Neuerstellung und normalem `docker compose down` erhalten.

Nicht routinemäßig das Volume löschen oder `docker compose down -v` verwenden. Wiederholte Neuausstellung kann ACME-Limits auslösen. Für Experimente einen separaten Staging-Resolver und getrennten Speicher verwenden. Wildcards benötigen DNS-01; das ist hier nicht eingerichtet.

## Watchtower: gezielte automatische Updates

Watchtower ist optional und gehört zum Compose-Profil `updates`:

```bash
docker compose --profile updates up -d
docker compose --profile updates logs --tail=100 watchtower
```

Vorher eine bestehende Watchtower-Instanz auf demselben Host stoppen, damit nicht zwei Instanzen dieselben Container verwalten.

Freigabe im Anwendungsdienst:

```yaml
labels:
  com.centurylinklabs.watchtower.enable: "true"
```

Ohne Freigabe wird ein Container nicht regulär für automatische Updates ausgewählt. Traefik, Socket-Proxy und Watchtower selbst sind in der Edge-Compose ausdrücklich ausgeschlossen. Der Updater arbeitet über Compose-Projektgrenzen hinweg auf dem gesamten Docker-Host.

Die Registry-Datei wird als `/config.json` eingebunden und muss vorhanden sein. Eine Datei, die lediglich einen auf dem Host installierten Credential-Helper referenziert, reicht im Container nicht zwangsläufig aus.

Watchtower prüft das konfigurierte Image/Tag auf eine neue Image-Version. Ein festes Tag wie `1.23.1` wird dadurch nicht automatisch auf `1.24.0` umgeschrieben. Für eigene Anwendungen können bewusst aktualisierte Deployment-Tags genutzt werden. Watchtower aktualisiert keine Compose-Dateien und übernimmt keine Änderungen an deren Labels oder Umgebungsvariablen.

Container-Austausch kann laufende MQTT-, gRPC- und SSE-Verbindungen unterbrechen. Automatische Updates sind kein garantiert unterbrechungsfreies Deployment. `WATCHTOWER_CLEANUP=true` kann alte Images entfernen; deshalb für Rollbacks die bisherige Version auch in der Registry verfügbar halten.

Watchtower ausdrücklich anhalten:

```bash
docker compose --profile updates stop watchtower
```

Das Weglassen von `--profile updates` bei einem späteren `up -d` beendet eine bereits laufende Instanz nicht automatisch.

## Betrieb und manuelle Updates

```bash
# Status und Logs
docker compose ps
docker compose logs -f --tail=100 traefik
docker compose logs --tail=100 socket-proxy

# Traefik-interner Healthcheck
docker compose exec traefik traefik healthcheck --ping

# Nach einer bewussten Änderung der Image-Versionen
docker compose pull traefik socket-proxy
docker compose up -d traefik socket-proxy

# Optional auch Watchtower kontrolliert aktualisieren
docker compose --profile updates pull watchtower
docker compose --profile updates up -d watchtower
```

Bei fest gepinnten Images zuerst das gewünschte Tag in `compose.yml` ändern. Der Socket-Proxy verwendet in der Ausgangskonfiguration `latest`: ein bewusstes `pull` kann dort eine neue Version laden. Für vollständig reproduzierbare Deployments auch diesen auf ein geprüftes Tag oder einen Digest festlegen.

Der Healthcheck prüft Traefiks Ping-Endpunkt, nicht sämtliche Backends und Zertifikate. Docker Compose startet einen lediglich als `unhealthy` markierten Container nicht allein aufgrund dieses Zustands automatisch neu.

Access- und Container-Logs werden durch Docker auf fünf Dateien à 50 MB je Dienst begrenzt. Access-Logs können URLs einschließlich sensibler Query-Parameter enthalten. Bei Weiterleitung oder Langzeitarchivierung berücksichtigen.

## Backup und Wiederherstellung

Sichern: `compose.yml`, die geschützte `.env`, gegebenenfalls Registry-Zugangsdaten sowie das Volume `edge-certificates`. Anwendungsdaten benötigen separate Backups.

Beispiel für ein konsistentes Zertifikatsbackup mit kurzer Proxy-Unterbrechung; im Stack-Verzeichnis ausführen:

```bash
mkdir -p backups
chmod 700 backups
docker pull alpine:3.22
docker compose stop traefik
docker run --rm \
  -v edge-certificates:/data:ro \
  -v "$PWD/backups:/backup" \
  alpine:3.22 sh -c 'umask 077; tar -czf /backup/edge-certificates-$(date -u +%Y%m%dT%H%M%SZ).tar.gz -C /data .'
docker compose start traefik
```

Den Erfolg des Archivs prüfen und es verschlüsselt außerhalb dieses Servers sichern. Auch bei einem Backup-Fehler Traefik wieder starten. Das Archiv enthält private Schlüssel.

Wiederherstellung auf einem Ersatzhost, bevor Traefik startet:

1. Compose-Dateien und `.env` wiederherstellen.
2. Netzwerk `edge` und Volume `edge-certificates` anlegen.
3. Ein geprüftes Backup nach `backups/edge-certificates-restore.tar.gz` kopieren.
4. In das leere Volume entpacken:

```bash
docker volume create edge-certificates
docker run --rm \
  -v edge-certificates:/data \
  -v "$PWD/backups:/backup:ro" \
  alpine:3.22 sh -c 'tar -xzf /backup/edge-certificates-restore.tar.gz -C /data && chmod 600 /data/acme.json'
```

5. DNS/NAT prüfen, Anwendungs-Stacks wiederherstellen und Edge starten.

Nicht in ein aktiv beschriebenes Zertifikatsvolume zurückspielen. Diese lokale ACME-Datei nicht gleichzeitig durch mehrere Traefik-Instanzen beschreiben lassen.

## Fehlersuche

| Symptom | Häufige Ursache / Prüfung |
| --- | --- |
| Port 80/443 bereits belegt | Ein anderer Reverse Proxy oder Webserver verwendet die Ports |
| Netzwerk `edge` fehlt | Externes Netzwerk vor dem Compose-Start anlegen |
| 404 von Traefik | Keine passende Router-Regel; Host, EntryPoint, Labels und `traefik.enable` prüfen |
| 502/504 | Backend-Port falsch, Prozess nicht erreichbar, falsches Netzwerk oder falsches HTTP/h2c-Schema |
| Standardzertifikat / TLS-Warnung | Zertifikat noch nicht ausgestellt; DNS, Port 80, Resolver und Traefik-Logs prüfen |
| Docker-API meldet 403 | Freigaben und Logs des Socket-Proxys prüfen; nicht pauschal Schreibzugriff aktivieren |
| Änderungen an `.env` greifen nicht | Im betroffenen Stack `docker compose up -d` statt nur `restart` ausführen |
| Anwendung landet im falschen Backend | Überlappende Regeln und Prioritäten prüfen |
| Watchtower startet nicht | Profil aktiviert? Registry-Datei vorhanden und tatsächlich eine Datei? |
| Watchtower aktualisiert nicht | Label, Registry-Anmeldung, Image-Tag und Logs prüfen |

Netzwerkzugehörigkeit prüfen:

```bash
docker network inspect edge
```

## Referenzen

- [Traefik Docker Provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/)
- [Traefik Routing-Regeln und Prioritäten](https://doc.traefik.io/traefik/reference/routing-configuration/http/routing/rules-and-priority/)
- [Traefik ACME / Let's Encrypt](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/)
- [Docker-Socket-Proxy](https://github.com/Tecnativa/docker-socket-proxy)
- [Watchtower-Fork](https://github.com/nicholas-fedor/watchtower)
- [Compose-Netzwerke](https://docs.docker.com/reference/compose-file/networks/)

