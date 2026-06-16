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

# REST API Styleguide — Erklärungen auf Deutsch

> Version 2.0 — Basierend auf [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/), [Adidas API Guidelines](https://adidas.gitbook.io/api-guidelines/) und [Stripe API](https://docs.stripe.com/api).  
> Zalando-interne Regeln wurden entfernt. Quercheck mit Adidas und Stripe eingearbeitet.  
> Alle Regeln sind auf Deutsch erklärt. Eigene Regeln sind explizit gekennzeichnet.

## Inhaltsverzeichnis

- [Konventionen](#konventionen)
- [Abweichungen gegenüber Zalando](#abweichungen)
- [1. Allgemeine Richtlinien](#kap-1)
  - [#100 · API-First-Prinzip](#r-100)
  - [#101 · API-Spezifikation mit OpenAPI](#r-101)
  - [#102 · API-Benutzerhandbuch](#r-102)
  - [#103 · U.S. English](#r-103)
  - [C-08 · Minimale API-Oberfläche (YAGNI)](#c-08)
  - [C-09 · Postel's Law](#c-09)
  - [C-12 · API-Specs in Git versionieren](#c-12)
- [2. Meta-Informationen](#kap-2)
  - [#218 · API Meta-Informationen](#r-218)
  - [#116 · Semantic Versioning](#r-116)
  - [#215 · API-Identifier](#r-215)
  - [#219 · API-Zielgruppe](#r-219)
- [3. Sicherheit](#kap-3)
  - [#104 · Alle Endpunkte absichern](#r-104)
  - [#105 · Berechtigungen (Scopes)](#r-105)
  - [C-06 · Scope-Namenskonvention](#c-06)
- [4. Datenformate](#kap-4)
  - [#238 · Standarddatenformate](#r-238)
  - [#171 · Format für Zahlen und Integer](#r-171)
  - [#169 · Datum/Zeit-Formate](#r-169)
  - [#255 · Geeignete Datum/Zeit-Formate](#r-255)
  - [#127 · Zeitdauern](#r-127)
  - [#170 · Land, Sprache, Währung](#r-170)
  - [#244 · Content Negotiation](#r-244)
  - [#144 · UUIDs](#r-144)
- [5. URLs](#kap-5)
  - [C-01 · URL-Versionierung](#c-01)
  - [#134 · Ressourcennamen im Plural](#r-134)
  - [#228 · URL-kompatible IDs](#r-228)
  - [#129 · kebab-case für Pfadsegmente](#r-129)
  - [#136 · Normalisierte Pfade](#r-136)
  - [#141 · URLs verb-frei](#r-141)
  - [#138 · Ressourcen statt Aktionen](#r-138)
  - [#142 · Domänenspezifische Namen](#r-142)
  - [#143 · Ressourcen via Pfadsegmente](#r-143)
  - [#130 · snake_case für Query-Parameter](#r-130)
  - [#137 · Konventionelle Query-Parameter](#r-137)
- [6. JSON Payload](#kap-6)
  - [#167 · JSON als Datenformat](#r-167)
  - [#118 · snake_case für Properties](#r-118)
  - [C-02 · Kein HATEOAS](#c-02)
  - [#174 · Gemeinsame Feldnamen](#r-174)
  - [C-07 · metadata-Feld](#c-07)
  - [C-11 · description vs. metadata](#c-11)
  - [#235 · _at-Suffix für Datum/Zeit](#r-235)
  - [#240 · Enum UPPER_SNAKE_CASE](#r-240)
  - [#120 · Array-Namen im Plural](#r-120)
  - [#123 · null und fehlende Properties](#r-123)
  - [#122 · null für Boolean](#r-122)
  - [#124 · null für Arrays](#r-124)
  - [#216 · Maps mit additionalProperties](#r-216)
- [7. HTTP-Anfragen](#kap-7)
  - [#148 · HTTP-Methoden korrekt](#r-148)
  - [#149 · Methoden-Eigenschaften](#r-149)
  - [#229 · Idempotentes POST/PATCH](#r-229)
  - [#231 · Sekundärschlüssel für POST](#r-231)
  - [#253 · Asynchrone Verarbeitung](#r-253)
  - [#154 · Collection-Format für Parameter](#r-154)
  - [#236 · Einfache Filter](#r-236)
  - [#237 · Komplexe Filter](#r-237)
  - [#226 · Implizite Filterung dokumentieren](#r-226)
- [8. HTTP-Statuscodes](#kap-8)
  - [#243 · Nur offizielle Statuscodes](#r-243)
  - [#151 · Alle Statuscodes spezifizieren](#r-151)
  - [#150 · Gebräuchliche Statuscodes](#r-150)
  - [#220 · Spezifischster Statuscode](#r-220)
  - [#152 · Code 207 für Batch](#r-152)
  - [#153 · Code 429 mit Retry-After](#r-153)
  - [#176 · Problem JSON](#r-176)
  - [#177 · Keine Stack Traces](#r-177)
- [9. HTTP-Header](#kap-9)
  - [C-03/C-04 · W3C Trace Context](#c-03-04)
  - [C-05 · trace_id in Fehler-Responses](#c-05)
  - [#132 · kebab-case für Header](#r-132)
  - [#178 · Content-Header](#r-178)
  - [#180 · Location Header](#r-180)
  - [#182 · ETag](#r-182)
  - [#230 · Idempotency-Key](#r-230)
  - [#181 · Prefer Header](#r-181)
- [10. Performance](#kap-10)
  - [#227 · Cacheable Endpunkte](#r-227)
  - [#156 · gzip-Komprimierung](#r-156)
  - [#157 · Partial Responses](#r-157)
  - [#158 · Sub-Ressourcen einbetten](#r-158)
- [11. Paginierung](#kap-11)
  - [#159 · Paginierung für alle Collections](#r-159)
  - [#160 · Cursor-basierte Paginierung](#r-160)
  - [#248 · Pagination Response Object](#r-248)
  - [#254 · Gesamtanzahl vermeiden](#r-254)
- [12. Kompatibilität und Erweiterbarkeit](#kap-12)
  - [#106 · Keine Breaking Changes](#r-106)
  - [C-10 · Vier Erweiterungsregeln](#c-10)
  - [#108 · Clients auf Erweiterungen vorbereiten](#r-108)
  - [#110 · JSON-Objekte als Top-Level](#r-110)
  - [#111 · OpenAPI als erweiterbar](#r-111)
  - [#112 · Offene Enum-Listen](#r-112)
- [13. Deprecation](#kap-13)
  - [#187 · Deprecation in Spec markieren](#r-187)
  - [#185 · Genehmigung vor Abschaltung](#r-185)
  - [#186 · Consent externer Partner](#r-186)
  - [#188 · Nutzung monitoren](#r-188)
  - [#191 · Keine deprecated APIs nutzen](#r-191)
  - [#189 · Deprecation Header](#r-189)
  - [#190 · Monitoring für Sunset](#r-190)
- [14. Betrieb](#kap-14)
  - [#192 · OpenAPI veröffentlichen](#r-192)
  - [#193 · API-Nutzung monitoren](#r-193)
- [Übersicht: Alle eigenen Regeln](#eigene-regeln)
- [Entfernte Zalando-interne Regeln](#entfernte-regeln)

---

<a name="konventionen"></a>
## Konventionen

| Begriff | Bedeutung |
|---|---|
| **MUSS** | Verpflichtend — keine Ausnahmen |
| **SOLLTE** | Empfohlen — Abweichungen müssen begründet werden |
| **KANN** | Optional — nach eigenem Ermessen |
| ✦ EIGENE REGEL | Unsere Abweichung oder Ergänzung gegenüber Zalando |
| *(Adidas)* | Regel aus Adidas Guidelines übernommen |
| *(Stripe)* | Pattern aus Stripe API übernommen |

---

<a name="abweichungen"></a>
## Zusammenfassung unserer Abweichungen gegenüber Zalando

| Zalando-Regel | Original | Unsere Regelung |
|---|---|---|
| #113 / #114 / #115 | Media Type Versioning, kein URL-Versioning | ✦ **MUSS** URL-Versionierung: `/v{n}/resource` |
| #163 / #164 / #165 | HATEOAS optional | ✦ Entfernt — kein HATEOAS, kein `_links`, `href`, `self` |
| #233 | X-Flow-ID | ✦ Ersetzt durch W3C `traceparent` + `tracestate` (OpenTelemetry) |
| #223 / #224 | Zalando Functional Naming | Entfernt — Zalando-intern |
| #183 | Zalando Proprietary Headers | Entfernt — Zalando-intern |
| #173 / #249 | Zalando Money / Address | Entfernt — Zalando-intern |

---

<a name="kap-1"></a>
## 1. Allgemeine Richtlinien

<a name="r-100"></a>
### #100 · MUSS · API-First-Prinzip befolgen

APIs müssen **vor** der Implementierung spezifiziert werden — nicht danach. Das bedeutet:

- Die API-Spezifikation (OpenAPI) wird als erstes erstellt, bevor Code geschrieben wird
- Kolleginnen und Kollegen sowie Client-Entwickler geben frühzeitig Feedback
- Die API ist stabil, auch wenn sich die Implementierung dahinter ändert

**Warum?** APIs sind Verträge mit Konsumenten. Wer zuerst implementiert und dann dokumentiert, baut Abhängigkeiten ein, die später schwer zu ändern sind.

---

<a name="r-101"></a>
### #101 · MUSS · API-Spezifikation mit OpenAPI bereitstellen

Alle APIs müssen mit **OpenAPI 3.1** als einzelne, in sich geschlossene YAML-Datei spezifiziert werden.

- Die Datei muss versioniert in einem Source-Control-System (Git) liegen
- Keine externen Referenzen auf URLs die sich ändern könnten
- Die Spezifikation muss zusammen mit dem Service deployed werden

```yaml
openapi: 3.1.0
info:
  title: Order Management API
  version: 1.0.0
```

---

<a name="r-102"></a>
### #102 · SOLLTE · API-Benutzerhandbuch bereitstellen

Zusätzlich zur technischen Spezifikation sollte ein Benutzerhandbuch für API-Konsumenten existieren mit:

- Zweck und Anwendungsfällen der API
- Konkreten Beispielen zur Nutzung
- Typischen Fehlerfällen und wie man sie behebt
- Architekturkontext und wichtige Abhängigkeiten

Das Handbuch wird über `#/externalDocs/url` in der OpenAPI-Spezifikation verlinkt.

---

<a name="r-103"></a>
### #103 · MUSS · APIs auf amerikanischem Englisch schreiben

Alle API-Bezeichnungen, Beschreibungen, Fehlermeldungen und Dokumentationen müssen auf **US-Englisch** verfasst sein. Das gilt für:

- Ressourcennamen (`/orders`, `/customers`)
- Property-Namen (`order_id`, `created_at`)
- OpenAPI `description`-Felder
- Fehlermeldungen und Problem-JSON-Texte

---

<a name="c-08"></a>
### ✦ C-08 · MUSS · Minimale API-Oberfläche (YAGNI-Prinzip) *(Adidas)*

Jedes API-Design MUSS auf eine minimale API-Oberfläche abzielen, ohne Produktanforderungen zu vernachlässigen.

- Keine Ressourcen, Relationen, Aktionen oder Felder die noch nicht gebraucht werden
- Keine vorauseilende Generalisierung
- Neue Funktionalität wird erst hinzugefügt wenn ein konkreter Bedarf besteht

**YAGNI:** "You Ain't Gonna Need It" — was heute nicht gebraucht wird, kommt auch nicht rein.

```
# Falsch: generische "items"-Ressource für alle Entitäten
GET /v1/items?type=order

# Richtig: spezifische Ressource nur wenn gebraucht
GET /v1/orders
```

---

<a name="c-09"></a>
### ✦ C-09 · MUSS · Robustheit nach Postel's Law *(Adidas)*

Jede API-Implementierung und jeder API-Konsument MUSS Postel's Law befolgen:

> *"Be conservative in what you send, be liberal in what you accept."*

**Als Server:**
- Nur notwendige Daten senden — niemals mehr als erforderlich
- Keine internen Details, Stack Traces oder Debug-Informationen exponieren

**Als Client:**
- Unbekannte Properties ignorieren (nicht mit Fehler ablehnen)
- Neue Enum-Werte tolerieren
- Zusätzliche HTTP-Header tolerieren

Dies stärkt Kompatibilität und Erweiterbarkeit des gesamten API-Ökosystems.

---

<a name="c-12"></a>
### ✦ C-12 · MUSS · API-Spezifikationen in Git versionieren *(Adidas)*

OpenAPI-Spezifikationen MÜSSEN in einem Versionskontrollsystem (Git) verwaltet werden:

- Gleiche Repository-Konventionen wie Code
- Git Tags für jede veröffentlichte API-Version: `api/v1.2.0`
- `CHANGELOG.md` dokumentiert alle Breaking Changes und Deprecations
- Pull Requests für alle API-Änderungen — kein direktes Commit auf `main`

---

<a name="kap-2"></a>
## 2. Meta-Informationen

<a name="r-218"></a>
### #218 · MUSS · API Meta-Informationen enthalten

Jede OpenAPI-Spezifikation muss folgende Pflichtfelder im `info`-Block enthalten:

```yaml
info:
  title: Order Management API          # Eindeutiger, beschreibender Name
  version: 1.2.3                       # Semantic Versioning (MAJOR.MINOR.PATCH)
  description: |
    API zur Verwaltung von Bestellungen.
    Erlaubt das Anlegen, Abfragen und Stornieren von Bestellungen.
  contact:
    name: Platform Team
    email: platform-team@company.com
    url: https://wiki.company.com/platform
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70   # Pflicht: UUID
  x-audience: external-partner                        # Pflicht: Zielgruppe
```

---

<a name="r-116"></a>
### #116 · MUSS · Semantic Versioning verwenden

Die API-Spec-Version folgt dem Schema `MAJOR.MINOR.PATCH`:

| Änderung | Aktion |
|---|---|
| Breaking Change (inkompatibel) | MAJOR erhöhen: `1.x.x` → `2.0.0` |
| Neue Funktion (rückwärtskompatibel) | MINOR erhöhen: `1.2.x` → `1.3.0` |
| Bugfix / Typo in Doku | PATCH erhöhen: `1.2.3` → `1.2.4` |

**Wichtig:** Diese Versionsnummer betrifft die API-Spezifikationsdatei — nicht die URL-Version (siehe ✦ C-01).

---

<a name="r-215"></a>
### #215 · MUSS · API-Identifier bereitstellen

Jede API bekommt eine **global eindeutige, unveränderliche UUID** als `x-api-id`. Diese ändert sich nie — auch nicht bei Breaking Changes oder Umbenennung der API.

```yaml
info:
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70
```

**Warum?** Der Identifier erlaubt die lückenlose Nachverfolgung der API-Evolution über alle Versionen hinweg, unabhängig von Namensänderungen.

---

<a name="r-219"></a>
### #219 · MUSS · API-Zielgruppe angeben

Jede API muss ihre Zielgruppe deklarieren. Dies steuert Qualitätsanforderungen, Review-Prozesse und Zugriffsrechte:

| Wert | Bedeutung |
|---|---|
| `component-internal` | Nur innerhalb derselben Anwendungskomponente |
| `business-unit-internal` | Innerhalb derselben Business Unit |
| `company-internal` | Alle internen Teams des Unternehmens |
| `external-partner` | Externe Geschäftspartner |
| `external-public` | Öffentlich zugänglich für alle |

```yaml
info:
  x-audience: external-partner
```

---

<a name="kap-3"></a>
## 3. Sicherheit

<a name="r-104"></a>
### #104 · MUSS · Alle Endpunkte absichern

Jeder API-Endpunkt muss durch Authentifizierung und Autorisierung geschützt sein. Empfohlene Methoden:

- **OAuth 2.0 mit JWT Bearer Token** (bevorzugt für interne APIs)
- **OAuth 2.0 Authorization Code Flow** (für externe/Partner-APIs)
- **API Key** (nur für einfache Use Cases ohne Benutzerkontext)

```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - BearerAuth: []
```

---

<a name="r-105"></a>
### #105 · MUSS · Berechtigungen (Scopes) definieren und zuweisen

Endpunkte, die klassifizierte Daten exponieren, müssen mit Scopes abgesichert werden:

```yaml
paths:
  /v1/orders:
    get:
      security:
        - BearerAuth: [read:orders]
    post:
      security:
        - BearerAuth: [write:orders]
```

---

<a name="c-06"></a>
### ✦ C-06 · MUSS · Einheitliche Scope-Namenskonvention (löst #225 ab)

#### Kontext

Scopes sind der Mechanismus mit dem ein OAuth2-Token deklariert, **was ein Client mit einer API darf**. Ohne einheitliche Benennung entstehen schnell willkürliche Namen wie `fullAccess`, `myScope` oder `perm_1` — die für Konsumenten unverständlich und für das Authorization-System schwer verwaltbar sind.

Zalando Regel #225 löste dieses Problem über ein internes Functional Component Registry (`{application-id}.{access-type}`). Da wir dieses Zalando-interne System nicht verwenden, definieren wir ein eigenes, einfaches Schema.

#### Schema

```
{aktion}:{ressource}
```

- **Aktion:** Kleinbuchstaben, eine der drei erlaubten Werte: `read`, `write`, `admin`
- **Ressource:** Kleinbuchstaben, kebab-case, entspricht dem Ressourcennamen im URL-Pfad (Plural)
- **Trennzeichen:** Doppelpunkt `:`

#### Erlaubte Aktionen

| Aktion | Bedeutung | Typische HTTP-Methoden |
|---|---|---|
| `read` | Lesender Zugriff | `GET`, `HEAD` |
| `write` | Schreibender Zugriff (anlegen, ändern, löschen) | `POST`, `PUT`, `PATCH`, `DELETE` |
| `admin` | Administrative Operationen (z.B. Konfiguration, Massenoperationen) | `POST`, `DELETE` auf Admin-Endpunkten |

#### Anwendungsbeispiele

**Einfache Ressource:**
```yaml
# OpenAPI Security Scheme Definition
components:
  securitySchemes:
    OAuth2:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://auth.example.com/oauth/token
          scopes:
            read:orders: Bestellungen und Bestellpositionen lesen
            write:orders: Bestellungen anlegen, ändern und stornieren
            admin:orders: Bestellungen im Auftrag anderer Mandanten verwalten

# Endpunkt-Absicherung
paths:
  /v1/orders:
    get:
      security:
        - OAuth2: [read:orders]
    post:
      security:
        - OAuth2: [write:orders]
  /v1/orders/{id}:
    patch:
      security:
        - OAuth2: [write:orders]
    delete:
      security:
        - OAuth2: [write:orders]
```

**Mehrere Ressourcen in einer API:**
```yaml
scopes:
  read:orders:        Bestellungen lesen
  write:orders:       Bestellungen schreiben
  read:customers:     Kunden lesen
  write:customers:    Kunden schreiben
  admin:customers:    Kunden administrieren (Merge, Löschen)
```

**Endpunkt der mehrere Scopes akzeptiert:**
```yaml
# Entweder read:orders ODER admin:orders berechtigt
/v1/orders:
  get:
    security:
      - OAuth2: [read:orders]
      - OAuth2: [admin:orders]
```

#### Wann `write` vs. `admin`?

`write` ist für normale CRUD-Operationen durch reguläre Konsumenten. `admin` ist für Operationen die erweiterte Rechte erfordern — typischerweise:

- Operationen im Namen anderer Mandanten / Nutzer
- Massenoperationen (Batch-Delete, Bulk-Update)
- Konfigurationsänderungen die alle Konsumenten betreffen
- Zugriff auf nicht-öffentliche Felder (z.B. interne Kostenfelder)

#### Was ist NICHT erlaubt

```
# ✗ Freitext ohne Schema
fullAccess
myOrderPermission
perm_read_1

# ✗ Falsches Trennzeichen
read.orders       (Punkt — Zalando-Schema)
read/orders       (Slash)
readOrders        (camelCase)

# ✗ Singular statt Plural
read:order        (Ressource muss Plural sein wie im URL)

# ✗ Verb im Ressourcennamen
read:get-orders   (Verb gehört in die Aktion, nicht die Ressource)
```

#### Registrierung in Gravitee

Scopes werden in Gravitee beim API-Plan definiert und müssen exakt mit der OpenAPI-Spezifikation übereinstimmen:

```json
{
  "name": "Premium Plan",
  "security": "OAUTH2",
  "scopes": ["read:orders", "write:orders"]
}
```

#### Abgrenzung zu #104 und #105

- **#104** sagt: *jeder Endpunkt muss abgesichert sein* — das **Ob**
- **#105** sagt: *Scopes müssen definiert und zugewiesen werden* — das **Was**
- **C-06** sagt: *Scopes müssen diesem Namensschema folgen* — das **Wie**

---

<a name="kap-4"></a>
## 4. Datenformate

<a name="r-238"></a>
### #238 · MUSS · Standarddatenformate verwenden

Für alle Datentypen müssen die OpenAPI-Standardformate verwendet werden:

| Typ | Format | Beispiel |
|---|---|---|
| `integer` | `int32` / `int64` / `bigint` | `42`, `7721071004` |
| `number` | `float` / `double` / `decimal` | `3.14`, `99.95` |
| `string` | `date` | `"2024-01-15"` |
| `string` | `date-time` | `"2024-01-15T10:30:00Z"` |
| `string` | `time` | `"10:30:00Z"` |
| `string` | `duration` | `"P1DT3H"` (1 Tag, 3 Stunden) |
| `string` | `email` | `"user@example.com"` |
| `string` | `uri` | `"https://api.example.com/v1/orders"` |
| `string` | `uuid` | `"e2ab873e-b295-11e9-9c02-..."` |
| `string` | `iso-639-1` | `"de"`, `"en"` |
| `string` | `iso-3166-alpha-2` | `"DE"`, `"GB"` |
| `string` | `iso-4217` | `"EUR"`, `"USD"` |

---

<a name="r-171"></a>
### #171 · MUSS · Format für Zahlen und Integer definieren

Jede Zahl- oder Integer-Property **muss** ein explizites Format angeben:

```yaml
properties:
  quantity:
    type: integer
    format: int32       # MUSS angegeben werden
  price:
    type: number
    format: decimal     # MUSS angegeben werden
  large_id:
    type: integer
    format: int64       # Für große IDs
```

**Warum?** Ohne Format raten Clients die Präzision — und liegen oft falsch, was zu Datenverlust führt.

---

<a name="r-169"></a>
### #169 · MUSS · Standardformate für Datum/Zeit verwenden

- Immer **RFC 3339 / ISO 8601** verwenden
- Datum und Zeit mit grossem `T` trennen
- UTC-Zeitstempel mit grossem `Z` abschliessen
- Zeitstempel immer in UTC speichern, Lokalisierung beim Client

```json
// ✓ Richtig
{ "created_at": "2024-01-15T10:30:00Z" }

// ✗ Falsch — numerischer Unix Timestamp (wie Stripe)
{ "created": 1483565364 }

// ✗ Falsch — Kleinbuchstaben
{ "created_at": "2024-01-15t10:30:00z" }
```

> **Hinweis:** Stripe verwendet Unix Integer-Timestamps (`"created": 1483565364`). Das ist in JavaScript-nahen Ökosystemen verbreitet, hat aber Nachteile: nicht menschenlesbar, kein eingebautes Timezone-Handling und kein direktes Mapping auf OpenAPI `date-time`. Wir verwenden ISO 8601 als universelleren Standard.

---

<a name="r-255"></a>
### #255 · SOLLTE · Geeignete Datum/Zeit-Formate wählen

| Format | Verwendung | Beispiel |
|---|---|---|
| `date-time` | Exakter Zeitpunkt (UTC) | Bestellzeitpunkt, Lieferzeitpunkt |
| `date` | Nur Datum ohne Uhrzeit | Geburtstag, Lieferdatum |
| `time-local` | Lokale Uhrzeit (ohne UTC) | Öffnungszeiten |
| `date-time-local` | Lokaler Zeitpunkt + Zeitzone separat | Kampagnenstartzeit |

---

<a name="r-127"></a>
### #127 · SOLLTE · Standardformate für Zeitdauern verwenden

Zeitdauern und Intervalle müssen als ISO 8601 Strings dargestellt werden:

```
"P1DT3H4S"          # 1 Tag, 3 Stunden, 4 Sekunden
"PT30M"             # 30 Minuten
"2024-01-01T00:00:00Z/2024-12-31T23:59:59Z"   # Intervall
"2024-01-01T00:00:00Z/P1Y"                    # Anfang + Dauer
```

Query-Parameter für Zeitintervalle: `{feld}_between` statt `{feld}_before` + `{feld}_after`.

---

<a name="r-170"></a>
### #170 · MUSS · Standardformate für Land, Sprache, Währung

| Datentyp | Standard | Format | Beispiel |
|---|---|---|---|
| Land | ISO 3166-1 alpha-2 | `iso-3166-alpha-2` | `"DE"`, `"GB"` |
| Sprache | ISO 639-1 | `iso-639-1` | `"de"`, `"en"` |
| Sprache + Region | BCP 47 | `bcp47` | `"de-AT"`, `"en-GB"` |
| Währung | ISO 4217 | `iso-4217` | `"EUR"`, `"USD"` |

---

<a name="r-244"></a>
### #244 · SOLLTE · Content Negotiation unterstützen

Wenn eine Ressource in verschiedenen Formaten geliefert werden kann, soll Content Negotiation über Standard-HTTP-Header verwendet werden:

```
Accept: application/json
Accept: application/pdf
Accept-Language: de
Accept-Encoding: gzip
```

---

<a name="r-144"></a>
### #144 · SOLLTE · UUIDs nur wenn notwendig verwenden

UUIDs sind sinnvoll für dezentrale ID-Generierung ohne Koordination. Sie haben aber Nachteile: schwer lesbar, nicht sortierbar, hoher Speicherverbrauch. Alternativen prüfen, z.B. serverseitige ID-Generierung via POST.

---

<a name="kap-5"></a>
## 5. URLs

<a name="c-01"></a>
### ✦ C-01 · MUSS · URL-Versionierung verwenden (ersetzt #113, #114, #115)

**Jeder API-Pfad muss die Major-Version im Pfad enthalten:**

```
/v1/orders
/v1/order-items/{id}
/v2/orders          ← Breaking Change → neue Major Version
```

- Nur **Major Versions** im Pfad (`v1`, `v2`, `v3`)
- **Keine** Media Type Versioning (`Accept: application/vnd.api+json;version=2`)
- **Keine** Header-Versionierung
- Neue Major Version nur bei wirklich inkompatiblen Breaking Changes

**Warum?** URL-Versionierung ist für Clients einfacher zu verstehen, in Logs klar erkennbar und erfordert keine spezielle Header-Konfiguration.

---

<a name="r-134"></a>
### #134 · MUSS · Ressourcennamen im Plural

Ressourcen sind immer im Plural:

```
/v1/orders          ✓ Richtig
/v1/order           ✗ Falsch (Singular)
/v1/order-items     ✓ Richtig
```

---

<a name="r-228"></a>
### #228 · MUSS · URL-kompatible Ressourcen-IDs

IDs in URLs dürfen nur enthalten: `[a-zA-Z0-9:._\-/]*`

Keine Sonderzeichen, keine Leerzeichen, keine leeren Werte.

---

<a name="r-129"></a>
### #129 · MUSS · kebab-case für Pfadsegmente

Pfadsegmente bestehen nur aus Kleinbuchstaben und Bindestrichen:

```
/v1/order-items         ✓ kebab-case
/v1/orderItems          ✗ camelCase
/v1/order_items         ✗ snake_case
```

---

<a name="r-136"></a>
### #136 · MUSS · Normalisierte Pfade ohne Trailing Slashes

```
/v1/orders/123          ✓
/v1/orders/123/         ✗ Trailing Slash verboten
/v1//orders/123         ✗ Leeres Segment verboten
```

---

<a name="r-141"></a>
### #141 · MUSS · URLs frei von Verben halten

URLs beschreiben **Ressourcen**, nicht **Aktionen**:

```
GET  /v1/orders              ✓ Liste abrufen
POST /v1/orders              ✓ Erstellen
POST /v1/order-cancellations ✓ Stornierung als Ressource

GET  /v1/getOrders           ✗ Verb in URL
POST /v1/cancelOrder         ✗ Verb in URL
```

---

<a name="r-138"></a>
### #138 · MUSS · Aktionen vermeiden — in Ressourcen denken

REST modelliert Ressourcen, nicht Prozeduraufrufe:

```
PUT /v1/article-locks/{article-id}   ✓ Ressource
POST /v1/articles/{id}/lock          ✗ Aktion
```

---

<a name="r-142"></a>
### #142 · MUSS · Domänenspezifische Ressourcennamen

Namen sollen den Geschäftskontext widerspiegeln:

```
/v1/sales-order-items    ✓ Klar und spezifisch
/v1/items                ✗ Zu generisch
```

---

<a name="r-143"></a>
### #143 · MUSS · Ressourcen via Pfadsegmente identifizieren

```
/v1/orders/{order-id}/items/{item-id}
```

Jedes Teilsegment muss für sich allein eine gültige Ressource sein.

---

<a name="r-130"></a>
### #130 · MUSS · snake_case für Query-Parameter

```
?page_size=20    ✓
?pageSize=20     ✗ camelCase verboten
```

---

<a name="r-137"></a>
### #137 · MUSS · Konventionelle Query-Parameter verwenden

| Parameter | Bedeutung |
|---|---|
| `q` | Generische Suchanfrage |
| `sort` | Sortierung: `+created_at` (asc), `-created_at` (desc) |
| `fields` | Feldauswahl: `?fields=id,status,created_at` |
| `embed` | Sub-Ressourcen einbetten |
| `cursor` | Cursor für Pagination |
| `limit` | Maximale Anzahl Ergebnisse |

---

### #135 · SOLLTE · `/api` nicht als Basispfad

```
/v1/orders          ✓
/api/v1/orders      ✗ Unnötiger /api Präfix
```

---

### #140 · SOLLTE · Nützliche und notwendige Ressourcen definieren

Eine Ressource sollte 90% der Anwendungsfälle abdecken. Zu granulare oder zu generische Ressourcen vermeiden. Neue Ressource erst einführen wenn ein konkreter, aktueller Bedarf besteht — nicht auf Vorrat (YAGNI, siehe C-08).

---

### #139 · SOLLTE · Vollständige Geschäftsprozesse modellieren

Eine API sollte alle Ressourcen eines Geschäftsprozesses enthalten, damit Clients den Ablauf nachvollziehen können.

---

### #146 · SOLLTE · Anzahl Ressourcentypen begrenzen

Erfahrungswert: gut designte APIs haben 4–8 Ressourcentypen.

---

### #147 · SOLLTE · Sub-Ressource-Ebenen begrenzen

Maximal **3 Ebenen** Verschachtelung:

```
/v1/orders/{id}/items/{item-id}/attachments/{att-id}    ✓ 3 Ebenen max.
/v1/a/{id}/b/{id}/c/{id}/d/{id}                        ✗ Zu tief
```

---

### #145 · KANN · Verschachtelte URLs in Betracht ziehen

Nested URLs nur wenn die Sub-Ressource ohne Elternressource nicht existiert.

---

### #241 · KANN · Zusammengesetzte Schlüssel als Ressourcen-ID

```
/v1/price-advices/{sku}/{sales-channel}
```

---

<a name="kap-6"></a>
## 6. JSON Payload

<a name="r-167"></a>
### #167 · MUSS · JSON als Datenformat verwenden

Alle Request- und Response-Bodies verwenden JSON (RFC 7159):

- UTF-8 Encoding
- Keine duplizierten Property-Namen
- Immer JSON-Objekt als Top-Level-Struktur (nie direkt ein Array)

---

<a name="r-118"></a>
### #118 · MUSS · snake_case für Property-Namen (niemals camelCase)

```json
// ✓ Richtig
{ "order_id": "123", "created_at": "2024-01-15T10:30:00Z" }

// ✗ Falsch — camelCase wie bei Stripe
{ "orderId": "123", "createdAt": "2024-01-15T10:30:00Z" }
```

Regex: `^[a-z_][a-z_0-9]*$`

> **Hinweis:** Stripe verwendet konsequent camelCase, weil ihre Client-Libraries primär auf JavaScript ausgerichtet sind. Für enterprise B2B APIs ist snake_case der breitere Industrie-Standard — bestätigt von Adidas, GitHub, Twilio und AWS.

---

<a name="c-02"></a>
### ✦ C-02 · MUSS NICHT · Kein HATEOAS (ersetzt #163, #164, #165)

**Keine** hypermedia controls in Response Bodies:

```json
// ✗ Verboten — kein HATEOAS
{
  "id": "123",
  "_links": { "self": { "href": "/orders/123" } }
}

// ✓ Richtig — nur Daten
{
  "id": "123",
  "status": "OPEN"
}
```

REST Maturity Level 2 (Ressourcen + HTTP-Methoden). Bestätigt durch Adidas und Stripe — beide produktiven APIs weltweit ohne einen einzigen `_links`-Block.

---

<a name="r-174"></a>
### #174 · MUSS · Gemeinsame Feldnamen verwenden

| Feldname | Typ | Bedeutung |
|---|---|---|
| `id` | `string` | Eindeutiger, unveränderlicher Bezeichner |
| `{entity}_id` | `string` | Referenz auf andere Ressource (`partner_id`) |
| `created_at` | `string` (date-time) | Erstellungszeitpunkt |
| `modified_at` | `string` (date-time) | Letzter Änderungszeitpunkt |
| `etag` | `string` | ETag für optimistisches Locking |

---

<a name="c-07"></a>
### ✦ C-07 · SOLLTE · `metadata`-Feld für erweiterbare Ressourcen *(Stripe)*

Alle mutierbaren Ressourcen SOLLTEN ein optionales `metadata`-Feld unterstützen für strukturierte Zusatzdaten ohne Breaking Changes:

```json
{
  "id": "ord_123",
  "status": "OPEN",
  "metadata": {
    "external_ref": "ERP-456",
    "campaign": "summer24",
    "cost_center": "CC-001"
  }
}
```

**Regeln für `metadata`:**
- Maximal 50 Key-Value-Paare pro Ressource
- Keys: snake_case, max. 40 Zeichen, keine eckigen Klammern `[ ]`
- Values: nur Strings, max. 500 Zeichen
- **Keine sensitiven Daten** (Passwörter, Tokens, Bankdaten)
- Server speichert und gibt zurück — keine Verarbeitungslogik

```yaml
# OpenAPI Schema
metadata:
  type: object
  additionalProperties:
    type: string
    maxLength: 500
  maxProperties: 50
  description: |
    Optionale Key-Value-Paare für Zusatzdaten.
    Keine sensitiven Informationen speichern.
```

---

<a name="c-11"></a>
### ✦ C-11 · SOLLTE · `description` und `metadata` klar trennen *(Stripe)*

| Feld | Typ | Zweck | Sichtbarkeit |
|---|---|---|---|
| `description` | `string` | Menschenlesbarer Freitext | Ggf. im UI/E-Mail angezeigt |
| `metadata` | `object` | Maschinenlesbare Key-Value-Daten | Nur intern / API |

```json
{
  "id": "ord_123",
  "description": "2 Shirts für Kundenbestellung Frühjahr",
  "metadata": { "erp_order_id": "ERP-456", "channel": "web" }
}
```

Nie Metadaten in `description` schreiben und nie `description` für maschinenlesbare Daten missbrauchen.

---

<a name="r-235"></a>
### #235 · SOLLTE · `_at`-Suffix für Datum/Zeit-Properties

```json
{ "created_at": "2024-01-15T10:30:00Z", "shipped_at": "2024-01-16T08:00:00Z" }
```

---

<a name="r-240"></a>
### #240 · SOLLTE · Enum-Werte in UPPER_SNAKE_CASE

```yaml
status:
  type: string
  enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]
```

---

<a name="r-120"></a>
### #120 · SOLLTE · Array-Namen im Plural

```json
{ "items": [...], "addresses": [...] }
```

---

<a name="r-123"></a>
### #123 · MUSS · Gleiche Semantik für `null` und fehlende Properties

Fehlendes Feld `{}` und `{"field": null}` müssen identisch behandelt werden — keine unterschiedlichen Semantiken.

---

<a name="r-122"></a>
### #122 · MUSS · `null` nicht für Boolean-Properties

```json
// ✗ Falsch
{ "accepted_terms": null }

// ✓ Richtig — Enum verwenden
{ "accepted_terms": "UNDECIDED" }   // Enum: ACCEPTED, DECLINED, UNDECIDED
```

---

<a name="r-124"></a>
### #124 · SOLLTE · `null` nicht für leere Arrays

```json
{ "items": [] }    // ✓
{ "items": null }  // ✗
```

---

<a name="r-216"></a>
### #216 · SOLLTE · Maps mit `additionalProperties` definieren

```yaml
translations:
  type: object
  additionalProperties:
    type: string
  description: Schlüssel sind BCP-47 Sprachcodes (z.B. "de", "en-GB")
```

---

<a name="r-252"></a>
### #252 · SOLLTE · Einheitliches Schema für Lesen und Schreiben

Dasselbe Schema für GET und POST/PUT/PATCH — Unterschiede via `readOnly: true` / `writeOnly: true`.

---

<a name="r-172"></a>
### #172 · SOLLTE · Standard-Medientypen verwenden

| Content-Type | Verwendung |
|---|---|
| `application/json` | Standard JSON |
| `application/problem+json` | Fehler (RFC 7807) |
| `application/pdf` | PDF-Dokumente |
| `multipart/form-data` | Datei-Uploads |

---

<a name="kap-7"></a>
## 7. HTTP-Anfragen

<a name="r-148"></a>
### #148 · MUSS · HTTP-Methoden korrekt verwenden

| Methode | Semantik | Idempotent | Sicher |
|---|---|---|---|
| `GET` | Ressource lesen | ✓ | ✓ |
| `POST` | Ressource erstellen | ✗ | ✗ |
| `PUT` | Ressource vollständig ersetzen | ✓ | ✗ |
| `PATCH` | Ressource partiell ändern | ✗ | ✗ |
| `DELETE` | Ressource löschen | ✓ | ✗ |
| `HEAD` | Wie GET, nur Header | ✓ | ✓ |

**GET** darf keinen Request-Body haben. Bei komplexen Suchanfragen → POST mit Body.

---

<a name="r-149"></a>
### #149 · MUSS · Gemeinsame Methoden-Eigenschaften einhalten

- **Sicher (Safe):** GET, HEAD — dürfen den Zustand nicht verändern
- **Idempotent:** GET, PUT, DELETE — mehrfache Ausführung hat denselben Effekt

---

<a name="r-229"></a>
### #229 · SOLLTE · POST und PATCH idempotent gestalten

Idempotente POST/PATCH-Requests verhindern Duplikate bei Netzwerkfehlern — via `Idempotency-Key` Header oder Sekundärschlüssel.

---

<a name="r-231"></a>
### #231 · SOLLTE · Sekundärschlüssel für idempotentes POST

```json
POST /v1/orders
{
  "external_order_id": "EXT-2024-001",
  "items": [...]
}
```

Bei Wiederholung mit gleichem `external_order_id` → dieselbe Bestellung zurückgeben, nicht neu anlegen.

---

<a name="r-253"></a>
### #253 · KANN · Asynchrone Anfrageverarbeitung

Langläufige Operationen können asynchron verarbeitet werden:
1. `POST /v1/exports` → `202 Accepted` + `Location: /v1/exports/{job-id}`
2. `GET /v1/exports/{job-id}` → Status prüfen
3. Bei Fertigstellung → Ergebnis oder `303 See Other`

---

<a name="r-154"></a>
### #154 · MUSS · Collection-Format für Header und Query-Parameter definieren

#### Kontext

Viele Parameter können mehrere Werte gleichzeitig annehmen — zum Beispiel mehrere Statuswerte filtern, mehrere Felder sortieren oder mehrere IDs abfragen. Ohne eine dokumentierte Konvention, wie mehrere Werte in einem Parameter übergeben werden, entscheiden Entwickler das spontan und inkonsistent. Das Ergebnis: APIs in denen `?status=OPEN,CANCELLED`, `?status=OPEN&status=CANCELLED` und `?status[]=OPEN&status[]=CANCELLED` gleichzeitig im Einsatz sind — alle leicht unterschiedlich.

Diese Regel verlangt: **Jeder Parameter der mehrere Werte annehmen kann, muss in der OpenAPI-Spezifikation explizit dokumentieren, welches Format verwendet wird.**

#### Die vier Collection-Formate in OpenAPI

OpenAPI 3.1 kennt vier Serialisierungsformate für Arrays in Query-Parametern, gesteuert durch `style` und `explode`:

| Format | `style` | `explode` | Beispiel für `?status=OPEN,CANCELLED` |
|---|---|---|---|
| **csv** (Standard) | `form` | `false` | `?status=OPEN,CANCELLED` |
| **multi** | `form` | `true` | `?status=OPEN&status=CANCELLED` |
| **ssv** | `spaceDelimited` | `false` | `?status=OPEN%20CANCELLED` |
| **pipes** | `pipeDelimited` | `false` | `?status=OPEN\|CANCELLED` |

**Unsere Empfehlung: `csv` (kommagetrennt) als Standard** — lesbar, URL-freundlich und von den meisten HTTP-Clients direkt unterstützt. `multi` als Alternative wenn der API-Consumer ein Framework nutzt das Arrays automatisch explodiert (z.B. Spring, axios).

#### OpenAPI Spezifikation

```yaml
parameters:
  # csv — kommagetrennt (Standard, empfohlen)
  - name: status
    in: query
    style: form
    explode: false
    schema:
      type: array
      items:
        type: string
        enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]
    description: |
      Filtert nach Bestellstatus. Mehrere Werte kommagetrennt.
      Beispiel: ?status=OPEN,IN_PROGRESS

  # multi — Schlüssel wiederholen
  - name: tag
    in: query
    style: form
    explode: true
    schema:
      type: array
      items:
        type: string
    description: |
      Filtert nach Tags. Parameter wird pro Wert wiederholt.
      Beispiel: ?tag=sale&tag=new-arrival
```

#### Alle Query-Parameter aus #137 mit ihrem Collection-Format

Diese konventionellen Parameter aus Regel #137 haben ein festgelegtes Format:

| Parameter | Format | Beispiel | Erklärung |
|---|---|---|---|
| `sort` | csv mit Präfix | `?sort=+created_at,-status` | `+` aufsteigend, `-` absteigend, kommagetrennt |
| `fields` | csv | `?fields=id,status,created_at` | Feldauswahl, kommagetrennt |
| `embed` | csv | `?embed=items,customer` | Sub-Ressourcen einbetten, kommagetrennt |
| `cursor` | single | `?cursor=eyJpZCI6IjEyMyJ9` | Einzelwert, kein Array |
| `limit` | single | `?limit=20` | Einzelwert, kein Array |
| `q` | single | `?q=winter+jacket` | Suchbegriff, Leerzeichen URL-encoded |

#### Vollständige Beispiele

**Mehrere Statuswerte filtern (csv):**
```
GET /v1/orders?status=OPEN,IN_PROGRESS
→ Liefert Bestellungen mit Status OPEN oder IN_PROGRESS
```

**Mehrere Felder sortieren:**
```
GET /v1/orders?sort=-created_at,+status
→ Neueste zuerst, bei Gleichstand alphabetisch nach Status
```

**Felder kombinieren:**
```
GET /v1/orders?status=OPEN&sort=-created_at&fields=id,status,total_amount&limit=20
→ Offene Bestellungen, neueste zuerst, nur 3 Felder, max 20 Ergebnisse
```

**Mehrere IDs abfragen (multi-Format):**
```
GET /v1/orders?id=abc&id=def&id=ghi
→ Liefert genau diese drei Bestellungen
```

#### Was in der OpenAPI-Spec MUSS dokumentiert sein

Für jeden Parameter der Arrays akzeptiert:

```yaml
- name: status
  in: query
  required: false
  style: form          # MUSS angegeben sein
  explode: false       # MUSS angegeben sein
  schema:
    type: array        # MUSS array sein wenn mehrere Werte möglich
    minItems: 1
    maxItems: 10       # SOLLTE ein sinnvolles Limit haben
    items:
      type: string
  description: |       # MUSS das Format im Text beschreiben
    Kommagetrennte Liste von Statuswerten.
    Beispiel: ?status=OPEN,CANCELLED
  example: "OPEN,IN_PROGRESS"   # SOLLTE ein konkretes Beispiel haben
```

#### Header vs. Query-Parameter

Die Regel gilt für beide, aber es gibt einen wichtigen Unterschied:

- **Query-Parameter:** alle vier Formate (csv, multi, ssv, pipes) sind möglich
- **HTTP-Header:** in OpenAPI 3.x nur `style: simple` (`explode: false`) unterstützt — entspricht kommagetrennt

```yaml
# Header mit mehreren Werten — nur simple/csv möglich
parameters:
  - name: X-Custom-Tags
    in: header
    style: simple      # Einzige sinnvolle Option für Header in OpenAPI 3.x
    explode: false
    schema:
      type: array
      items:
        type: string
    example: "tag1,tag2,tag3"
```

#### Was NICHT erlaubt ist

```
# ✗ Kein dokumentiertes Format — Konsumenten müssen raten
/v1/orders?status=OPEN,CANCELLED    ohne OpenAPI style/explode Angabe

# ✗ PHP-Array-Notation — nicht in OpenAPI abbildbar
/v1/orders?status[]=OPEN&status[]=CANCELLED

# ✗ JSON-Array im Query-Parameter — schwer zu URL-encoden
/v1/orders?status=["OPEN","CANCELLED"]
```

---

<a name="r-236"></a>
### #236 · SOLLTE · Einfache Filter als Query-Parameter

```
GET /v1/orders?status=OPEN&customer_id=abc123
GET /v1/orders?created_at_between=2024-01-01/2024-12-31
```

---

<a name="r-237"></a>
### #237 · SOLLTE · Komplexe Filter als JSON-Body (POST)

```json
POST /v1/orders/search
{
  "filter": {
    "status": ["OPEN", "IN_PROGRESS"],
    "total_amount": { "gte": 100.00 }
  }
}
```

---

<a name="r-226"></a>
### #226 · MUSS · Implizite Response-Filterung dokumentieren

Wenn ein Endpunkt automatisch filtert (z.B. nur eigene Daten), muss dies in der Spec dokumentiert sein.

---

<a name="kap-8"></a>
## 8. HTTP-Statuscodes

<a name="r-243"></a>
### #243 · MUSS · Nur offizielle HTTP-Statuscodes

Nur Statuscodes aus offiziellen RFCs verwenden. Keine proprietären Codes.

---

<a name="r-151"></a>
### #151 · MUSS · Alle Statuscodes spezifizieren

Jeder Endpunkt muss alle möglichen Statuscodes mit Beispiel-Responses in OpenAPI dokumentiert haben.

---

<a name="r-150"></a>
### #150 · SOLLTE · Nur gebräuchliche Statuscodes verwenden

| Code | Bedeutung | Verwendung |
|---|---|---|
| `200 OK` | Erfolg | GET, PUT, PATCH |
| `201 Created` | Erstellt | POST (neue Ressource) |
| `202 Accepted` | Angenommen | Asynchrone Verarbeitung |
| `204 No Content` | Kein Inhalt | DELETE, PUT ohne Body |
| `207 Multi-Status` | Teilerfolg | Batch-Operationen |
| `400 Bad Request` | Ungültige Anfrage | Syntaxfehler |
| `401 Unauthorized` | Nicht authentifiziert | Kein/ungültiger Token |
| `403 Forbidden` | Keine Berechtigung | Fehlende Scopes |
| `404 Not Found` | Nicht gefunden | Unbekannte Ressource |
| `409 Conflict` | Konflikt | Optimistic Locking |
| `410 Gone` | Dauerhaft entfernt | Gelöschte Ressource |
| `422 Unprocessable Entity` | Semantischer Fehler | Validierungsfehler |
| `429 Too Many Requests` | Rate Limit | Mit `Retry-After` |
| `500 Internal Server Error` | Serverfehler | |
| `503 Service Unavailable` | Nicht verfügbar | Wartung / Überlast |

---

<a name="r-220"></a>
### #220 · MUSS · Spezifischsten Statuscode verwenden

`422 Unprocessable Entity` statt generischem `400 Bad Request`, wenn die Syntax korrekt aber die Semantik fehlerhaft ist.

---

<a name="r-152"></a>
### #152 · MUSS · Code 207 für Batch/Bulk-Requests

```json
POST /v1/orders/batch → 207 Multi-Status
{
  "items": [
    { "id": "1", "status": 201, "order": {...} },
    { "id": "2", "status": 422, "problem": {...} }
  ]
}
```

---

<a name="r-153"></a>
### #153 · MUSS · Code 429 mit Retry-After bei Rate Limits

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "You have exceeded 1000 requests per minute."
}
```

---

<a name="r-176"></a>
### #176 · MUSS · Problem JSON für alle Fehler (RFC 7807)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Das Feld 'quantity' muss grösser als 0 sein.",
  "instance": "/v1/orders/abc123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

| Feld | Pflicht | Beschreibung |
|---|---|---|
| `type` | ✓ | URI des Fehlertyps |
| `title` | ✓ | Kurze, menschenlesbare Fehlerbeschreibung |
| `status` | ✓ | HTTP-Statuscode als Zahl |
| `detail` | ✗ | Detaillierte Fehlerbeschreibung |
| `instance` | ✗ | URI der betroffenen Ressource |
| `trace_id` | ✗ (5xx) | Aus `traceparent` extrahiert — für Log-Korrelation |

---

<a name="r-177"></a>
### #177 · MUSS · Keine Stack Traces in Fehler-Responses

Stack Traces, Datenbankfehler oder interne Pfade dürfen niemals in Fehler-Responses erscheinen.

---

<a name="r-251"></a>
### #251 · SOLLTE · Keine Weiterleitungs-Codes

`301`, `302`, `307`, `308` nach Möglichkeit vermeiden. Korrekte URLs direkt zurückgeben.

---

<a name="kap-9"></a>
## 9. HTTP-Header

<a name="r-178"></a>
### #178 · MUSS · `Content-*` Header korrekt verwenden

```http
Content-Type: application/json
Content-Type: application/problem+json
Content-Encoding: gzip
```

---

<a name="c-03-04"></a>
### ✦ C-03 / C-04 · MUSS · W3C Trace Context (ersetzt #233 X-Flow-ID)

**Jeder Service MUSS den `traceparent`-Header propagieren:**

| Header | Level | Format |
|---|---|---|
| `traceparent` | **MUSS** | `00-{32hex traceId}-{16hex spanId}-{8bit flags}` |
| `tracestate` | SOLLTE | `vendor=value,other=data` |
| `baggage` | KANN | W3C Baggage für Kontext-Weitergabe |

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate:  company=backend-service
```

**Verhalten im Gateway (Gravitee):**
1. Eingehender Request **mit** `traceparent` → propagieren, nicht überschreiben
2. Eingehender Request **ohne** `traceparent` → neuen Trace generieren
3. `trace_id` und `span_id` in Access Logs schreiben
4. Bei 5xx-Fehler: `trace_id` in Problem JSON einbauen (siehe C-05)

**Warum W3C statt X-Flow-ID?** Offener Standard, unterstützt von Jaeger, Zipkin, Azure Monitor, OpenTelemetry Collector und allen modernen Observability-Plattformen.

---

<a name="c-05"></a>
### ✦ C-05 · SOLLTE · `trace_id` in 5xx Problem JSON Responses

```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

Die `trace_id` wird aus dem `traceparent`-Header extrahiert (die 32-stellige Hex-ID nach `00-`). Dies ermöglicht direkte Log-Korrelation beim Debugging.

---

<a name="r-132"></a>
### #132 · SOLLTE · kebab-case mit Grossbuchstaben für eigene HTTP-Header

```
Content-Type        ✓ Standard
traceparent         ✓ W3C Standard (Kleinbuchstaben korrekt)
X-Request-Id        ✓ Eigene Header in Title-Case
```

---

<a name="r-180"></a>
### #180 · SOLLTE · `Location` Header nach POST

```http
HTTP/1.1 201 Created
Location: /v1/orders/abc123
```

---

<a name="r-182"></a>
### #182 · KANN · ETag mit If-Match / If-None-Match

```http
GET /v1/orders/123 → ETag: "abc123def456"

PUT /v1/orders/123
If-Match: "abc123def456"   # Schlägt fehl wenn zwischenzeitlich geändert
→ 409 Conflict
```

---

<a name="r-230"></a>
### #230 · KANN · Idempotency-Key Header

```http
POST /v1/orders
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b
```

---

<a name="r-181"></a>
### #181 · KANN · Prefer Header

```http
Prefer: return=minimal          # Nur Statuscode, kein Body
Prefer: return=representation   # Vollständige Ressource zurück
Prefer: respond-async           # Asynchrone Verarbeitung
```

---

<a name="kap-10"></a>
## 10. Performance

<a name="r-227"></a>
### #227 · MUSS · Cacheable Endpunkte dokumentieren

```http
Cache-Control: max-age=3600, must-revalidate
Cache-Control: no-cache
```

---

<a name="r-156"></a>
### #156 · SOLLTE · gzip-Komprimierung unterstützen

```http
Accept-Encoding: gzip       # Request
Content-Encoding: gzip      # Response
```

---

<a name="r-157"></a>
### #157 · SOLLTE · Partial Responses via Feldauswahl

```
GET /v1/orders?fields=id,status,created_at
```

---

<a name="r-158"></a>
### #158 · SOLLTE · Einbetten von Sub-Ressourcen erlauben

```
GET /v1/orders/123?embed=items
→ { "id": "123", "items": [...] }
```

---

<a name="r-155"></a>
### #155 · SOLLTE · Bandbreite reduzieren

Kombination aus Komprimierung, Feldauswahl und Caching für minimale Datenübertragung.

---

<a name="kap-11"></a>
## 11. Paginierung

<a name="r-159"></a>
### #159 · MUSS · Paginierung für alle Collection-Ressourcen

Jeder Endpunkt, der eine Liste zurückgibt, muss Paginierung unterstützen. Keine unlimitierten Responses.

---

<a name="r-160"></a>
### #160 · SOLLTE · Cursor-basierte Paginierung bevorzugen

Cursor-basierte Paginierung ist stabiler als Offset (keine doppelten/fehlenden Einträge bei gleichzeitigen Änderungen):

```json
{
  "items": [
    { "id": "abc123", "status": "OPEN", "created_at": "2024-01-15T10:30:00Z" }
  ],
  "cursor": {
    "next": "eyJpZCI6ImFiYzEyMyJ9",
    "prev": null
  }
}
```

```
GET /v1/orders?cursor=eyJpZCI6ImFiYzEyMyJ9&limit=20
```

---

<a name="r-248"></a>
### #248 · SOLLTE · Standard Pagination Response Object

```json
{
  "items": [...],
  "cursor": {
    "next": "...",
    "prev": "..."
  }
}
```

---

<a name="r-254"></a>
### #254 · SOLLTE · Gesamtanzahl vermeiden

`total_count` vermeiden — teuer bei grossen Datensätzen. Cursor für Navigation verwenden.

---

<a name="kap-12"></a>
## 12. Kompatibilität und Erweiterbarkeit

<a name="r-106"></a>
### #106 · MUSS · Keine Breaking Changes

Bestehende API-Konsumenten dürfen nicht ohne Abstimmung brechen. Breaking Changes erfordern eine neue Major Version.

**Was ist ein Breaking Change?**

| Änderung | Breaking? |
|---|---|
| Pflichtfeld in Request hinzufügen | ✓ Breaking |
| Feld entfernen oder umbenennen | ✓ Breaking |
| Ressource umbenennen (`/orders` → `/purchase-orders`) | ✓ Breaking |
| Typ ändern (`string` → `integer`) | ✓ Breaking |
| Bedeutung eines Feldes ändern (ohne Umbenennung) | ✓ Breaking |
| Endpunkt entfernen | ✓ Breaking |
| Statuscode ändern | ✓ Breaking |
| Optionales Feld hinzufügen | ✗ Kompatibel |
| Neuen Endpunkt hinzufügen | ✗ Kompatibel |
| Enum-Wert hinzufügen | ✗ Kompatibel (wenn Client tolerant) |

---

<a name="c-10"></a>
### ✦ C-10 · MUSS · Vier Erweiterungsregeln einhalten *(Adidas)*

Jede Änderung an einer bestehenden API MUSS diese vier Regeln einhalten:

1. **Du DARFST NICHT etwas wegnehmen** — keine Properties, Endpunkte oder Enum-Werte entfernen
2. **Du DARFST NICHT Processing Rules ändern** — Semantik von Feldern bleibt stabil
3. **Du DARFST NICHT Optionales zu Pflicht machen** — existing clients würden brechen
4. **Alles was du hinzufügst MUSS optional sein** — neue Felder niemals required

> Diese Regeln gelten auch für Umbenennungen und URI-Änderungen. Namen und IDs sollen über die Zeit stabil bleiben — inklusive ihrer Semantik.

---

<a name="r-108"></a>
### #108 · MUSS · Clients auf Erweiterungen vorbereiten

Clients müssen so implementiert werden, dass sie unbekannte Properties ignorieren (Tolerant Reader Pattern nach Postel's Law, siehe C-09).

---

<a name="r-110"></a>
### #110 · MUSS · JSON-Objekte als Top-Level-Datenstruktur

```json
// ✓ Richtig
{ "items": [1, 2, 3], "cursor": {...} }

// ✗ Falsch — Array direkt als Top-Level
[1, 2, 3]
```

**Warum?** Ermöglicht späteres Hinzufügen von Metadaten ohne Breaking Change.

---

<a name="r-111"></a>
### #111 · MUSS · OpenAPI Spec als erweiterbar behandeln

Neue optionale Properties, Endpunkte und Enum-Werte können jederzeit hinzugefügt werden.

---

<a name="r-107"></a>
### #107 · SOLLTE · Kompatible Erweiterungen bevorzugen

Neue Funktionalität als optionale Erweiterungen hinzufügen, die bestehende Konsumenten ignorieren können.

---

<a name="r-109"></a>
### #109 · SOLLTE · APIs konservativ designen

- So wenig wie nötig exponieren (siehe C-08 YAGNI)
- Unnötige Flexibilität vermeidet Komplexität
- Einfache APIs sind wartbarer

---

<a name="r-112"></a>
### #112 · SOLLTE · Offene Enum-Listen verwenden

```yaml
status:
  type: string
  x-extensible-enum:
    - OPEN
    - COMPLETED
    - CANCELLED
  description: Neue Werte können hinzugefügt werden. Clients müssen unbekannte Werte tolerieren.
```

---

<a name="kap-13"></a>
## 13. Deprecation

<a name="r-187"></a>
### #187 · MUSS · Deprecation in API-Spezifikation markieren

```yaml
paths:
  /v1/legacy-orders:
    get:
      deprecated: true
      description: |
        **Deprecated** — Bitte auf /v2/orders migrieren.
        Sunset-Datum: 2025-06-30
```

---

<a name="r-185"></a>
### #185 · MUSS · Genehmigung der Konsumenten vor API-Abschaltung

Alle bekannten Konsumenten müssen informiert werden und dem Migrationszeitplan zustimmen.

---

<a name="r-186"></a>
### #186 · MUSS · Consent externer Partner

Externe Partner müssen dem Deprecation-Zeitplan explizit zustimmen.

---

<a name="r-188"></a>
### #188 · MUSS · Nutzung der deprecated API monitoren

Tatsächliche Nutzung messen, um sicherzustellen dass alle Konsumenten migriert haben.

---

<a name="r-191"></a>
### #191 · MUSS · Keine deprecated APIs neu verwenden

Neue Services dürfen keine als deprecated markierten Endpunkte verwenden.

---

<a name="r-189"></a>
### #189 · SOLLTE · Deprecation und Sunset Header

```http
Deprecation: true
Sunset: Sat, 30 Jun 2025 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

---

<a name="r-190"></a>
### #190 · SOLLTE · Monitoring für Deprecation und Sunset

Alerts wenn Sunset-Datum näher rückt und noch aktive Konsumenten vorhanden sind.

---

<a name="kap-14"></a>
## 14. Betrieb

<a name="r-192"></a>
### #192 · MUSS · OpenAPI-Spezifikation veröffentlichen

```
GET /openapi.yaml    # Spezifikation abrufbar
GET /docs            # Optional: Swagger UI
```

---

<a name="r-193"></a>
### #193 · SOLLTE · API-Nutzung monitoren

- Requests pro Endpunkt und Statuscode
- Latenz (p50, p95, p99)
- Fehlerrate
- Nutzung pro API-Konsument (für Deprecation-Monitoring)

---

<a name="eigene-regeln"></a>
## Übersicht: Alle eigenen Regeln

| ID | Level | Regel | Quelle | Ersetzt |
|---|---|---|---|---|
| C-01 | **MUSS** | URL-Versionierung: `/{version}/{resource}` | Eigene | #113, #114, #115 |
| C-02 | **MUSS NICHT** | Kein HATEOAS — kein `_links`, `href`, `self` | Eigene | #163, #164, #165, #161 |
| C-03 | **MUSS** | `traceparent` (W3C Trace Context) propagieren | Eigene | #233 |
| C-04 | SOLLTE | `tracestate` propagieren falls vorhanden | Eigene | #233 |
| C-05 | SOLLTE | `trace_id` in Problem JSON 5xx Responses | Eigene | neu |
| C-06 | **MUSS** | Scope-Format: `read:<resource>`, `write:<resource>` | Eigene | #225 |
| C-07 | SOLLTE | `metadata`-Feld für erweiterbare Ressourcen | Stripe | neu |
| C-08 | **MUSS** | Minimale API-Oberfläche (YAGNI-Prinzip) | Adidas | ergänzt #109 |
| C-09 | **MUSS** | Postel's Law für Server und Client | Adidas | ergänzt #108 |
| C-10 | **MUSS** | Vier Erweiterungsregeln (kein Wegnehmen, keine Pflichtfelder) | Adidas | schärft #106 |
| C-11 | SOLLTE | `description` vs. `metadata` klar trennen | Stripe | neu |
| C-12 | **MUSS** | API-Specs in Git mit CHANGELOG und Tags | Adidas | ergänzt #101 |

---

<a name="entfernte-regeln"></a>
## Entfernte Zalando-interne Regeln

| #ID | Grund |
|---|---|
| #234 | Verweist auf `zalandoapis.com` interne GitHub URLs |
| #223 | Functional Naming basiert auf Zalando-internem Component Registry |
| #224 | Hostname-Convention für `.zalandoapis.com` / `.zalan.do` Domains |
| #183 | Explizit Zalando-spezifische proprietäre Header-Liste |
| #173 | Zalando Money Object mit `jackson-datatype-money` |
| #249 | Zalando-spezifisches Adressformat |

---

*Version 2.0 — Basiert auf Zalando, Adidas und Stripe API Guidelines*  
*Quercheck: [Stripe API](https://docs.stripe.com/api) · [Adidas Guidelines](https://adidas.gitbook.io/api-guidelines)*








