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

# REST API Styleguide — Erklärungen auf Deutsch

> Basierend auf den [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/).  
> Zalando-interne Regeln wurden entfernt. Alle Regeln sind auf Deutsch erklärt.  
> Abweichungen und eigene Regeln sind explizit gekennzeichnet.

---

## Konventionen

| Begriff | Bedeutung |
|---|---|
| **MUSS** | Verpflichtend — keine Ausnahmen |
| **SOLLTE** | Empfohlen — Abweichungen müssen begründet werden |
| **KANN** | Optional — nach eigenem Ermessen |
| ~~Durchgestrichen~~ | Deaktivierte Zalando-Regel |
| ✦ EIGENE REGEL | Unsere Abweichung oder Ergänzung |

---

## Zusammenfassung unserer Abweichungen

| Zalando-Regel | Original | Unsere Regelung |
|---|---|---|
| #113 / #114 / #115 | Media Type Versioning | ✦ **MUSS** URL-Versionierung verwenden: `/v{n}/resource` |
| #163 / #164 / #165 | HATEOAS optional | ✦ **MUSS NICHT** — kein `_links`, `href`, `self` |
| #233 | X-Flow-ID | ✦ Ersetzt durch W3C `traceparent` + `tracestate` (OpenTelemetry) |
| #223 / #224 | Zalando Functional Naming | Entfernt — Zalando-intern |
| #183 | Zalando Proprietary Headers | Entfernt — Zalando-intern |
| #173 / #249 | Zalando Money / Address | Entfernt — Zalando-intern |

---

## 1. Allgemeine Richtlinien

### #100 · MUSS · API-First-Prinzip befolgen

APIs müssen **vor** der Implementierung spezifiziert werden — nicht danach. Das bedeutet:

- Die API-Spezifikation (OpenAPI) wird als erstes erstellt, bevor Code geschrieben wird
- Kolleginnen und Kollegen sowie Client-Entwickler geben frühzeitig Feedback
- Die API ist stabil, auch wenn sich die Implementierung dahinter ändert

**Warum?** APIs sind Verträge mit Konsumenten. Wer zuerst implementiert und dann dokumentiert, baut Abhängigkeiten ein, die später schwer zu ändern sind.

---

### #101 · MUSS · API-Spezifikation mit OpenAPI bereitstellen

Alle APIs müssen mit **OpenAPI 3.1** als einzelne, in sich geschlossene YAML-Datei spezifiziert werden.

- Die Datei muss versioniert in einem Source-Control-System (z.B. Git) liegen
- Keine externen Referenzen auf URLs die sich ändern könnten
- Die Spezifikation muss zusammen mit dem Service deployed werden

```yaml
openapi: 3.1.0
info:
  title: Order Management API
  version: 1.0.0
```

---

### #102 · SOLLTE · API-Benutzerhandbuch bereitstellen

Zusätzlich zur technischen Spezifikation sollte ein Benutzerhandbuch für API-Konsumenten existieren. Es sollte enthalten:

- Zweck und Anwendungsfälle der API
- Konkrete Beispiele zur Nutzung
- Typische Fehlerfälle und wie man sie behebt
- Architekturkontext und wichtige Abhängigkeiten

Das Handbuch wird über `#/externalDocs/url` in der OpenAPI-Spezifikation verlinkt.

---

### #103 · MUSS · APIs auf amerikanischem Englisch schreiben

Alle API-Bezeichnungen, Beschreibungen, Fehlermeldungen und Dokumentationen müssen auf **US-Englisch** verfasst sein. Das gilt für:

- Ressourcennamen (`/orders`, `/customers`)
- Property-Namen (`order_id`, `created_at`)
- OpenAPI `description`-Felder
- Fehlermeldungen und Problem-JSON-Texte

---

## 2. Meta-Informationen

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

### #116 · MUSS · Semantic Versioning verwenden

Die API-Spec-Version folgt dem Schema `MAJOR.MINOR.PATCH`:

| Änderung | Aktion |
|---|---|
| Breaking Change (inkompatibel) | MAJOR erhöhen: `1.x.x` → `2.0.0` |
| Neue Funktion (rückwärtskompatibel) | MINOR erhöhen: `1.2.x` → `1.3.0` |
| Bugfix / Typo in Doku | PATCH erhöhen: `1.2.3` → `1.2.4` |

**Wichtig:** Diese Versionsnummer betrifft die API-Spezifikationsdatei — nicht die URL-Version (die wir separat in der URL führen, siehe ✦ Eigene Regel).

---

### #215 · MUSS · API-Identifier bereitstellen

