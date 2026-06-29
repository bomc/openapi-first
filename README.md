# OpenAPI Spring Boot

## Working on your OpenAPI Definition

### Install

1. Install [Node JS](https://nodejs.org/).
2. Clone this repo and run `npm install` in the repo root.

### Usage

#### `npm start`
Starts the reference docs preview server.

#### `npm run build`
Bundles the definition to the dist folder.

#### `npm test`
Validates the definition.


```yaml
bomc:
  $example: ./for.code
```
---

# Gravitee API Gateway – Integration Styleguide

**Verbindliche Richtlinien für die Integration von REST-APIs**

| | |
|---|---|
| **Deployment-Modell** | Hybrid (Gateway on-premise, Control Plane SaaS) |
| **API-Typen** | REST/HTTP |
| **Zielgruppen** | API-Producer-Teams, Platform/Gateway-Admins, Security & Compliance |
| **Gravitee-Version** | APIM 4.x |
| **Version** | 1.4 |

---

## 1  Einleitung

Dieser Styleguide legt verbindliche Regeln und Empfehlungen für die Integration von REST-APIs über das Gravitee API Management Gateway fest.

Er **ergänzt den übergeordneten REST API Styleguide** [`<Platzhalter: Link zum REST API Styleguide>`] um Gateway-spezifische Vorgaben. Bei Konflikten gilt:
- der **REST API Styleguide** für das API-Design selbst (URL-Struktur, Payloads, Statuscodes …)
- **dieses Dokument** für die Gateway-Konfiguration

### 1.1  Zeichenerklärung

| Symbol | Bedeutung | Konsequenz bei Verstoß |
|---|---|---|
| ⚑ **PFLICHT** | Verbindliche Anforderung | API wird nicht deployt / Onboarding blockiert |
| ✓ **EMPFOHLEN** | Best Practice | Begründung bei Abweichung erforderlich |
| ℹ **INFO** | Hinweis / Erläuterung | Keine |

---

## 2  Naming & Metadaten

### 2.1  API-Name

⚑ **PFLICHT** – API-Name folgt dem Schema: `<domäne>-<ressource>-v<major>`

| Feld | Beispiel | Erlaubt | Nicht erlaubt |
|---|---|---|---|
| Domäne | `order` | Kleinbuchstaben, Bindestriche | Leerzeichen, Großbuchstaben |
| Ressource | `shipments` | Plural-Substantiv | Verben (z. B. `getOrders`) |
| Version | `v2` | `v` + Integer | `v2.1`, `2`, `V2` |
| Vollständig | `order-shipments-v2` | – | `Order Shipments`, `orderShipmentsV2` |

### 2.2  Context-Path

⚑ **PFLICHT** – Context-Path enthält die Hauptversion: `/<domäne>/<ressource>/v<major>`

```
/order/shipments/v2
```

✓ **EMPFOHLEN** – Kein trailing slash; nur Kleinbuchstaben und Bindestriche.

ℹ Detailregeln zu URL-Design (Pluralform, Query-Parameter, Filter etc.) sind im REST API Styleguide geregelt [`<Link zum REST API Styleguide>`].

### 2.3  Beschreibung & Dokumentation

- ⚑ **PFLICHT** – Beschreibung in der APIM-Konsole hinterlegt (min. 2 Sätze).
- ⚑ **PFLICHT** – OpenAPI-Spezifikation (OAS 3.x) als Dokumentation importiert oder verlinkt.
- ✓ **EMPFOHLEN** – Kontaktinformationen des verantwortlichen Teams (E-Mail oder Slack-Channel).

### 2.4  Labels & Tags

⚑ **PFLICHT** – Folgende Labels sind für jede API verpflichtend:

| Label-Key | Beispielwert | Pflicht | Zweck |
|---|---|---|---|
| `team` | `checkout-squad` | Ja | Zuordnung Producer-Team |
| `domain` | `order` | Ja | Fachliche Domäne |
| `sla-tier` | `bronze` / `silver` / `gold` | Ja | SLA-Klasse (siehe Kap. 4) |
| `environment` | `dev` / `staging` / `prod` | Ja | Deployment-Stage |
| `data-classification` | `internal` / `confidential` / `public` | Ja | Datenschutz |
| `lifecycle` | `active` / `deprecated` / `sunset` | Nein | Lifecycle-Status |

---

## 3  Sicherheit & Authentifizierung

### 3.1  Plans und Authentifizierung

- ⚑ **PFLICHT** – Jede API muss mindestens einen Plan mit Authentifizierung besitzen.
- ⚑ **PFLICHT** – Keyless-Plans sind in Produktionsumgebungen verboten.

| Methode | Einsatzgebiet | Bewertung |
|---|---|---|
| OAuth2 / JWT | Standard für externe & interne APIs | **Bevorzugt** |
| API Key | Einfache M2M-Szenarien | Akzeptiert |
| mTLS | Hochsicherheits-Integrationen | Akzeptiert |
| Keyless | Nur Dev-Sandbox mit expliziter Freigabe | Ausnahme |

### 3.2  JWT-Konfiguration

- ⚑ **PFLICHT** – Signature-Algorithmus: `RS256` oder `ES256` (kein `HS256` in Produktion).
- ⚑ **PFLICHT** – Token-Expiry prüfen (`exp`-Claim).
- ⚑ **PFLICHT** – Issuer (`iss`) und Audience (`aud`) müssen validiert werden.
- ✓ **EMPFOHLEN** – JWKS-Endpoint statt statischem Public Key.

### 3.3  OAuth2-Konfiguration

- ⚑ **PFLICHT** – Token Introspection Endpoint über HTTPS.
- ⚑ **PFLICHT** – Scopes auf Plan-Ebene dokumentiert.
- ✓ **EMPFOHLEN** – Access Token Cache aktivieren (Cache TTL < Token Expiry).

### 3.4  Subscription-Prozess

- ⚑ **PFLICHT** – Auto-Validierung von Subscriptions deaktivieren; manuelle Prüfung durch API-Owner.
- ⚑ **PFLICHT** – Subscription-Kommentar als Pflichtfeld aktivieren (Begründung des Konsumenten).
- ✓ **EMPFOHLEN** – Subscriptions mindestens quartalsweise reviewen und verwaiste Subscriptions widerrufen.
- ✓ **EMPFOHLEN** – Benachrichtigungen für neue Subscription-Anfragen an das Producer-Team konfigurieren.

### 3.5  CORS

- ⚑ **PFLICHT** – CORS auf API-Ebene konfigurieren; **niemals** Wildcard (`*`) in Produktionsumgebungen.
- ⚑ **PFLICHT** – Erlaubte Origins explizit whitelist-basiert pflegen.
- ✓ **EMPFOHLEN** – Allowed Methods auf das notwendige Minimum beschränken.

```
Access-Control-Allow-Origin: https://app.example.com
```

### 3.6  TLS

- ⚑ **PFLICHT** – Alle Backend-Verbindungen (Endpoint) über HTTPS/TLS 1.2+.
- ⚑ **PFLICHT** – Self-signed Certificates nur in Dev/Staging; in Produktion nur CA-signierte Zertifikate.
- ✓ **EMPFOHLEN** – `trustAll=false` in der Gateway-Konfiguration belassen (Standard seit Gravitee 4.4).

---

## 4  SLA-Tiers & Service Levels

Service Level Agreements (SLAs) definieren Zusagen über Verfügbarkeit, Latenz, Durchsatz und Support einer API. Sie sind die Grundlage für Rate Limiting (Kap. 5.1), Monitoring-Alerts (Kap. 6.4), Eskalationsketten und Wartungsplanung.

Jede API wird über das Label `sla-tier` (siehe Kap. 2.4) genau einem Tier zugeordnet: **Bronze**, **Silver** oder **Gold**.

ℹ Die Werte gelten für den **Gateway-Layer**. Backend-Services können zusätzlich eigene SLAs definieren – das End-to-End-SLA ist nur so gut wie das schwächste Glied.

### 4.1  SLA-Tier-Matrix

| Dimension | Bronze | Silver | Gold |
|---|---|---|---|
| **Verfügbarkeit** | 99,0 % | 99,5 % | 99,9 % |
| **Maximale Downtime/Jahr** | ~ 87,6 h | ~ 43,8 h | ~ 8,76 h |
| **P95-Latenz (Gateway-Overhead)** | < 1.000 ms | < 500 ms | < 200 ms |
| **Rate Limit (Burst)** | 10 req/s | 50 req/s | 200 req/s |
| **Quota (Langzeit)** | 10.000 req/Tag | 100.000 req/Tag | Fair Use |
| **Error Budget** | 1,0 % | 0,5 % | 0,1 % |
| **Support-Reaktion P1** | 4 h (Werktage) | 1 h (24/7) | 15 min (24/7) |
| **Support-Reaktion P2** | 1 Werktag | 4 h | 1 h |
| **Wartungsfenster** | beliebig | werktags 22:00 – 06:00 | nur Sa/So 02:00 – 06:00 |
| **Deprecation-Frist** | siehe REST API Styleguide [`<Link>`] | siehe REST API Styleguide [`<Link>`] | siehe REST API Styleguide [`<Link>`] |

### 4.2  Wartungsfenster

#### Braucht Gravitee Downtime?

**Nein – bei korrekter HA-Konfiguration nicht.** Gravitee unterstützt Rolling Updates, Blue/Green- und Canary-Deployments. Ein produktiver Gateway-Cluster mit mindestens 2 Nodes hinter einem Load Balancer kann ohne Service-Unterbrechung aktualisiert werden.

Wartungsfenster sind dennoch erforderlich, weil das Gesamtsystem mehr umfasst als nur den Gateway:

| Szenario | Warum Wartungsfenster? |
|---|---|
| Backend-Service-Wartung | Restarts, Schema-Migrationen, Breaking Deployments der eigentlichen API |
| Gravitee Major-Upgrade | Konfigurationsmigration, Plugin-Updates, ggf. Repository-Migration |
| Datenbank-Wartung | Elasticsearch/OpenSearch Upgrades, Index-Rebuilds, MongoDB-Wartung |
| Infrastruktur-Arbeiten | Netzwerk, Load Balancer, Zertifikat-Rotation, Firewall-Regeln |
| Breaking-Config-Changes | Konfigurationsänderungen, die einen Gateway-Restart erfordern |

#### Warum sind die Fenster für höhere Tiers enger?

Höhere Verfügbarkeitszusagen lassen weniger Spielraum für Wartung:

- **Gold (99,9 %)**: max. ~ 8,76 h Downtime/Jahr → Wartung nur in Nebenzeiten (Wochenende, Nacht), um Konsumenten-Impact zu minimieren
- **Silver (99,5 %)**: max. ~ 43,8 h/Jahr → werktags abends/nachts vertretbar
- **Bronze (99,0 %)**: max. ~ 87,6 h/Jahr → flexibles Fenster, auch geschäftszeiten-nah

#### Pflichten beim Wartungsfenster

- ⚑ **PFLICHT** – Wartungsfenster mindestens **5 Werktage** vorher ankündigen (E-Mail an alle Subscriber + Status-Page-Eintrag).
- ⚑ **PFLICHT** – Bei Notfall-Wartung: Ankündigung sobald möglich, Post-Mortem binnen 5 Werktagen.
- ✓ **EMPFOHLEN** – Auch bei Zero-Downtime-Deployments einen Status-Page-Eintrag setzen („Wartung läuft, keine Beeinträchtigung erwartet").
- ✓ **EMPFOHLEN** – Bei Gold-APIs: Maintenance-Mode-Plan vorbereiten (Read-only-Fallback, Cache-only-Modus).

### 4.3  Support & Eskalation

- ⚑ **PFLICHT** – Jedes Producer-Team benennt einen primären und einen Stellvertreter-Ansprechpartner pro API.
- ⚑ **PFLICHT** – Für Silver- und Gold-APIs: 24/7-Erreichbarkeit per On-Call-Rotation.
- ⚑ **PFLICHT** – Incident-Klassifizierung nach P1/P2/P3 (Definition im Anhang 10.4).
- ✓ **EMPFOHLEN** – Gravitee Alert Engine (Kap. 6.5) als primärer Trigger für Eskalation nutzen.

---

## 5  Traffic Management & Policies

### 5.1  Rate Limiting & Quota

- ⚑ **PFLICHT** – Jede API muss mindestens eine Rate-Limit-Policy pro Plan besitzen.
- ⚑ **PFLICHT** – Quota (langfristiges Limit) und Rate Limit (kurzfristiger Burst-Schutz) **getrennt** konfigurieren.
- ⚑ **PFLICHT** – Werte gemäß SLA-Tier (siehe Kap. 4.1) setzen; Abweichungen erfordern Genehmigung des Platform-Teams.
- ✓ **EMPFOHLEN** – Spike Arrest zusätzlich zum Rate Limit.
- ✓ **EMPFOHLEN** – Redis als Rate-Limit-Store (synchrone Zähler über Gateway-Nodes).

### 5.2  Timeout-Konfiguration

- ⚑ **PFLICHT** – Connect Timeout: max. **5 Sekunden**.
- ⚑ **PFLICHT** – Read Timeout: max. **30 Sekunden** (Default); Long-Polling-APIs explizit dokumentieren und genehmigen lassen.
- ✓ **EMPFOHLEN** – Backend-Timeout kürzer als Gateway-Timeout setzen.

### 5.3  Health Check (Kubernetes-Kontext)

Die meisten Backend-Services laufen in einem **Kubernetes-Cluster**. Dadurch entsteht eine **Zwei-Ebenen-Health-Architektur**:

| Ebene | Wer prüft? | Was wird geprüft? | Reaktion |
|---|---|---|---|
| **Pod-Ebene** | Kubernetes (`livenessProbe` / `readinessProbe`) | Einzelner Pod gesund? | Ungesunde Pods aus dem K8s-Service entfernen, ggf. neu starten |
| **API-/Endpoint-Ebene** | Gravitee Health Check | K8s-Service erreichbar und funktional? | Endpoint im Gateway als unhealthy markieren, Alerts auslösen, Analytics aktualisieren |

Beide Ebenen sind **komplementär**, nicht redundant: K8s reagiert granular auf Pod-Ebene, Gravitee aggregiert auf API-Ebene für Monitoring, Alerts und Developer-Portal-Status.

#### Pflichten

- ⚑ **PFLICHT** – Gravitee Health Check pro API aktivieren. Ziel ist der **K8s-Service** (Cluster-IP / Service-Name), nicht einzelne Pods.
- ⚑ **PFLICHT** – Gravitee Health Check **nicht aggressiver** konfigurieren als die K8s Readiness Probe. Andernfalls markiert Gravitee Endpoints als unhealthy, bevor Kubernetes den Pod austauschen kann (Race Condition, unnötige Alarme).
- ✓ **EMPFOHLEN** – **Identischer `/health`-Endpoint** für K8s und Gravitee (Single Source of Truth).
- ✓ **EMPFOHLEN** – Intervalle abstimmen:

| Probe | Intervall | Timeout | Threshold |
|---|---|---|---|
| K8s Readiness Probe | 5 – 10 s | 1 – 3 s | failure: 3 |
| K8s Liveness Probe | 10 – 30 s | 1 – 5 s | failure: 3 |
| **Gravitee Health Check** | **30 s** | **5 s** | **healthy: 2 / unhealthy: 3** |

#### Hinweis: Gravitee Gateway selbst in Kubernetes

Wenn das Gravitee Gateway selbst in Kubernetes läuft (via Helm Chart oder Gravitee Kubernetes Operator GKO), übernehmen die K8s-Probes die Verwaltung des Gateway-Pods. Es ist **kein zusätzlicher Health Check für das Gateway** zu konfigurieren – die Gravitee-Health-Check-Policy aus diesem Kapitel betrifft ausschließlich die **Backend-Endpoints**, die das Gateway proxiet.

### 5.4  Request-Validation & Transformation

- ✓ **EMPFOHLEN** – OAS Validation Policy aktivieren.
- ✓ **EMPFOHLEN** – Interne Infrastruktur-Header vor Weiterleitung entfernen.
- ⚑ **PFLICHT** – Keine sensitiven Daten (Passwörter, Tokens) in Query-Parametern.

### 5.5  Caching

- ✓ **EMPFOHLEN** – Cache-Policy nur für GET-Endpunkte mit deterministischen Antworten.
- ✓ **EMPFOHLEN** – Cache-TTL an `Cache-Control`-Header des Backends anpassen.
- ⚑ **PFLICHT** – Caching **niemals** für Endpunkte mit personenbezogenen Daten.

---

## 6  Logging, Monitoring & Observability

### 6.1  Request-/Response-Logging

- ⚑ **PFLICHT** – Full Request/Response Logging in Produktion **deaktivieren** (Performance & Datenschutz).
- ⚑ **PFLICHT** – Für Debugging nur temporär und ausschließlich für definierte Test-Subscriptions aktivieren.

### 6.2  Distributed Tracing (OpenTelemetry / W3C Trace Context)

Verteiltes Tracing erfolgt nach dem [W3C Trace Context Standard](https://www.w3.org/TR/trace-context/), kompatibel mit OpenTelemetry. Damit ist End-to-End-Tracing über den Gateway und alle nachgelagerten Backend-Services hinweg möglich.

- ⚑ **PFLICHT** – Die W3C Trace Context Header müssen vom Gateway transparent an das Backend weitergereicht werden.
- ⚑ **PFLICHT** – Falls kein `traceparent`-Header im eingehenden Request vorhanden ist, generiert der Gateway einen neuen (per Policy oder OpenTelemetry-Plugin).

| Header | Standard | Zweck |
|---|---|---|
| `traceparent` | W3C Trace Context | Trace-ID, Span-ID, Sampling-Flag (Pflicht-Header) |
| `tracestate` | W3C Trace Context | Vendor-spezifischer Tracing-Kontext (optional) |
| `baggage` | W3C Baggage | Anwendungs-Kontext (optional, OpenTelemetry) |

Beispiel:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

- ⚑ **PFLICHT** – Logging und Metriken am Gateway müssen Trace-ID und Span-ID aus dem `traceparent`-Header extrahieren und in alle Log-Einträge übernehmen.
- ✓ **EMPFOHLEN** – OpenTelemetry-Exporter im Gateway konfigurieren (OTLP-Endpoint auf zentralen Collector, z. B. Tempo, Jaeger, Datadog APM).

### 6.3  Gravitee-eigene Tracing-Header

Gravitee setzt zusätzlich eigene Tracing-Header. Diese sind **komplementär** zum W3C-Standard, **kein Ersatz**:

| Header | Bedeutung |
|---|---|
| `X-Gravitee-Transaction-Id` | Gateway-interne Transaktions-ID (mehrere Requests einer Transaktion) |
| `X-Gravitee-Request-Id` | Gateway-interne Request-ID (einzelner Request) |

- ✓ **EMPFOHLEN** – Gravitee-Header in den Gateway-Logs belassen; für Backend-Tracing wird ausschließlich `traceparent`/`tracestate` verwendet.
- ✓ **EMPFOHLEN** – Bei Bedarf können die Gravitee-Header per Header-Transformation-Policy entfernt werden, bevor der Request das Backend erreicht.

### 6.4  Analytics & Dashboards

- ⚑ **PFLICHT** – Analytics aktiviert lassen (Elasticsearch/OpenSearch).
- ✓ **EMPFOHLEN** – Pro SLA-Tier ein dediziertes Grafana/Kibana-Dashboard.
- ✓ **EMPFOHLEN** – Trace-ID als Drilldown-Link zwischen Logs/Metrics/Traces nutzen (z. B. Grafana Tempo Integration).
- ✓ **EMPFOHLEN** – Alerts für folgende Schwellwerte (am SLA-Tier orientiert, siehe Kap. 4.1):
  - Error Rate > Error Budget × 5 über 5 Minuten → Warning
  - Error Rate > Error Budget × 10 über 5 Minuten → Critical
  - P95-Latenz > Tier-Latenz-Ziel über 10 Minuten → Warning
  - Rate Limit Quota > 80 % ausgeschöpft → Warning

### 6.5  Gravitee Alert Engine

- ✓ **EMPFOHLEN** – Gravitee Alert Engine für proaktive Benachrichtigungen.
- ✓ **EMPFOHLEN** – Benachrichtigungen an Slack-Channel des Producer-Teams.
- ✓ **EMPFOHLEN** – SLA-Tier-basierte Eskalationsketten definieren (P1/P2/P3 → siehe Kap. 4.3).

---

## 7  Deployment & Lifecycle

### 7.1  Deployment-Prozess für API-Definitionen

- ⚑ **PFLICHT** – APIs dürfen **nicht manuell** über die APIM-Console in Produktion deployt werden – ausschließlich über die Azure-Pipelines-CI/CD.
- ⚑ **PFLICHT** – **Azure Pipelines** ist die verbindliche CI/CD-Plattform; andere CI-Systeme sind nicht zugelassen.
- ⚑ **PFLICHT** – API-Definitionen (JSON/YAML) müssen in einem Git-Repository (Azure Repos oder mit Azure DevOps verbundenes Git) versioniert sein (Single Source of Truth).

Zugelassene Verfahren:

| Verfahren | Einsatzgebiet | Bewertung |
|---|---|---|
| **Gravitee Management API via Azure Pipelines** | Pipeline ruft REST-Endpoints des Management API aus `azure-pipelines.yml` auf | **Standard** |
| **Gravitee Kubernetes Operator (GKO)** | GitOps mit Custom Resources (CRDs); Sync via Argo CD oder Flux | **Bevorzugt bei Kubernetes-Workloads** |
| Terraform-Provider | Für einzelne API-Definitionen **nicht zugelassen** (Community-Provider mit eingeschränkter Coverage) | Nicht empfohlen |

- ⚑ **PFLICHT** – Pull-Request-Workflow: jede Änderung durchläuft einen Code-Review (mindestens 1 Approver aus dem Platform-Team für Prod-Deployments).
- ✓ **EMPFOHLEN** – JSON-Schema-Validierung der API-Definition in der Pipeline.

### 7.2  Azure Pipelines – Struktur & Konventionen

Azure Pipelines ist die verbindliche CI/CD-Plattform für API-Deployments in Gravitee. Pro API-Definition existiert eine `azure-pipelines.yml` im jeweiligen Git-Repository.

#### Pflichten

- ⚑ **PFLICHT** – Pipeline-Definition als **Multi-Stage YAML** (`azure-pipelines.yml`) im API-Repository. Build- und Deployment-Stages liegen in derselben YAML-Datei. Classic Build/Release Pipelines sowie hybride Setups (YAML-Build + Classic-Release) sind nicht zugelassen.
- ⚑ **PFLICHT** – Pipeline durchläuft folgende Stages in dieser Reihenfolge:

| Stage | Zweck | Approval |
|---|---|---|
| `validate` | JSON-Schema-Validierung, Lint, OAS-Check | – |
| `deploy_dev` | Deployment ins Dev-Environment via Management API | – |
| `test_dev` | Smoke- und Integrationstests gegen Dev | – |
| `deploy_staging` | Deployment ins Staging-Environment | Auto nach grünem Test |
| `test_staging` | Vollständige QA, Quality-Score-Check | – |
| `deploy_prod` | Deployment in Produktion | **Manuelles Approval (Platform-Team)** |

- ⚑ **PFLICHT** – Pro Stage ein eigenes Azure DevOps **Environment** (`gravitee-dev`, `gravitee-staging`, `gravitee-prod`). Prod-Environment mit Approval-Gate konfiguriert.
- ⚑ **PFLICHT** – Authentifizierung gegen die Gravitee Management API via **Azure DevOps Service Connection** (Generic / OAuth2); kein hartkodierter Token in der Pipeline.
- ⚑ **PFLICHT** – Secrets (API-Tokens, Credentials) ausschließlich über **Azure Key Vault** + Variable Group; keine Secrets in YAML oder Repo-Variablen.
- ⚑ **PFLICHT** – Jeder Pipeline-Run muss die **Trace-ID** des Deployments in den Gravitee-Audit-Log schreiben (Build-ID als Tag an die API-Definition).

#### Empfehlungen

- ✓ **EMPFOHLEN** – Wiederverwendbare **Pipeline-Templates** aus dem zentralen Templates-Repo des Platform-Teams nutzen (siehe Anhang 10.2).
- ✓ **EMPFOHLEN** – **Branch Policies** in Azure Repos: PR-Validierung (`validate` + `deploy_dev`) muss vor Merge in `main` grün sein.
- ✓ **EMPFOHLEN** – Pipeline-Caching für npm/Maven-Abhängigkeiten zur Schema-Validierung aktivieren.
- ✓ **EMPFOHLEN** – Bei GKO-basiertem Deployment: Azure Pipeline pusht die CRDs ins Git-Repo, Argo CD/Flux übernehmen den Sync (GitOps-Pattern).

#### Beispielstruktur `azure-pipelines.yml`

```yaml
trigger:
  branches:
    include: [ main, release/* ]

variables:
  - group: gravitee-secrets   # via Azure Key Vault

stages:
  - stage: validate
    jobs:
      - job: lint_and_schema
        steps:
          - script: npm ci && npm run validate:api

  - stage: deploy_dev
    dependsOn: validate
    jobs:
      - deployment: deploy
        environment: gravitee-dev
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/gravitee-deploy.yml@platform-templates

  - stage: deploy_prod
    dependsOn: test_staging
    jobs:
      - deployment: deploy
        environment: gravitee-prod   # Approval-Gate konfiguriert
        strategy:
          runOnce:
            deploy:
              steps:
                - template: templates/gravitee-deploy.yml@platform-templates
```

#### Repo-Layout

Jede API hat ein eigenes Git-Repository in Azure Repos. Verbindliches Grundlayout:

```
order-shipments-v2/
├── README.md                       # Zweck, Owner, Slack-Channel, On-Call
├── CODEOWNERS                      # Pflicht-Reviewer pro Pfad
├── azure-pipelines.yml             # Multi-Stage Pipeline (validate → dev → staging → prod)
├── api/
│   ├── api-definition.json         # Gravitee API-Definition (v4)
│   ├── openapi.yaml                # OpenAPI 3.x Spezifikation
│   └── plans/                      # Plan-Konfigurationen (JWT, API-Key, ...)
├── environments/
│   ├── dev.vars.yml                # Endpoint-URLs, Tier, Rate Limits pro Env
│   ├── staging.vars.yml            # KEINE Secrets - die kommen aus Azure Key Vault
│   └── prod.vars.yml
├── tests/
│   ├── smoke/                      # Newman / Postman Collection für Smoke-Tests
│   └── integration/                # Vollständige Integrationstests (z.B. k6, REST Assured)
├── docs/
│   ├── changelog.md                # API-Changelog (siehe REST API Styleguide)
│   └── runbook.md                  # Operatives Runbook für On-Call
└── .gitignore
```

| Element | Pflicht | Zweck |
|---|---|---|
| `README.md` | ⚑ | Owner, Kontakt, Slack-Channel, On-Call-Verweis |
| `CODEOWNERS` | ⚑ | Automatische Reviewer-Zuweisung in PRs (Azure Repos) |
| `azure-pipelines.yml` | ⚑ | Multi-Stage Pipeline (siehe oben) |
| `api/api-definition.json` | ⚑ | Gravitee-API-Definition als Single Source of Truth |
| `api/openapi.yaml` | ⚑ | OpenAPI 3.x (referenziert in Gravitee als Dokumentation) |
| `environments/*.vars.yml` | ⚑ | Pro Environment getrennte Variablen |
| `tests/smoke/` | ⚑ | Mindestens ein Smoke-Test, der in der Pipeline läuft |
| `tests/integration/` | ✓ | Vollständige Tests |
| `docs/runbook.md` | ✓ | Pflicht für Silver/Gold-APIs |

- ⚑ **PFLICHT** – Keine Secrets, Tokens oder Credentials im Repo (auch nicht in `environments/*.vars.yml`). Alle sensiblen Werte über Azure Key Vault + Variable Group beziehen.
- ⚑ **PFLICHT** – Repo-Name entspricht dem API-Namen aus Kap. 2.1 (`<domäne>-<ressource>-v<major>`).
- ✓ **EMPFOHLEN** – Pre-Commit Hooks für lokale Schema-Validierung (`api-definition.json`, `openapi.yaml`).

### 7.3  Environments

- ⚑ **PFLICHT** – Drei Environments sind Pflicht: `dev`, `staging`, `prod`.
- ⚑ **PFLICHT** – Promotion `dev → staging → prod` nur über definierte Approval-Prozesse.

| Environment | Besonderheiten |
|---|---|
| `dev` | Keyless-Plans erlaubt, volle Logs, kein HA |
| `staging` | Produktionsnahe Konfiguration, Integrationstests |
| `prod` | Kein Keyless, minimale Logs, HA mit min. 2 Nodes, Redis Pflicht |

### 7.4  Versionierung & Breaking Changes

- ⚑ **PFLICHT** – Breaking Changes erfordern eine neue Major-Version (`v1` → `v2`) und einen neuen Context-Path.
- ⚑ **PFLICHT** – Deprecation- und Sunset-Prozess (Fristen, Kommunikation, Sunset-Header) sind im REST API Styleguide geregelt:

> `<Platzhalter: Link zum REST API Styleguide, Kapitel Deprecation & Versionierung>`

- ⚑ **PFLICHT** – Sunset-Datum im API-Header technisch kommunizieren (gemäß REST API Styleguide):

```
Sunset: Sat, 01 Jan 2026 00:00:00 GMT
Deprecation: true
```

- ✓ **EMPFOHLEN** – Konsumenten bei Deprecation automatisch per E-Mail benachrichtigen (APIM Subscription-Notification).

### 7.5  Hybrid-spezifische Hinweise

Im Hybrid-Deployment läuft der Gateway on-premise, die Control Plane (APIM Console, Developer Portal) als SaaS:

- ⚑ **PFLICHT** – Gateway muss Outbound-Verbindung zur Gravitee Cloud Control Plane haben (Port 443).
- ⚑ **PFLICHT** – API-Schlüssel und Subscriptions werden lokal gecacht – Sync-Intervall beachten (Standard: 5 Sekunden).
- ✓ **EMPFOHLEN** – Lokale Redis-Instanz für Rate-Limit-Synchronisation zwischen Gateway-Nodes.
- ✓ **EMPFOHLEN** – Netzwerk-Firewall-Regeln dokumentieren und regelmäßig reviewen.

---

## 8  API Review & Quality Gate

### 8.1  Quality-Scoring (Gravitee APIM)

Gravitee APIM bietet ein konfigurierbares Quality-Scoring:

| Kriterium | Gewicht | Pflicht | Prüfung |
|---|---|---|---|
| Beschreibung vorhanden | 10 % | Ja | Automatisch |
| OpenAPI-Spec hinterlegt | 20 % | Ja | Automatisch |
| Min. 1 sicherer Plan | 25 % | Ja | Automatisch |
| Rate Limit konfiguriert | 20 % | Ja | Automatisch |
| Labels vollständig | 15 % | Ja | Manuell |
| Health Check aktiv | 10 % | Ja | Automatisch |

⚑ **PFLICHT** – Minimum Quality Score: **80 %** – APIs unterhalb dieses Wertes werden blockiert.

### 8.2  Review-Checkliste (manuell)

- [ ] Namenskonventionen eingehalten (Kap. 2)
- [ ] Security-Policy korrekt konfiguriert (Kap. 3)
- [ ] SLA-Tier zugewiesen und passend zur Nutzung (Kap. 4)
- [ ] Rate Limits dem SLA-Tier entsprechend gesetzt (Kap. 5.1)
- [ ] Health Check K8s-konform (Kap. 5.3)
- [ ] W3C Trace Context Header werden weitergereicht (Kap. 6.2)
- [ ] Keine sensitiven Daten in Logs oder Query-Parametern
- [ ] Azure Pipeline (`azure-pipelines.yml`) vorhanden und getestet
- [ ] Verantwortlicher Ansprechpartner hinterlegt

---

## 9  Onboarding-Prozess für Producer-Teams

| Schritt | Aktion |
|---|---|
| 1. Anfrage | Formular im internen Service-Katalog ausfüllen (Name, Domäne, SLA-Tier, Owner) |
| 2. Template | Gravitee-API-Template (JSON/CRD) vom Platform-Team anfordern oder aus Git-Repo klonen |
| 3. Konfiguration | Template anpassen: Endpoint, Policies, Labels, Plan gemäß diesem Styleguide |
| 4. Validierung | Lokale Schema-Validierung; Import in Dev-Environment und Smoke-Test |
| 5. Review | Pull Request im API-Definitions-Repo; Platform-Team reviewt |
| 6. Staging | Nach Approval: automatisches Deployment nach Staging via Azure Pipelines |
| 7. QA | Integrationstests und Quality-Score-Check in Staging |
| 8. Produktion | Nach QA-Sign-off: Deployment in Prod via Azure Pipelines (manuelles Approval-Gate) |

### 9.1  Kontakt & Support

- **Slack:** `#platform-api-gateway`
- **E-Mail:** `api-platform@<euer-unternehmen>.de`
- **Ticket:** Jira-Projekt `APIGW`

---

## 10  Anhang

### 10.1  Schnell-Referenz Pflichtanforderungen

| Kategorie | Pflichtanforderungen (Kurzübersicht) |
|---|---|
| **Naming** | Schema `<domäne>-<ressource>-v<major>` · Context-Path mit `/v<major>` · OAS-Spec |
| **Sicherheit** | Kein Keyless in Prod · JWT: RS256/ES256 · Kein Auto-Approve · CORS-Whitelist |
| **SLA** | SLA-Tier zugewiesen · Werte gemäß Tier-Matrix · On-Call für Silver/Gold |
| **Traffic** | Rate Limit pro Plan · Health Check K8s-konform · TLS für Backend-Verbindungen |
| **Tracing** | W3C `traceparent` / `tracestate` durchreichen · OpenTelemetry-konforme Logs |
| **Logging** | Kein Full-Log in Prod · Trace-ID in Logs übernehmen |
| **Deployment** | Kein manuelles Deployment in Prod · Git-Versionierung · 3 Environments · Azure Pipelines (Multi-Stage YAML) · Management API oder GKO |
| **Quality Gate** | Min. 80 % Quality Score · Manuelle Review-Checkliste bestanden |

### 10.2  Weiterführende Dokumentation

| Ressource | Link / Pfad |
|---|---|
| **REST API Styleguide (intern)** | `<Platzhalter: Link zum REST API Styleguide>` |
| Gravitee APIM Dokumentation | <https://documentation.gravitee.io/apim> |
| Production Best Practices | <https://documentation.gravitee.io/apim/prepare-a-production-environment> |
| Gravitee Management API Referenz | <https://documentation.gravitee.io/apim/reference/management-api> |
| Gravitee Kubernetes Operator (GKO) | <https://documentation.gravitee.io/gravitee-kubernetes-operator-gko> |
| W3C Trace Context Standard | <https://www.w3.org/TR/trace-context/> |
| OpenTelemetry Specification | <https://opentelemetry.io/docs/specs/otel/> |
| Kubernetes Probes Doku | <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/> |
| Azure Pipelines Dokumentation | <https://learn.microsoft.com/azure/devops/pipelines/> |
| API-Definitions Git-Repo (Azure Repos) | `<interne URL>` |
| Azure Pipelines Templates (Platform-Team) | `<interner Repo-Pfad: platform-templates>` |

### 10.3  Glossar

| Begriff | Bedeutung |
|---|---|
| **SLA** | Service Level Agreement – Zusage über Servicequalität (Verfügbarkeit, Latenz, Support) |
| **SLO** | Service Level Objective – internes Ziel, an dem das SLA gemessen wird |
| **SLI** | Service Level Indicator – konkrete Metrik (z. B. P95-Latenz) |
| **Error Budget** | Erlaubter Anteil fehlgeschlagener Requests pro Zeitfenster (= 100 % – SLO) |
| **GKO** | Gravitee Kubernetes Operator |
| **OAS** | OpenAPI Specification |
| **OTLP** | OpenTelemetry Protocol |
| **P50/P95/P99** | Perzentile der Antwortzeitverteilung |

### 10.4  Incident-Klassifizierung

| Priorität | Beschreibung | Beispiele |
|---|---|---|
| **P1** | Produktions-Ausfall, hoher Geschäftsimpact | API komplett down, Datenverlust, Security-Breach |
| **P2** | Eingeschränkte Funktionalität, mittlerer Impact | Hohe Fehlerrate, Latenz weit über SLA |
| **P3** | Geringer Impact, kein Workaround nötig | Einzelne Endpoints betroffen, kosmetische Fehler |

### 10.5  Änderungshistorie

| Version | Datum | Änderung |
|---|---|---|
| 1.0 | – | Initiale Version |
| 1.1 | – | Logging auf W3C Trace Context / OpenTelemetry umgestellt; Terraform-Nutzung präzisiert; Verweis auf REST API Styleguide ergänzt |
| 1.2 | – | Neues Kap. 4 SLA-Tiers & Service Levels; Wartungsfenster erklärt; Health Check um K8s-Kontext erweitert; Deprecation-Frist als Verweis auf REST API Styleguide; Glossar und Incident-Klassifizierung ergänzt |
| 1.3 | – | Azure Pipelines als verbindliche CI/CD-Plattform; neuer Abschnitt 7.2 zu Pipeline-Struktur, Stages, Approval-Gates und Service Connections; Folgeabschnitte renummeriert |
| 1.4 | – | Multi-Stage YAML als Pflicht präzisiert; verbindliches Repo-Layout (azure-pipelines.yml, api/, environments/, tests/, docs/) in 7.2 ergänzt; Kapitel "Infrastructure as Code" entfernt; Folgekapitel renummeriert |