Jede API bekommt eine **global eindeutige, unveränderliche UUID** als `x-api-id`. Diese ändert sich nie — auch nicht bei Breaking Changes oder Umbenennung der API.

```yaml
info:
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70
```

**Warum?** Der Identifier erlaubt die lückenlose Nachverfolgung der API-Evolution über alle Versionen hinweg, unabhängig von Namensänderungen.

---

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

## 3. Sicherheit

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

**✦ Eigene Regel (C-06) — Scope-Namenskonvention** (ersetzt #225):

Format: `{aktion}:{ressource}`

| Beispiel | Bedeutung |
|---|---|
| `read:orders` | Bestellungen lesen |
| `write:orders` | Bestellungen anlegen/ändern |
| `admin:orders` | Administrative Operationen |

---

## 4. Datenformate

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

### #169 · MUSS · Standardformate für Datum/Zeit verwenden

- Immer RFC 3339 / ISO 8601 verwenden
- Datum und Zeit mit grossem `T` trennen
- UTC-Zeitstempel mit grossem `Z` abschliessen
- Zeitstempel immer in UTC speichern, Lokalisierung beim Client

```yaml
# Richtig
created_at: "2024-01-15T10:30:00Z"

# Falsch
created_at: 1705311000        # Numerischer Timestamp — verboten
created_at: "2024-01-15t10:30:00z"   # Kleinbuchstaben — verboten
```

---

### #255 · SOLLTE · Geeignete Datum/Zeit-Formate wählen

| Format | Verwendung | Beispiel |
|---|---|---|
| `date-time` | Exakter Zeitpunkt (UTC) | Bestellzeitpunkt, Lieferzeitpunkt |
| `date` | Nur Datum ohne Uhrzeit | Geburtstag, Lieferdatum |
| `time-local` | Lokale Uhrzeit (ohne UTC) | Öffnungszeiten |
| `date-time-local` | Lokaler Zeitpunkt + Zeitzone separat | Kampagnenstartzeit |

---

### #127 · SOLLTE · Standardformate für Zeitdauern verwenden

Zeitdauern und Intervalle müssen als ISO 8601 Strings dargestellt werden:

```yaml
# Dauer
"P1DT3H4S"          # 1 Tag, 3 Stunden, 4 Sekunden
"PT30M"             # 30 Minuten

# Intervall (Anfang/Ende)
"2024-01-01T00:00:00Z/2024-12-31T23:59:59Z"

# Intervall (Anfang + Dauer)
"2024-01-01T00:00:00Z/P1Y"    # 1 Jahr ab Jahresanfang
```

Query-Parameter für Zeitintervalle: `{feld}_between` statt `{feld}_before` + `{feld}_after`.

---

### #170 · MUSS · Standardformate für Land, Sprache, Währung verwenden

| Datentyp | Standard | Format | Beispiel |
|---|---|---|---|
| Land | ISO 3166-1 alpha-2 | `iso-3166-alpha-2` | `"DE"`, `"GB"` |
| Sprache | ISO 639-1 | `iso-639-1` | `"de"`, `"en"` |
| Sprache + Region | BCP 47 | `bcp47` | `"de-AT"`, `"en-GB"` |
| Währung | ISO 4217 | `iso-4217` | `"EUR"`, `"USD"` |

---

### #244 · SOLLTE · Content Negotiation unterstützen

Wenn eine Ressource in verschiedenen Formaten geliefert werden kann (JSON, PDF, CSV), soll Content Negotiation über Standard-HTTP-Header verwendet werden:

```
Accept: application/json
Accept: application/pdf
Accept-Language: de
Accept-Encoding: gzip
```

---

### #144 · SOLLTE · UUIDs nur wenn notwendig verwenden

UUIDs sind sinnvoll für dezentrale ID-Generierung ohne Koordination. Sie haben aber Nachteile: schwer lesbar, nicht sortierbar, hoher Speicherverbrauch. Alternativen prüfen, z.B. serverseitige ID-Generierung via POST.

---

## 5. URLs

### ✦ EIGENE REGEL C-01 · MUSS · URL-Versionierung verwenden

> Ersetzt und kehrt #113, #114, #115 um

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

### #134 · MUSS · Ressourcennamen im Plural

Ressourcen sind immer im Plural:

```
/v1/orders          ✓ Richtig
/v1/order           ✗ Falsch (Singular)
/v1/order-items     ✓ Richtig
```

---

### #228 · MUSS · URL-kompatible Ressourcen-IDs verwenden

IDs in URLs dürfen nur enthalten: `[a-zA-Z0-9:._\-/]*`

Keine Sonderzeichen, keine Leerzeichen, keine leeren Werte.

---

### #129 · MUSS · kebab-case für Pfadsegmente

Pfadsegmente bestehen nur aus Kleinbuchstaben und Bindestrichen:

```
/v1/order-items         ✓ kebab-case
/v1/orderItems          ✗ camelCase
/v1/order_items         ✗ snake_case
/v1/OrderItems          ✗ PascalCase
```

---

### #136 · MUSS · Normalisierte Pfade ohne leere Segmente oder Trailing Slashes

```
/v1/orders/123          ✓ Richtig
/v1/orders/123/         ✗ Trailing Slash verboten
/v1//orders/123         ✗ Leeres Segment verboten
```

---

### #141 · MUSS · URLs frei von Verben halten

URLs beschreiben **Ressourcen**, nicht **Aktionen**. Verben gehören in die HTTP-Methode:

```
GET  /v1/orders             ✓ Liste abrufen
POST /v1/orders             ✓ Erstellen
POST /v1/order-cancellations ✓ Stornierung (Ressource!)

GET  /v1/getOrders          ✗ Verb in URL
POST /v1/cancelOrder        ✗ Verb in URL
```

---

### #138 · MUSS · Aktionen vermeiden — in Ressourcen denken

REST modelliert Ressourcen, nicht Prozeduraufrufe. Statt einer Aktion `lock` für Artikel lieber eine Ressource `article-locks`:

```
PUT /v1/article-locks/{article-id}      ✓ Ressource
POST /v1/articles/{id}/lock             ✗ Aktion
```

---

### #142 · MUSS · Domänenspezifische Ressourcennamen verwenden

Namen sollen den Geschäftskontext widerspiegeln:

```
/v1/sales-order-items       ✓ Klar: welche Bestellungen?
/v1/order-items             △ Unklar: welche Art?
/v1/items                   ✗ Zu generisch
```

---

### #143 · MUSS · Ressourcen und Sub-Ressourcen über Pfadsegmente identifizieren

```
/v1/orders/{order-id}/items/{item-id}
```

Jedes Teilsegment muss für sich allein eine gültige Ressource sein:
- `/v1/orders` — Liste aller Bestellungen
- `/v1/orders/{order-id}` — Einzelne Bestellung
- `/v1/orders/{order-id}/items` — Positionen dieser Bestellung

---

### #130 · MUSS · snake_case für Query-Parameter

```
?page_size=20           ✓ snake_case
?pageSize=20            ✗ camelCase
?page-size=20           ✗ kebab-case
```

---

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

### #135 · SOLLTE · `/api` nicht als Basispfad verwenden

```
/v1/orders              ✓ Direkt unter Root
/api/v1/orders          ✗ Unnötiger /api Präfix
```

---

### #140 · SOLLTE · Nützliche Ressourcen definieren

Eine Ressource sollte 90% der Anwendungsfälle abdecken. Zu granulare oder zu generische Ressourcen vermeiden.

---

### #139 · SOLLTE · Vollständige Geschäftsprozesse modellieren

Eine API sollte alle Ressourcen eines Geschäftsprozesses enthalten, damit Clients den Ablauf nachvollziehen können — ohne implizite Abhängigkeiten zwischen verschiedenen APIs.

---

### #146 · SOLLTE · Anzahl Ressourcentypen begrenzen

Erfahrungswert: gut designte APIs haben 4–8 Ressourcentypen. Mehr deutet auf fehlende Segmentierung hin.

---

### #147 · SOLLTE · Anzahl Sub-Ressource-Ebenen begrenzen

Maximal **3 Ebenen** Verschachtelung:

```
/v1/orders/{id}/items/{item-id}/attachments/{att-id}    ✓ 3 Ebenen
/v1/a/{id}/b/{id}/c/{id}/d/{id}                        ✗ Zu tief
```

---

### #145 · KANN · Verschachtelte URLs in Betracht ziehen

Nested URLs nur wenn die Sub-Ressource ohne Elternressource nicht existiert:

```
/v1/orders/{order-id}/items     ✓ Items gehören zur Bestellung
/v1/customers/{id}              ✓ Direkt erreichbar (eigene ID)
```

---

### #241 · KANN · Zusammengesetzte Schlüssel als Ressourcen-ID

Wenn eine Ressource durch mehrere Schlüssel identifiziert wird:

```
/v1/price-advices/{sku}/{sales-channel}
```

---

## 6. JSON Payload

### #167 · MUSS · JSON als Datenformat verwenden

Alle Request- und Response-Bodies verwenden JSON (RFC 7159 / RFC 7493):

- UTF-8 Encoding
- Keine duplizierten Property-Namen
- Immer JSON-Objekt als Top-Level-Struktur (nie direkt ein Array)

---

### #118 · MUSS · snake_case für Property-Namen (niemals camelCase)

```json
// ✓ Richtig
{ "order_id": "123", "created_at": "2024-01-15T10:30:00Z" }

// ✗ Falsch
{ "orderId": "123", "createdAt": "2024-01-15T10:30:00Z" }
```

Regex: `^[a-z_][a-z_0-9]*$` — Kleinbuchstaben, Underscores, Ziffern.

---

### ✦ EIGENE REGEL C-02 · MUSS NICHT · Kein HATEOAS

> Deaktiviert #163, #164, #165, #161

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
  "status": "open"
}
```

REST Maturity Level 2 (Ressourcen + HTTP-Methoden) — kein Level 3.

---

### #174 · MUSS · Gemeinsame Feldnamen verwenden

| Feldname | Typ | Bedeutung |
|---|---|---|
| `id` | `string` | Eindeutiger, unveränderlicher Bezeichner der Ressource |
| `{entity}_id` | `string` | Referenz auf eine andere Ressource (`partner_id`) |
| `created_at` | `string` (date-time) | Erstellungszeitpunkt |
| `modified_at` | `string` (date-time) | Letzter Änderungszeitpunkt |
| `etag` | `string` | ETag für optimistisches Locking |

---

### #235 · SOLLTE · `_at`-Suffix für Datum/Zeit-Properties

```json
// ✓ Richtig
{ "created_at": "2024-01-15T10:30:00Z", "shipped_at": "2024-01-16T08:00:00Z" }

// ✗ Vermeiden
{ "created": "...", "shipped": "..." }
```

---

### #240 · SOLLTE · Enum-Werte in UPPER_SNAKE_CASE

```yaml
status:
  type: string
  enum:
    - OPEN
    - IN_PROGRESS
    - COMPLETED
    - CANCELLED
```

---

### #120 · SOLLTE · Array-Namen im Plural

```json
{ "items": [...], "addresses": [...] }
```

---

### #123 · MUSS · Gleiche Semantik für `null` und fehlende Properties

Wenn ein Feld nicht `required` und `nullable` ist, müssen `{}` (fehlendes Feld) und `{"field": null}` identisch behandelt werden. Keine unterschiedlichen Semantiken für beide Fälle.

---

### #122 · MUSS · `null` nicht für Boolean-Properties verwenden

```yaml
// ✗ Falsch
accepted_terms: null

// ✓ Richtig — Enum verwenden
accepted_terms: UNDECIDED   # Enum: YES, NO, UNDECIDED
```

---

### #124 · SOLLTE · `null` nicht für leere Arrays verwenden

```json
// ✓ Richtig
{ "items": [] }

// ✗ Vermeiden
{ "items": null }
```

---

### #216 · SOLLTE · Maps mit `additionalProperties` definieren

```yaml
translations:
  type: object
  additionalProperties:
    type: string
  description: Schlüssel sind BCP-47 Sprachcodes (z.B. "de", "en-GB")
```

---

### #252 · SOLLTE · Einheitliches Schema für Lesen und Schreiben

Dasselbe Schema für GET und POST/PUT/PATCH verwenden. Unterschiede via:
- `readOnly: true` — nur in Responses (z.B. `id`, `created_at`)
- `writeOnly: true` — nur in Requests (z.B. `password`)

---

### #250 · SOLLTE · Auf JSON/Unicode-Inkompatibilitäten achten

Manche Datenbanken und Tools unterstützen nicht alle Unicode-Zeichen vollständig (z.B. PostgreSQL und `\u0000`). Gegebenenfalls validieren oder sanitisieren.

---

### #168 · KANN · Nicht-JSON-Medientypen mit datentypspezifischen Formaten

Binäre oder nicht-strukturierte Daten (Bilder, PDFs, Archive) können mit passenden Medientypen zurückgegeben werden. JSON bleibt das Standard-Format — andere Formate kommen per Content Negotiation hinzu.

---

### #172 · SOLLTE · Standard-Medientypen verwenden

| Content-Type | Verwendung |
|---|---|
| `application/json` | Standard JSON Responses |
| `application/problem+json` | Fehler-Responses (RFC 7807) |
| `application/pdf` | PDF-Dokumente |
| `multipart/form-data` | Datei-Uploads |

Keine eigenen Medientypen wie `application/x-company.order+json`.

---

## 7. HTTP-Anfragen

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

### #149 · MUSS · Gemeinsame Methoden-Eigenschaften einhalten

- **Sicher (Safe):** GET, HEAD — dürfen den Zustand nicht verändern
- **Idempotent:** GET, PUT, DELETE — mehrfache Ausführung hat denselben Effekt wie einmalige

---

### #229 · SOLLTE · POST und PATCH idempotent gestalten

Idempotente POST/PATCH-Requests verhindern doppelte Einträge bei Netzwerkfehlern. Empfohlene Methoden:

- `Idempotency-Key` Header (UUID vom Client)
- Sekundärschlüssel (z.B. externe Referenznummer)

---

### #231 · SOLLTE · Sekundärschlüssel für idempotentes POST

```json
POST /v1/orders
{
  "external_order_id": "EXT-2024-001",   // Sekundärschlüssel
  "items": [...]
}
```

Bei Wiederholung mit gleichem `external_order_id` → dieselbe Bestellung zurückgeben, nicht neu anlegen.

---

### #253 · KANN · Asynchrone Anfrageverarbeitung unterstützen

Langläufige Operationen können asynchron verarbeitet werden:
1. `POST /v1/exports` → `202 Accepted` + `Location: /v1/exports/{job-id}`
2. `GET /v1/exports/{job-id}` → Status prüfen
3. Bei Fertigstellung → `303 See Other` oder fertiges Ergebnis

---

### #154 · MUSS · Collection-Format für Header und Query-Parameter definieren

Wenn ein Parameter mehrere Werte annehmen kann, muss das Format dokumentiert sein:

```
?sort=+name,-created_at           # kommagetrennt
?fields=id,status,created_at      # kommagetrennt
```

---

### #236 · SOLLTE · Einfache Query-Sprachen via Query-Parameter

Einfache Filter als Query-Parameter:

```
GET /v1/orders?status=OPEN&customer_id=abc123
GET /v1/orders?created_at_between=2024-01-01/2024-12-31
```

---

### #237 · SOLLTE · Komplexe Query-Sprachen via JSON-Body

Bei komplexen Filterausdrücken POST mit Body verwenden (dokumentiert als GET-with-body):

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

### #226 · MUSS · Implizite Response-Filterung dokumentieren

Wenn ein Endpunkt automatisch Felder oder Einträge filtert (z.B. nur eigene Daten zurückgibt), muss dies in der API-Spezifikation dokumentiert sein.

---

## 8. HTTP-Statuscodes

### #243 · MUSS · Nur offizielle HTTP-Statuscodes verwenden

Nur Statuscodes verwenden, die in offiziellen RFCs definiert sind. Keine proprietären Codes.

---

### #151 · MUSS · Erfolgs- und Fehler-Responses spezifizieren

Jeder Endpunkt muss alle möglichen Statuscodes mit Beispiel-Responses in der OpenAPI-Spezifikation dokumentiert haben.

---

### #150 · SOLLTE · Nur die gebräuchlichsten Statuscodes verwenden

| Code | Bedeutung | Verwendung |
|---|---|---|
| `200 OK` | Erfolg | GET, PUT, PATCH |
| `201 Created` | Erstellt | POST (neue Ressource) |
| `202 Accepted` | Angenommen | Asynchrone Verarbeitung |
| `204 No Content` | Kein Inhalt | DELETE, PUT ohne Body |
| `207 Multi-Status` | Teilerfolg | Batch-Operationen |
| `301 Moved Permanently` | Umzug | API-Migration |
| `400 Bad Request` | Ungültige Anfrage | Validierungsfehler |
| `401 Unauthorized` | Nicht authentifiziert | Kein/ungültiger Token |
| `403 Forbidden` | Keine Berechtigung | Fehlende Scopes |
| `404 Not Found` | Nicht gefunden | Unbekannte Ressource |
| `405 Method Not Allowed` | Methode nicht erlaubt | |
| `409 Conflict` | Konflikt | Optimistic Locking |
| `410 Gone` | Dauerhaft entfernt | Gelöschte Ressource |
| `422 Unprocessable Entity` | Validierungsfehler | Semantisch invalid |
| `429 Too Many Requests` | Rate Limit | Mit Retry-After |
| `500 Internal Server Error` | Serverfehler | |
| `503 Service Unavailable` | Nicht verfügbar | Wartung / Überlast |

---

### #220 · MUSS · Spezifischsten Statuscode verwenden

`422 Unprocessable Entity` statt generisches `400 Bad Request`, wenn die Syntax korrekt ist aber die Semantik fehlerhaft.

---

### #152 · MUSS · Code 207 für Batch/Bulk-Requests

Wenn bei einer Batch-Operation einzelne Einträge fehlschlagen können:

```json
POST /v1/orders/batch
→ 207 Multi-Status
{
  "items": [
    { "id": "1", "status": 201, "order": {...} },
    { "id": "2", "status": 422, "problem": {...} }
  ]
}
```

---

### #153 · MUSS · Code 429 mit Retry-After Header bei Rate Limits

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

### #176 · MUSS · Problem JSON unterstützen (RFC 7807)

Alle Fehler-Responses müssen `application/problem+json` mit RFC 7807 Format verwenden:

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
| `type` | ✓ | URI des Fehlertyps (dokumentierte Fehlerseite) |
| `title` | ✓ | Kurze, menschenlesbare Fehlerbeschreibung |
| `status` | ✓ | HTTP-Statuscode als Zahl |
| `detail` | ✗ | Detaillierte Fehlerbeschreibung |
| `instance` | ✗ | URI der betroffenen Ressource |
| `trace_id` | ✗ (5xx) | Trace-ID für Log-Korrelation (aus `traceparent`) |

**✦ Eigene Regel (C-05):** Bei `5xx`-Fehlern SOLLTE `trace_id` aus dem `traceparent`-Header extrahiert und in die Problem-JSON-Response eingefügt werden.

---

### #177 · MUSS · Keine Stack Traces in Fehler-Responses

Stack Traces, Datenbankfehler oder interne Pfade dürfen niemals in Error-Responses erscheinen. Nur benutzerfreundliche Fehlermeldungen.

---

### #251 · SOLLTE · Keine Weiterleitungs-Codes verwenden

`301`, `302`, `307`, `308` nach Möglichkeit vermeiden. Stattdessen korrekte URLs direkt zurückgeben.

---

## 9. HTTP-Header

### #178 · MUSS · `Content-*` Header korrekt verwenden

```http
Content-Type: application/json
Content-Type: application/problem+json
Content-Encoding: gzip
Content-Length: 1234
```

---

### ✦ EIGENE REGEL C-03 / C-04 · W3C Trace Context (OpenTelemetry)

> Ersetzt #233 (X-Flow-ID)

**Jeder Service MUSS den `traceparent`-Header propagieren.**

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

1. Eingehender Request **mit** `traceparent` → propagieren (nicht überschreiben)
2. Eingehender Request **ohne** `traceparent` → neuen Trace generieren
3. `trace_id` und `span_id` in Access Logs schreiben
4. Bei 5xx-Fehler: `trace_id` in Problem JSON einbauen

**Warum W3C statt X-Flow-ID?** Offener Standard, unterstützt von allen modernen Observability-Tools (Jaeger, Zipkin, OpenTelemetry Collector, Azure Monitor, etc.).

---

### #132 · SOLLTE · kebab-case mit Grossbuchstaben für HTTP-Header

```
Content-Type        ✓
X-Request-Id        ✓
traceparent         ✓ (W3C Standard — Kleinbuchstaben korrekt)
content-type        ✗ (Kleinbuchstaben vermeiden bei eigenen Headern)
```

---

### #180 · SOLLTE · `Location` statt `Content-Location` Header

Nach einem `POST` (neue Ressource):

```http
HTTP/1.1 201 Created
Location: /v1/orders/abc123
```

---

### #182 · KANN · ETag mit If-Match / If-None-Match unterstützen

Für optimistisches Locking bei gleichzeitigen Änderungen:

```http
GET /v1/orders/123
→ ETag: "abc123def456"

PUT /v1/orders/123
If-Match: "abc123def456"    # Schlägt fehl wenn zwischenzeitlich geändert
→ 409 Conflict              # Falls ETag nicht mehr aktuell
```

---

### #230 · KANN · Idempotency-Key Header unterstützen

Für idempotente POST-Requests:

```http
POST /v1/orders
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b
```

---

### #181 · KANN · Prefer Header unterstützen

```http
Prefer: return=minimal          # Nur Statuscode, kein Body
Prefer: return=representation   # Vollständige Ressource zurück
Prefer: respond-async           # Asynchrone Verarbeitung
```

---

### #133 · KANN · Standard-HTTP-Header verwenden

Standard-Header wie `Authorization`, `Accept`, `Cache-Control` nach RFC-Standard verwenden.

---

### #179 · KANN · Content-Location Header verwenden

Wenn eine Response die kanonische URL der zurückgegebenen Ressource enthält:

```http
Content-Location: /v1/orders/abc123
```

---

## 10. Hypermedia

### #162 · MUSS · REST Maturity Level 2 verwenden

Unsere APIs implementieren **REST Level 2**: Ressourcen werden über URLs adressiert und mit Standard-HTTP-Methoden manipuliert.

---

### #217 · MUSS · Vollständige, absolute URIs für Ressource-Identifikation

Wenn auf andere Ressourcen verwiesen wird, immer absolute URIs verwenden:

```json
// ✓ Absolut
{ "order_url": "https://api.company.com/v1/orders/123" }

// ✗ Relativ — nicht verwenden
{ "order_url": "/v1/orders/123" }
```

---

### #166 · MUSS · Keine Link-Header mit JSON-Entities verwenden

```http
// ✗ Verboten
Link: </v1/orders?cursor=abc>; rel="next"
```

---

## 11. Performance

### #227 · MUSS · Cacheable Endpunkte dokumentieren

GET, HEAD und bestimmte POST-Endpunkte, die gecacht werden können, müssen mit Cache-Control Direktiven dokumentiert sein:

```http
Cache-Control: max-age=3600, must-revalidate
Cache-Control: no-cache   # Nicht cachebar
```

---

### #156 · SOLLTE · gzip-Komprimierung unterstützen

```http
# Request
Accept-Encoding: gzip

# Response
Content-Encoding: gzip
```

---

### #157 · SOLLTE · Partial Responses via Feldauswahl unterstützen

```
GET /v1/orders?fields=id,status,created_at
→ { "items": [{ "id": "123", "status": "OPEN", "created_at": "..." }] }
```

---

### #158 · SOLLTE · Optionales Einbetten von Sub-Ressourcen erlauben

```
GET /v1/orders/123?embed=items
→ { "id": "123", "items": [...] }   # Items direkt eingebettet
```

---

### #155 · SOLLTE · Bandbreite reduzieren und Antwortzeiten verbessern

Kombination aus Komprimierung, Feldauswahl und Caching verwenden, um unnötige Datenübertragung zu vermeiden.

---

## 12. Paginierung

### #159 · MUSS · Paginierung für alle Collection-Ressourcen

Jeder Endpunkt, der eine Liste zurückgibt, muss Paginierung unterstützen. Keine unlimitierten Responses.

---

### #160 · SOLLTE · Cursor-basierte Paginierung bevorzugen

Cursor-basierte Paginierung ist stabiler als Offset-basiert (keine doppelten/fehlenden Einträge bei gleichzeitigen Änderungen):

**Response-Format:**

```json
{
  "items": [
    { "id": "abc123", "status": "OPEN", "created_at": "2024-01-15T10:30:00Z" }
  ],
  "cursor": {
    "next": "eyJpZCI6ImFiYzEyMyJ9",   // Base64-kodierter Cursor
    "prev": null                        // null wenn erste Seite
  }
}
```

**Request:**
```
GET /v1/orders?cursor=eyJpZCI6ImFiYzEyMyJ9&limit=20
```

---

### #248 · SOLLTE · Pagination Response Page Object verwenden

Das Standard-Paginierungsobjekt:

```json
{
  "items": [...],           // Array der Ergebnisse
  "cursor": {
    "next": "...",          // Cursor für nächste Seite (null wenn letzte)
    "prev": "..."           // Cursor für vorherige Seite (null wenn erste)
  }
}
```

---

### #254 · SOLLTE · Gesamtanzahl vermeiden

`total_count` in Paginierungs-Responses vermeiden — teuer bei grossen Datensätzen und selten wirklich benötigt. Stattdessen Cursor für Navigation verwenden.

---

## 13. Kompatibilität

### #106 · MUSS · Keine Breaking Changes

Bestehende API-Konsumenten dürfen nicht ohne Abstimmung durch Änderungen an der API brechen. Breaking Changes erfordern eine neue Major Version.

**Was ist ein Breaking Change?**
- Pflichtfelder in Request hinzufügen
- Felder entfernen oder umbenennen
- Typen ändern (z.B. `string` → `integer`)
- Endpunkte entfernen
- Statuscodes ändern

**Was ist KEIN Breaking Change:**
- Optionale Felder hinzufügen
- Neue Endpunkte hinzufügen
- Enum-Werte hinzufügen (wenn Client damit umgehen kann)

---

### #108 · MUSS · Clients auf kompatible Erweiterungen vorbereiten

Clients müssen so implementiert werden, dass sie unbekannte Properties ignorieren (Robustheit nach Postel's Law: sei liberal beim Empfangen).

---

### #110 · MUSS · JSON-Objekte als Top-Level-Datenstruktur

```json
// ✓ Richtig — Objekt als Top-Level
{ "items": [1, 2, 3], "cursor": {...} }

// ✗ Falsch — Array direkt als Top-Level
[1, 2, 3]
```

**Warum?** Ermöglicht späteres Hinzufügen von Metadaten (Paginierung, etc.) ohne Breaking Change.

---

### #111 · MUSS · OpenAPI Spec als erweiterbar behandeln

Die Spezifikation muss offen für Erweiterungen sein. Neue optionale Properties, neue Endpunkte und neue Enum-Werte können jederzeit hinzugefügt werden.

---

### #107 · SOLLTE · Kompatible Erweiterungen bevorzugen

Neue Funktionalität als optionale Erweiterungen hinzufügen, die bestehende Konsumenten ignorieren können.

---

### #109 · SOLLTE · APIs konservativ designen

- So wenig wie nötig exponieren
- Unnötige Flexibilität vermeidet Kompliziertheit
- Einfache APIs sind wartbarer

---

### #112 · SOLLTE · Offene Enum-Listen verwenden

Statt fixer `enum`-Liste `x-extensible-enum` verwenden und Clients anweisen, unbekannte Werte zu tolerieren:

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

## 14. Deprecation

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

### #185 · MUSS · Genehmigung der Konsumenten vor API-Abschaltung

Bevor ein API-Endpunkt entfernt wird, müssen alle bekannten Konsumenten informiert werden und ihre Zustimmung zum Migrationszeitplan gegeben haben.

---

### #186 · MUSS · Consent externer Partner zum Deprecation-Zeitplan

Externe Partner (API-Konsumenten ausserhalb des Unternehmens) müssen explizit dem Deprecation-Zeitplan zustimmen.

---

### #188 · MUSS · Nutzung der deprecated API monitoren

Solange eine deprecated API aktiv ist, muss die tatsächliche Nutzung gemessen werden, um sicherzustellen, dass alle Konsumenten migriert haben.

---

### #191 · MUSS · Keine neuen deprecated APIs verwenden

Neue Services oder Funktionen dürfen keine als deprecated markierten API-Endpunkte verwenden.

---

### #189 · SOLLTE · Deprecation und Sunset Header

```http
Deprecation: true
Sunset: Sat, 30 Jun 2025 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

---

### #190 · SOLLTE · Monitoring für Deprecation und Sunset Header

Alerts wenn Sunset-Datum näher rückt und noch aktive Konsumenten vorhanden sind.

---

## 15. Betrieb

### #192 · MUSS · OpenAPI-Spezifikation mit Service veröffentlichen

Die aktuelle API-Spezifikation muss mit dem Deployment verfügbar sein:

```
GET /openapi.yaml       # Spezifikation abrufbar
GET /docs               # Optional: Swagger UI
```

---

### #193 · SOLLTE · API-Nutzung monitoren

Metriken zur API-Nutzung sammeln:
- Requests pro Endpunkt und Statuscode
- Latenz (p50, p95, p99)
- Fehlerrate
- Nutzung pro API-Konsument (für Deprecation-Monitoring)

---

## Übersicht: Eigene Regeln

| ID | Level | Regel | Ersetzt |
|---|---|---|---|
| C-01 | **MUSS** | URL-Versionierung: `/{version}/{resource}` | #113, #114, #115 |
| C-02 | **MUSS NICHT** | Kein HATEOAS — kein `_links`, `href`, `self` | #163, #164, #165, #161 |
| C-03 | **MUSS** | `traceparent` Header (W3C Trace Context) propagieren | #233 |
| C-04 | SOLLTE | `tracestate` Header propagieren falls vorhanden | #233 |
| C-05 | SOLLTE | `trace_id` in Problem JSON 5xx Responses | neu |
| C-06 | **MUSS** | Scope-Format: `read:<resource>`, `write:<resource>` | #225 |

---

## Entfernte Zalando-interne Regeln

| #ID | Grund |
|---|---|
| #234 | Verweist auf `zalandoapis.com` interne GitHub URLs |
| #223 | Functional Naming basiert auf Zalando-internem Component Registry |
| #224 | Hostname-Convention für `.zalandoapis.com` / `.zalan.do` Domains |
| #183 | Explizit Zalando-spezifische proprietäre Header-Liste |
| #173 | Zalando Money Object mit `jackson-datatype-money` Bibliothek |
| #249 | Zalando-spezifisches Adressformat (salutation, care_of, zip) |

---

*Version 1.0 — Basiert auf Zalando RESTful API Guidelines*  
*Sprache: Deutsch | Basis: [opensource.zalando.com/restful-api-guidelines](https://opensource.zalando.com/restful-api-guidelines/)*



