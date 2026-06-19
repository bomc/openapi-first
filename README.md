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

# Datenformate und Typen — Leitfaden für Entwickler

> Basierend auf den Regeln #118, #120, #122, #123, #124, #127, #144, #169, #170, #171, #174, #216, #235, #238, #240, #252, #255, C-07, C-11 des REST API Styleguides.

-----

## Warum einheitliche Datenformate?

Ein aufrufender Dienst liest ein Datumsfeld und erhält `"2024-01-15"`. Ein anderer Endpunkt derselben API liefert `"15.01.2024"`, ein dritter `1705276800`. Alle drei beschreiben denselben Tag — aber keiner ist ohne Kontext klar, und alle drei erfordern unterschiedliche Parsing-Logik auf Konsumentenseite.

Einheitliche Datenformate sind der Vertrag auf Feldebene. Sie legen fest wie Werte repräsentiert werden — unabhängig davon welche Sprache, welches Framework oder welches Team den Endpunkt konsumiert. Fehler auf dieser Ebene führen zu stillen Datenfehlern: ein falsch geparster Timestamp der um eine Stunde abweicht, ein Geldbetrag der durch Floating-Point-Arithmetik ungenau wird, ein Ländercode der nicht validiert und deshalb in verschiedenen Formaten gespeichert wird.

-----

## Property-Namen — snake_case (#118)

Alle Property-Namen in JSON-Requests und -Responses werden in **snake_case** geschrieben. Das gilt ausnahmslos — für alle Felder, auf allen Ebenen, in allen Endpunkten.

```json
// ✓ Richtig — snake_case
{
  "order_id": "ord_abc123",
  "customer_id": "cust_789",
  "total_amount": 149.95,
  "created_at": "2024-01-15T10:30:00Z",
  "is_gift_wrapping_requested": false
}

// ✗ Falsch — camelCase
{
  "orderId": "ord_abc123",
  "customerId": "cust_789",
  "totalAmount": 149.95,
  "createdAt": "2024-01-15T10:30:00Z",
  "isGiftWrappingRequested": false
}
```

Das gültige Zeichenmuster für Property-Namen lautet: `^[a-z_][a-z_0-9]*$`

- Beginnt mit Kleinbuchstabe oder Unterstrich
- Enthält nur Kleinbuchstaben, Unterstriche und Ziffern
- Keine Grossbuchstaben, keine Bindestriche, keine Leerzeichen

Abkürzungen werden wie reguläre Wörter behandelt:

```json
// ✓ Richtig
{ "api_version": "v1", "sku_id": "SKU-001", "vat_rate": 0.19 }

// ✗ Falsch
{ "APIVersion": "v1", "SKUId": "SKU-001", "VATRate": 0.19 }
```

-----

## Gemeinsame Feldnamen (#174)

Bestimmte Felder kommen in fast jeder Ressource vor. Für diese gibt es festgelegte Namen die über alle Endpunkte hinweg identisch verwendet werden:

|Feldname     |Typ                   |Bedeutung                                             |
|-------------|----------------------|------------------------------------------------------|
|`id`         |`string`              |Eindeutiger, unveränderlicher Bezeichner der Ressource|
|`{entity}_id`|`string`              |Serverinterner Verweis auf eine andere Ressource      |
|`created_at` |`string` (`date-time`)|Zeitpunkt der Erstellung — immer UTC                  |
|`updated_at` |`string` (`date-time`)|Zeitpunkt der letzten Änderung — immer UTC            |
|`etag`       |`string`              |Versionshash für optimistisches Locking               |

```json
{
  "id": "ord_abc123",
  "customer_id": "cust_789",
  "warehouse_id": "wh_001",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T14:22:00Z",
  "etag": "a1b2c3d4e5f6"
}
```

Das `{entity}_id`-Muster gilt für alle Referenzen auf andere Ressourcen. Der Präfix entspricht dem Namen der referenzierten Ressource im Singular:

```json
// ✓ Richtig — {entity}_id Muster
{ "customer_id": "cust_789", "product_id": "prod_abc", "warehouse_id": "wh_001" }

// ✗ Falsch — abweichende Suffixe
{ "customer_uid": "cust_789", "product_key": "prod_abc", "warehouseRef": "wh_001" }
```

-----

## Zahlen und Integer (#171, #238)

Jede numerische Property muss in der OpenAPI-Spezifikation ein explizites Format angeben. Ohne Format raten aufrufende Dienste die Präzision — und liegen oft falsch, was zu Datenverlust führt.

### Integer-Formate

```yaml
properties:
  quantity:
    type: integer
    format: int32        # 32-bit: -2.147.483.648 bis 2.147.483.647
    minimum: 1

  position:
    type: integer
    format: int64        # 64-bit: für grosse IDs und Zeitstempel
    example: 7721071004

  item_count:
    type: integer
    format: bigint       # Für sehr grosse Zahlen jenseits int64
```

Wann welches Format:

|Format  |Wertebereich   |Verwenden für                                         |
|--------|---------------|------------------------------------------------------|
|`int32` |±2,1 Milliarden|Mengen, Positionen, Alter, Zähler                     |
|`int64` |±9,2 Trillionen|Grosse IDs, Transaktionsnummern                       |
|`bigint`|Unbegrenzt     |Finanzwerte ohne Dezimalstellen, kryptografische Werte|

### Number-Formate

```yaml
properties:
  price:
    type: number
    format: decimal      # Exakte Dezimalzahl — für Geldbeträge
    minimum: 0
    example: 149.95

  weight_kg:
    type: number
    format: float        # 32-bit Fliesskomma — für Messwerte
    example: 1.75

  conversion_rate:
    type: number
    format: double       # 64-bit Fliesskomma — für wissenschaftliche Berechnungen
    example: 1.08432
```

**Wichtig bei Geldbeträgen:** `float` und `double` sind für Währungen ungeeignet. IEEE 754 Fliesskommazahlen können Dezimalwerte wie `0.10` nicht exakt darstellen — `0.1 + 0.2` ergibt `0.30000000000000004`. Für Geldbeträge wird `decimal` verwendet:

```json
// ✓ Richtig — decimal für Geldbeträge
{ "total_amount": 149.95, "tax_amount": 23.99, "currency_code": "EUR" }

// ✗ Gefährlich — float für Geldbeträge
{ "total_amount": 149.95000000000001 }   // mögliche Ausgabe nach float-Arithmetik
```

-----

## Datum und Zeit (#169, #255, #235)

### Das einheitliche Format — RFC 3339 / ISO 8601

Alle Datums- und Zeitwerte werden als Strings im RFC 3339 / ISO 8601 Format übertragen. Unix-Timestamps als Integer sind nicht zulässig.

```json
// ✓ Richtig — RFC 3339
{ "created_at": "2024-01-15T10:30:00Z" }

// ✗ Falsch — Unix Timestamp
{ "created_at": 1705312200 }

// ✗ Falsch — Deutsches Datumsformat
{ "created_at": "15.01.2024 10:30" }

// ✗ Falsch — Kleinbuchstaben
{ "created_at": "2024-01-15t10:30:00z" }
```

Drei unveränderliche Regeln:

- Datum und Zeit werden mit grossem `T` getrennt
- UTC-Zeitstempel enden mit grossem `Z`
- Zeitstempel werden immer in UTC gespeichert — die Lokalisierung ist Aufgabe des aufrufenden Dienstes

### Das richtige Format für jeden Anwendungsfall (#255)

Nicht jeder Zeitwert benötigt eine vollständige UTC-Zeit. Die Wahl des Formats hängt davon ab was fachlich gemeint ist:

**`date-time` — exakter absoluter Zeitpunkt in UTC**

Für Ereignisse die einen genauen Moment beschreiben — unabhängig vom Standort:

```json
{
  "created_at": "2024-01-15T10:30:00Z",
  "shipped_at": "2024-01-16T08:15:00Z",
  "expires_at": "2024-07-15T23:59:59Z"
}
```

**`date` — Kalendertag ohne Uhrzeit**

Für Datumsangaben bei denen die Uhrzeit keine Rolle spielt:

```json
{
  "delivery_date": "2024-01-20",
  "birth_date": "1985-03-22",
  "valid_until": "2024-12-31"
}
```

`delivery_date: "2024-01-20"` ist eindeutig der 20. Januar — unabhängig davon in welcher Zeitzone das Paket zugestellt wird. Als UTC-Timestamp `"2024-01-20T00:00:00Z"` wäre es in UTC+1 bereits der 20. Januar um 01:00 Uhr morgens — fachlich dasselbe, technisch verwirrend.

**`time-local` — lokale Uhrzeit ohne Zeitzonenbezug**

Für wiederkehrende Zeitangaben die sich mit der Ortszeit mitbewegen:

```json
{
  "opening_time": "09:00:00",
  "closing_time": "18:00:00"
}
```

Ein Geschäft öffnet um 09:00 Uhr Ortszeit — in München wie in Wien. Eine Zeitzone wäre hier falsch, weil sich die Öffnungszeit bei Sommerzeit nicht verschiebt.

**`date-time-local` — geplanter Zeitpunkt mit expliziter Zeitzone**

Für Zeitpunkte die an eine bestimmte Zeitzone gebunden sind und sich mit der Sommerzeit mitbewegen sollen:

```json
{
  "campaign_start": "2024-06-01T08:00:00",
  "campaign_timezone": "Europe/Berlin"
}
```

Als UTC gespeichert (`2024-06-01T06:00:00Z`) würde im Winter — wenn Berlin UTC+1 statt UTC+2 ist — die Kampagne um 07:00 Uhr starten statt um 08:00 Uhr. Die Zeitzone muss separat mitgeführt werden.

### Entscheidungsbaum für Datum/Zeit-Felder

```
Wird eine Uhrzeit benötigt?
├── Nein → date  (2024-01-20)
└── Ja
    ├── Absoluter Zeitpunkt — wann ist etwas passiert oder läuft ab?
    │   └── date-time UTC  (2024-01-15T10:30:00Z)
    │
    ├── Wiederkehrende Ortszeit — wann öffnet/schliesst etwas?
    │   └── time-local  (09:00:00)
    │
    └── Geplanter Zeitpunkt mit Zeitzonen-Semantik?
        └── date-time-local + timezone  (2024-06-01T08:00:00 + Europe/Berlin)
```

### Feldnamen für Datum/Zeit (#235)

Alle Properties die einen Zeitpunkt enthalten, erhalten den `_at`-Suffix:

```json
// ✓ Richtig — _at Suffix
{
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T14:22:00Z",
  "shipped_at": "2024-01-16T08:00:00Z",
  "cancelled_at": null
}

// ✗ Falsch — fehlender oder falscher Suffix
{
  "created": "2024-01-15T10:30:00Z",
  "update_time": "2024-01-15T14:22:00Z",
  "shipDate": "2024-01-16T08:00:00Z"
}
```

Reine Datumsfelder (ohne Uhrzeit) erhalten je nach Semantik einen beschreibenden Namen:

```json
{
  "delivery_date": "2024-01-20",
  "valid_until": "2024-12-31",
  "birth_date": "1985-03-22"
}
```

### Zeitdauern und Intervalle (#127)

Zeitdauern werden als ISO 8601 Duration Strings dargestellt — nicht als Sekunden-Integer:

```json
// ✓ Richtig — ISO 8601 Duration
{ "processing_time": "PT30M" }       // 30 Minuten
{ "validity_period": "P30D" }        // 30 Tage
{ "session_timeout": "PT1H30M" }     // 1 Stunde 30 Minuten
{ "contract_duration": "P1Y6M" }     // 1 Jahr 6 Monate

// ✗ Falsch — Sekunden als Integer
{ "processing_time": 1800 }          // Was ist 1800? Sekunden? Millisekunden?
```

Das ISO 8601 Format ist selbstbeschreibend: `P` leitet die Dauer ein, `T` trennt Datum von Uhrzeit innerhalb der Dauer:

```
P   = Period (Dauer)
1Y  = 1 Jahr
6M  = 6 Monate (oder Minuten nach T)
2W  = 2 Wochen
3D  = 3 Tage
T   = Trennzeichen für Uhrzeitkomponenten
4H  = 4 Stunden
5M  = 5 Minuten (nach T)
6S  = 6 Sekunden
```

Für Zeitintervalle (Anfang und Ende) gibt es zwei Notationen:

```json
// Anfang und Ende explizit
{ "valid_between": "2024-01-01T00:00:00Z/2024-12-31T23:59:59Z" }

// Anfang und Dauer
{ "valid_between": "2024-01-01T00:00:00Z/P1Y" }
```

Als Query-Parameter wird `{feld}_between` statt getrennter `before`/`after`-Parameter verwendet:

```
GET /v1/orders?created_at_between=2024-01-01T00:00:00Z/2024-12-31T23:59:59Z
```

-----

## Standardformate für internationale Felder (#170)

Für Länder, Sprachen und Währungen werden ausschliesslich die internationalen Normen verwendet:

|Datentyp        |Norm              |Format            |Beispiele                      |
|----------------|------------------|------------------|-------------------------------|
|Land            |ISO 3166-1 alpha-2|`iso-3166-alpha-2`|`"DE"`, `"CH"`, `"AT"`, `"GB"` |
|Sprache         |ISO 639-1         |`iso-639-1`       |`"de"`, `"en"`, `"fr"`         |
|Sprache + Region|BCP 47            |`bcp47`           |`"de-AT"`, `"en-GB"`, `"fr-CH"`|
|Währung         |ISO 4217          |`iso-4217`        |`"EUR"`, `"CHF"`, `"USD"`      |

```json
// ✓ Richtig — Normen
{
  "country_code": "DE",
  "language_code": "de",
  "locale": "de-AT",
  "currency_code": "EUR"
}

// ✗ Falsch — eigene Formate
{
  "country": "Deutschland",
  "language": "German",
  "currency": "Euro",
  "currency_symbol": "€"
}
```

In OpenAPI mit dem jeweiligen Format dokumentieren:

```yaml
properties:
  country_code:
    type: string
    format: iso-3166-alpha-2
    pattern: '^[A-Z]{2}$'
    example: "DE"

  currency_code:
    type: string
    format: iso-4217
    pattern: '^[A-Z]{3}$'
    example: "EUR"

  locale:
    type: string
    format: bcp47
    example: "de-AT"
```

-----

## Enumerationen (#240, #112)

Enum-Werte werden in **UPPER_SNAKE_CASE** geschrieben:

```yaml
status:
  type: string
  x-extensible-enum:
    - OPEN
    - IN_PROGRESS
    - COMPLETED
    - CANCELLED
  description: |
    Order status.
    New values may be added in future versions.
    Clients must handle unknown values gracefully.
```

```json
// ✓ Richtig
{ "status": "IN_PROGRESS" }

// ✗ Falsch — camelCase
{ "status": "inProgress" }

// ✗ Falsch — Kleinbuchstaben
{ "status": "in_progress" }

// ✗ Falsch — Leerzeichen
{ "status": "In Progress" }
```

### Offene Enum-Listen (#112)

Enum-Listen werden als offen deklariert: neue Werte können in zukünftigen API-Versionen hinzugefügt werden ohne einen Breaking Change auszulösen. Aufrufende Dienste müssen unbekannte Enum-Werte tolerieren — weder mit Fehler ablehnen noch als ungültigen Zustand behandeln:

```yaml
# x-extensible-enum statt enum — signalisiert offene Liste
status:
  type: string
  x-extensible-enum:
    - OPEN
    - IN_PROGRESS
    - COMPLETED
    - CANCELLED
    # Zukünftig möglich: PARTIALLY_DELIVERED, ON_HOLD, ...
```

Ein aufrufender Dienst implementiert die Toleranz explizit:

```javascript
// ✓ Robust — unbekannte Werte werden toleriert
switch (order.status) {
  case 'OPEN':      handleOpen(order); break;
  case 'COMPLETED': handleCompleted(order); break;
  default:
    // Unbekannter Status: ignorieren oder als generischen Zustand behandeln
    handleUnknown(order);
}

// ✗ Fehleranfällig — bricht bei neuen Enum-Werten
const KNOWN_STATUSES = ['OPEN', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED'];
if (!KNOWN_STATUSES.includes(order.status)) {
  throw new Error(`Unknown status: ${order.status}`);
}
```

-----

## Null-Werte und fehlende Felder (#123, #122, #124)

### Gleiche Semantik für null und fehlendes Feld (#123)

Fehlendes Feld und explizites `null` müssen für aufrufende Dienste identisch behandelt werden:

```json
// Diese beiden sind semantisch äquivalent
{ "id": "ord_123" }
{ "id": "ord_123", "discount_percentage": null }
```

Das hat direkte Konsequenzen für OpenAPI: Ein optionales Feld wird nie gleichzeitig als `nullable: true` und ohne `required` definiert — das würde zwei verschiedene Abwesenheitszustände mit potenziell unterschiedlichen Semantiken erzeugen:

```yaml
# ✓ Richtig — optional, aber niemals null
discount_percentage:
  type: number
  format: decimal
  # kein required → darf fehlen
  # kein nullable  → darf nicht explizit null sein

# ✗ Falsch — darf fehlen UND null sein (doppelte Semantik)
discount_percentage:
  type: number
  nullable: true   # Nicht kombinieren mit fehlendem required
```

### Boolean-Felder — kein null (#122)

Boolean-Felder kennen zwei Zustände: `true` und `false`. Ein dritter Zustand `null` ist kein Boolean-Wert sondern ein eigener fachlicher Zustand der einen eigenen Typ erfordert.

Ist das Boolean-Feld optional und bedeutet “fehlendes Feld” dasselbe wie “nicht gesetzt”, wird das Feld einfach weggelassen:

```json
// ✓ Noch keine Auswahl getroffen — Feld fehlt
{ "id": "ord_123", "status": "OPEN" }

// ✓ Aktiv gewählt
{ "id": "ord_123", "is_gift_wrapping_requested": true }

// ✗ Verboten
{ "id": "ord_123", "is_gift_wrapping_requested": null }
```

Existieren drei fachlich unterschiedliche Zustände, wird ein Enum verwendet:

```yaml
# Drei Zustände → Enum statt nullable Boolean
terms_acceptance:
  type: string
  enum: [ACCEPTED, DECLINED, PENDING]
```

### Leere Arrays — kein null (#124)

Ein leeres Array ist ein valider Zustand und wird als `[]` zurückgegeben — nicht als `null`:

```json
// ✓ Richtig — kein Eintrag vorhanden
{ "items": [], "tags": [] }

// ✗ Falsch — null statt leeres Array
{ "items": null, "tags": null }
```

-----

## Array-Namen (#120)

Array-Namen werden immer im Plural geschrieben:

```json
// ✓ Richtig
{
  "items": [...],
  "addresses": [...],
  "line_items": [...],
  "product_ids": [...]
}

// ✗ Falsch — Singular für Arrays
{
  "item": [...],
  "address": [...],
  "line_item": [...]
}
```

-----

## Maps und dynamische Schlüssel (#216)

Wenn ein Objekt als Key-Value-Map verwendet wird — also mit variablen Schlüsseln — wird es in OpenAPI mit `additionalProperties` definiert:

```yaml
# Übersetzungen — Schlüssel sind BCP-47 Sprachcodes
translations:
  type: object
  additionalProperties:
    type: string
  description: |
    Map of translations keyed by BCP-47 language code.
  example:
    de: "Bestellung"
    en: "Order"
    fr: "Commande"
```

```json
{
  "id": "prod_abc",
  "name": "Winter Jacket",
  "translations": {
    "de": "Winterjacke",
    "fr": "Veste d'hiver",
    "it": "Giacca invernale"
  }
}
```

-----

## UUIDs — nur wenn notwendig (#144)

UUIDs sind sinnvoll wenn IDs dezentral generiert werden müssen ohne Koordination zwischen Services. Sie haben aber Nachteile:

- Schwer lesbar in Logs und beim Debugging
- Nicht sortierbar nach Erstellungszeit (ausser UUID v7)
- Höherer Speicherverbrauch als numerische IDs
- Datenbankindizes werden fragmentiert

```json
// UUID — dezentrale Generierung, kein Koordinationsaufwand
{ "id": "e2ab873e-b295-11e9-9c02-68f728c1ba70" }

// Serverseitige ID — lesbar, sortierbar, kompakt
{ "id": "ord_abc123" }
```

Wenn IDs ausschliesslich serverseitig generiert werden und kein Bedarf an dezentraler Erzeugung besteht, wird UUID vermieden. Die Alternative ist eine serverseitig generierte ID mit fachlichem Präfix (`ord_`, `cust_`, `prod_`) die Typ und Kontext sofort erkennbar macht.

-----

## Einheitliches Schema für Lesen und Schreiben (#252)

Für Requests (POST/PUT/PATCH) und Responses (GET) wird dasselbe Schema verwendet. Unterschiede werden innerhalb des Schemas über `readOnly` und `writeOnly` ausgedrückt:

```yaml
components:
  schemas:
    Order:
      type: object
      properties:
        id:
          type: string
          readOnly: true       # Nur in Response — vom Server vergeben
          example: "ord_abc123"

        external_order_id:
          type: string
          writeOnly: false     # In Request und Response
          example: "ERP-2024-001"

        status:
          type: string
          readOnly: true       # Nur in Response — serverseitig gesteuert
          x-extensible-enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]

        total_amount:
          type: number
          format: decimal
          readOnly: true       # Nur in Response — serverseitig berechnet

        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'

        created_at:
          type: string
          format: date-time
          readOnly: true       # Nur in Response — vom Server gesetzt

        updated_at:
          type: string
          format: date-time
          readOnly: true       # Nur in Response — vom Server gesetzt
```

Dieses Schema wird für GET, POST und PUT gleichermassen verwendet. Ein separates `OrderRequest`-Schema ist nur dann nötig wenn strukturelle Unterschiede zwischen Lesen und Schreiben existieren, die nicht über `readOnly`/`writeOnly` ausgedrückt werden können.

-----

## Metadata-Felder (C-07, C-11)

Veränderbare Ressourcen SOLLTEN ein optionales `metadata`-Feld für strukturierte Zusatzdaten unterstützen. Das ermöglicht Erweiterbarkeit ohne Breaking Changes:

```json
{
  "id": "ord_abc123",
  "status": "OPEN",
  "metadata": {
    "erp_order_id": "ERP-2024-00847",
    "cost_center": "CC-001",
    "campaign": "summer24"
  }
}
```

`metadata` und `description` haben unterschiedliche Zwecke und dürfen nicht verwechselt werden:

|Feld         |Typ     |Zweck                           |Verarbeitung                                                 |
|-------------|--------|--------------------------------|-------------------------------------------------------------|
|`description`|`string`|Menschenlesbarer Freitext       |Wird ggf. im UI angezeigt                                    |
|`metadata`   |`object`|Maschinenlesbare Key-Value-Daten|Wird gespeichert und zurückgegeben — keine Verarbeitungslogik|

Regeln für `metadata`-Felder:

- Keys: snake_case, max. 40 Zeichen
- Values: nur Strings, max. 500 Zeichen
- Maximal 50 Key-Value-Paare pro Ressource
- Keine sensitiven Daten (Passwörter, Tokens, Bankdaten)

-----

## Vollständiges OpenAPI-Beispiel

Ein vollständig typisiertes Order-Schema das alle Konventionen dieses Leitfadens anwendet:

```yaml
components:
  schemas:
    Order:
      type: object
      required: [items]
      properties:
        id:
          type: string
          readOnly: true
          description: Server-assigned unique identifier.
          example: "ord_abc123"

        external_order_id:
          type: string
          maxLength: 100
          description: Optional client-provided identifier for idempotency.
          example: "ERP-2024-00847"

        customer_id:
          type: string
          description: Reference to the customer resource.
          example: "cust_789"

        status:
          type: string
          readOnly: true
          x-extensible-enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]
          description: |
            Current order status.
            New values may be added in future versions.

        total_amount:
          type: number
          format: decimal
          readOnly: true
          minimum: 0
          description: Total order amount including taxes.
          example: 149.95

        currency_code:
          type: string
          format: iso-4217
          pattern: '^[A-Z]{3}$'
          example: "EUR"

        country_code:
          type: string
          format: iso-3166-alpha-2
          pattern: '^[A-Z]{2}$'
          example: "DE"

        delivery_date:
          type: string
          format: date
          description: Requested delivery date (calendar day, no time component).
          example: "2024-01-20"

        is_gift_wrapping_requested:
          type: boolean
          nullable: false
          description: |
            Whether gift wrapping was requested.
            Omitted if no selection has been made yet.

        terms_acceptance:
          type: string
          enum: [ACCEPTED, DECLINED, PENDING]
          description: Status of terms and conditions acceptance.

        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'

        tags:
          type: array
          items:
            type: string
          description: Optional tags. Empty array if no tags assigned.
          example: []

        translations:
          type: object
          additionalProperties:
            type: string
          description: Translations keyed by BCP-47 language code.

        metadata:
          type: object
          propertyNames:
            pattern: '^[a-z][a-z0-9_]{0,39}$'
          additionalProperties:
            type: string
            maxLength: 500
          maxProperties: 50
          description: |
            Optional key-value pairs for extensibility.
            Do not store sensitive data.

        created_at:
          type: string
          format: date-time
          readOnly: true
          description: Creation timestamp in UTC.
          example: "2024-01-15T10:30:00Z"

        updated_at:
          type: string
          format: date-time
          readOnly: true
          description: Last modification timestamp in UTC.
          example: "2024-01-15T14:22:00Z"

        etag:
          type: string
          readOnly: true
          description: Version hash for optimistic locking.
          example: "a1b2c3d4e5f6"
```

-----

## Häufige Fehler

**camelCase statt snake_case.** Code-Generatoren aus Java- oder TypeScript-Klassen produzieren automatisch camelCase. Ohne explizite Konfiguration des Serialisierungs-Frameworks (`@JsonProperty`, `snake_case`-Konfiguration) werden alle Properties in camelCase ausgegeben.

**`float` für Geldbeträge verwenden.** `total_amount: 149.95` gespeichert als `float` kann als `149.95000000000001` zurückkommen. Bei Berechnungen akkumulieren sich diese Fehler. Für Geldbeträge wird immer `decimal` verwendet.

**Unix-Timestamps statt ISO 8601.** `"created_at": 1705312200` ist nicht menschenlesbar, erfordert Konvertierungslogik und hat kein eingebautes Timezone-Handling. ISO 8601 ist universell verständlich und direkt in OpenAPI als `date-time` typisierbar.

**Fehlenden `_at`-Suffix bei Zeitstempeln.** `"created"`, `"modified"`, `"updated"` statt `"created_at"`, `"updated_at"`. Ohne Konvention weiss der aufrufende Dienst nicht ob das Feld ein Datum, ein Zeitstempel oder ein anderer Wert ist.

**`nullable: true` ohne `required` für optionale Felder.** Das erzeugt doppelte Abwesenheitssemantik: fehlendes Feld und `null` könnten unterschiedlich interpretiert werden. Optionale Felder werden ohne `required` und ohne `nullable: true` definiert.

**Enum-Werte in camelCase oder Kleinbuchstaben.** `"status": "inProgress"` oder `"status": "in_progress"` statt `"status": "IN_PROGRESS"`. Inkonsistente Gross-/Kleinschreibung bei Enums ist eine häufige Quelle von Parsing-Fehlern.

**Ländernamen statt ISO-Codes.** `"country": "Germany"` statt `"country_code": "DE"`. Ländernamen variieren je nach Sprache und Schreibweise — `"Deutschland"`, `"Germany"`, `"Allemagne"` beschreiben alle dasselbe Land. ISO 3166-1 alpha-2 ist eindeutig, zweibuchstabig und sprachunabhängig.

**Array als Top-Level-Struktur.** `GET /v1/orders` gibt `[...]` direkt zurück statt `{ "items": [...] }`. Damit können später keine Pagination-Metadaten oder andere Felder ohne Breaking Change hinzugefügt werden.

-----

## Zusammenfassung

|Regel|Kernaussage                                                                  |
|-----|-----------------------------------------------------------------------------|
|#118 |snake_case für alle Property-Namen — niemals camelCase                       |
|#174 |Standard-Feldnamen: `id`, `{entity}_id`, `created_at`, `updated_at`, `etag`  |
|#171 |Explizites Format für alle Zahlen: `int32`, `int64`, `decimal`, `float`      |
|#238 |Standard-Formate für alle Typen — URI, UUID, E-Mail, Datum, Sprache          |
|#169 |Datum/Zeit als RFC 3339 / ISO 8601 — kein Unix Timestamp                     |
|#255 |Richtiges Format wählen: `date-time`, `date`, `time-local`, `date-time-local`|
|#235 |`_at`-Suffix für alle Zeitstempel-Properties                                 |
|#127 |Zeitdauern als ISO 8601 Duration (`PT30M`, `P1Y`) — kein Sekunden-Integer    |
|#170 |ISO-Normen für Land (3166), Sprache (639-1 / BCP 47), Währung (4217)         |
|#240 |Enum-Werte in UPPER_SNAKE_CASE                                               |
|#112 |Offene Enum-Listen mit `x-extensible-enum` — neue Werte ohne Breaking Change |
|#123 |Gleiche Semantik für `null` und fehlendes Feld — nicht kombinieren           |
|#122 |Kein `null` für Boolean — Feld weglassen oder Enum verwenden                 |
|#124 |Leere Arrays als `[]` — nicht als `null`                                     |
|#120 |Array-Namen im Plural                                                        |
|#144 |UUIDs nur wenn dezentrale ID-Generierung nötig ist                           |
|#216 |Maps mit `additionalProperties` in OpenAPI definieren                        |
|#252 |Einheitliches Schema für Lesen und Schreiben — `readOnly`/`writeOnly`        |
|C-07 |`metadata`-Feld für erweiterbare Zusatzdaten                                 |
|C-11 |`description` und `metadata` klar trennen                                    |

---

# Idempotenz, Sicherheit und Caching von HTTP-Methoden — Leitfaden für Entwickler

> Basierend auf den Regeln #148, #149, #229, #230, #231, #182, #253, #227, #155, #156, #157, #158 des REST API Styleguides.

-----

## Warum Idempotenz und Sicherheit wichtig sind

In verteilten Systemen sind Netzwerkfehler keine Ausnahme — sie sind ein normaler Betriebszustand. Ein Request wird abgeschickt, die Verbindung bricht ab, und der aufrufende Dienst weiss nicht ob der Request den Server erreicht hat oder nicht. Was passiert bei einer Wiederholung?

Ist die Operation idempotent, kann der Request bedenkenlos wiederholt werden — das Ergebnis ist dasselbe wie beim ersten Aufruf. Ist sie es nicht, entsteht ein Duplikat: eine zweite Bestellung, eine doppelte Zahlung, ein mehrfach versendetes E-Mail.

Die Eigenschaften Safe und Idempotent sind keine optionalen Qualitätsmerkmale. Sie sind Teil des HTTP-Standards und werden von Infrastruktur-Komponenten wie Load Balancern, Proxies, Gateways und Client-Bibliotheken genutzt um Requests korrekt zu behandeln. Eine Verletzung dieser Eigenschaften führt zu unerwartetem Verhalten das schwer zu debuggen ist.

-----

## Safe — Methoden die keinen Zustand ändern (#149)

Eine HTTP-Methode gilt als **sicher (safe)** wenn ihre Ausführung den Serverzustand nicht verändert. Sichere Methoden sind reine Lesezugriffe — unabhängig davon wie oft sie aufgerufen werden, bleibt der Zustand des Systems unverändert.

Nach Regel **#149** sind `GET` und `HEAD` safe.

```
GET /v1/orders/ord_abc123      ← Sicher: liest nur, ändert nichts
HEAD /v1/orders/ord_abc123     ← Sicher: wie GET, gibt nur Header zurück
```

Konsequenzen der Safe-Eigenschaft für die Implementierung:

**Caching ist erlaubt.** Proxies, Gateways und Browser-Caches dürfen GET-Responses cachen und zwischenspeichern. Eine Implementierung die bei GET tatsächlich Daten verändert, würde gecachte Responses liefern und die Änderung nicht ausführen.

**Automatische Wiederholung ist erlaubt.** HTTP-Clients und Load Balancer dürfen fehlgeschlagene GET-Requests automatisch wiederholen, ohne Rückfrage. Wenn GET Daten verändert, führt das zu unbeabsichtigten Mehrfachausführungen.

**Parallele Ausführung ist unbedenklich.** Mehrere gleichzeitige GET-Requests auf dieselbe Ressource verursachen keine Race Conditions.

Ein GET-Endpunkt der intern einen Zähler inkrementiert, eine E-Mail versendet oder einen Datenbankwert ändert, verletzt die Safe-Eigenschaft — auch wenn es im Einzelfall praktisch erscheint.

-----

## Idempotenz — Mehrfachausführung ohne Nebeneffekte (#149)

Eine HTTP-Methode gilt als **idempotent** wenn mehrfache identische Ausführungen dasselbe Ergebnis liefern wie eine einzige Ausführung. Der Zustand des Systems ist nach dem zehnten Aufruf identisch mit dem Zustand nach dem ersten.

Nach Regel **#149** sind `GET`, `HEAD`, `PUT` und `DELETE` idempotent.

```
GET  /v1/orders/ord_abc123    ← Idempotent: beliebig oft wiederholbar
PUT  /v1/orders/ord_abc123    ← Idempotent: setzt Ressource auf definierten Zustand
DELETE /v1/orders/ord_abc123  ← Idempotent: nach dem ersten Aufruf ist die Ressource weg,
                                  weitere Aufrufe ändern daran nichts (→ 404 oder 200)
```

Wichtig: Idempotenz beschreibt den **Zustand des Systems**, nicht die **HTTP-Response**. Ein zweiter DELETE-Aufruf auf eine bereits gelöschte Ressource kann `404 Not Found` zurückgeben — das ist korrekt, weil der Systemzustand (Ressource gelöscht) nach dem ersten wie nach dem zehnten Aufruf identisch ist.

### PUT — vollständiges Ersetzen

PUT ersetzt eine Ressource vollständig durch den übermittelten Zustand. Das macht PUT inhärent idempotent: egal wie oft derselbe PUT-Request ausgeführt wird, die Ressource hat danach immer denselben Zustand.

```http
PUT /v1/orders/ord_abc123
Content-Type: application/json

{
  "status": "CANCELLED",
  "cancellation_reason": "Customer request",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

```http
HTTP/1.1 200 OK

{
  "id": "ord_abc123",
  "status": "CANCELLED",
  "cancellation_reason": "Customer request",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

Wird derselbe Request ein zweites Mal gesendet, ist das Ergebnis identisch. PUT darf nicht für partielle Updates verwendet werden — das ist die Domäne von PATCH.

### DELETE — idempotente Löschung

DELETE ist idempotent: nach dem ersten erfolgreichen Aufruf ist die Ressource nicht mehr vorhanden. Weitere Aufrufe ändern daran nichts. Wie mit dem `404`-Statuscode bei wiederholten DELETE-Aufrufen umgegangen wird, ist eine Implementierungsentscheidung:

```http
DELETE /v1/orders/ord_abc123
→ 204 No Content       ← Erster Aufruf: Ressource gelöscht

DELETE /v1/orders/ord_abc123
→ 404 Not Found        ← Zweiter Aufruf: Ressource existiert nicht mehr
→ 204 No Content       ← Alternativ: ebenfalls akzeptabel (idempotentes Verhalten)
```

Beide Varianten sind korrekt. `404` ist semantisch präziser, `204` vereinfacht die Fehlerbehandlung auf Konsumentenseite da keine Unterscheidung zwischen erstem und weiteren Aufrufen nötig ist.

-----

## POST und PATCH — nicht idempotent (#148)

`POST` und `PATCH` sind nach HTTP-Standard nicht idempotent. Das hat direkte Konsequenzen:

**POST** erstellt eine neue Ressource. Jeder Aufruf erzeugt eine neue Ressource mit einer neuen ID. Zwei identische POST-Requests erzeugen zwei Bestellungen.

**PATCH** ändert eine Ressource partiell. Abhängig von der Implementierung kann PATCH nicht idempotent sein — zum Beispiel wenn ein Feld inkrementiert wird:

```json
PATCH /v1/accounts/acc_123
{ "balance_delta": 100 }     ← Nicht idempotent: jeder Aufruf addiert 100
```

Die fehlende Idempotenz bedeutet: bei Netzwerkfehlern und Timeouts kann nicht einfach wiederholt werden. Der aufrufende Dienst muss selbst entscheiden ob der ursprüngliche Request den Server erreicht hat oder nicht.

-----

## Idempotentes POST und PATCH gestalten (#229)

Da Netzwerkfehler unvermeidbar sind, empfiehlt Regel **#229**: POST und PATCH SOLLTEN idempotent gestaltet werden. Dafür stehen zwei Mechanismen zur Verfügung.

### Mechanismus 1 — Sekundärschlüssel (#231)

Der aufrufende Dienst übermittelt einen fachlichen Schlüssel der die Anfrage eindeutig identifiziert. Der Server prüft ob ein Eintrag mit diesem Schlüssel bereits existiert — wenn ja, wird die bestehende Ressource zurückgegeben statt eine neue zu erstellen:

```http
POST /v1/orders
Content-Type: application/json

{
  "external_order_id": "ERP-2024-00847",
  "items": [
    { "product_id": "prod_abc", "quantity": 2 }
  ],
  "delivery_date": "2024-02-15"
}
```

```http
HTTP/1.1 201 Created
Location: /v1/orders/ord_abc123

{
  "id": "ord_abc123",
  "external_order_id": "ERP-2024-00847",
  "status": "OPEN"
}
```

Wird derselbe Request mit demselben `external_order_id` wiederholt — sei es durch einen Retry nach Timeout oder einen Programmfehler — gibt der Server die bereits existierende Bestellung zurück:

```http
POST /v1/orders
{ "external_order_id": "ERP-2024-00847", ... }

HTTP/1.1 200 OK              ← 200 statt 201: Ressource existierte bereits
Location: /v1/orders/ord_abc123

{
  "id": "ord_abc123",
  "external_order_id": "ERP-2024-00847",
  "status": "OPEN"
}
```

Der Sekundärschlüssel ist ein fachlicher Wert aus der Domäne des aufrufenden Dienstes — eine Bestellnummer aus dem ERP-System, eine Referenz aus einem externen Prozess. Er wird in der OpenAPI-Spezifikation als optionales Feld definiert, das serverseitig auf Eindeutigkeit geprüft wird.

```yaml
components:
  schemas:
    OrderRequest:
      type: object
      required: [items]
      properties:
        external_order_id:
          type: string
          maxLength: 100
          description: |
            Optional client-provided identifier for idempotency.
            If an order with this external_order_id already exists,
            the existing order is returned instead of creating a new one.
          example: "ERP-2024-00847"
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
```

### Mechanismus 2 — Idempotency-Key Header (#230)

Alternativ zum Sekundärschlüssel im Request-Body kann der `Idempotency-Key` Header verwendet werden. Der aufrufende Dienst generiert eine UUID und sendet sie als Header mit:

```http
POST /v1/orders
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b
Content-Type: application/json

{
  "items": [
    { "product_id": "prod_abc", "quantity": 2 }
  ]
}
```

Der Server speichert den `Idempotency-Key` zusammen mit dem Request-Ergebnis. Kommt derselbe Key erneut, wird das gespeicherte Ergebnis zurückgegeben ohne den Request erneut auszuführen:

```http
POST /v1/orders
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b

HTTP/1.1 200 OK
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b

{
  "id": "ord_abc123",
  "status": "OPEN"
}
```

Der Unterschied zum Sekundärschlüssel: der `Idempotency-Key` ist ein technischer, nicht fachlicher Schlüssel. Er wird vom aufrufenden Dienst generiert — typischerweise eine UUID v4 — und hat keine Bedeutung ausserhalb der Idempotenz-Semantik. Er eignet sich besonders wenn kein natürlicher fachlicher Schlüssel vorhanden ist.

**Gültigkeitsdauer:** Idempotency-Keys werden typischerweise 24 Stunden gespeichert. Danach ist keine Garantie mehr gegeben dass ein wiederholter Request dasselbe Ergebnis liefert.

**Verhalten bei unterschiedlichem Body:** Wird derselbe `Idempotency-Key` mit einem anderen Request-Body gesendet, gibt der Server `422 Unprocessable Entity` zurück:

```http
POST /v1/orders
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b

{
  "items": [{ "product_id": "prod_xyz", "quantity": 5 }]   ← Anderer Body
}

HTTP/1.1 422 Unprocessable Entity
{
  "type": "https://api.example.com/errors/idempotency-conflict",
  "title": "Idempotency Key Conflict",
  "status": 422,
  "detail": "The Idempotency-Key was already used with a different request body."
}
```

### Sekundärschlüssel vs. Idempotency-Key

|               |Sekundärschlüssel (#231)                  |Idempotency-Key (#230)             |
|---------------|------------------------------------------|-----------------------------------|
|Herkunft       |Fachlicher Wert des aufrufenden Dienstes  |Technisch generierte UUID          |
|Sichtbarkeit   |Im Request-Body, Teil der Ressource       |HTTP-Header, nicht im Body         |
|Dauerhaftigkeit|Permanent — Ressource behält den Schlüssel|Temporär — 24 Stunden              |
|Geeignet wenn  |Natürlicher fachlicher Schlüssel vorhanden|Kein fachlicher Schlüssel vorhanden|
|OpenAPI        |Als Property im Schema definiert          |Als Header-Parameter definiert     |

-----

## Optimistisches Locking mit ETag (#182)

Idempotenz allein löst nicht alle Probleme bei gleichzeitigen Zugriffen. Wenn zwei Prozesse dieselbe Ressource gleichzeitig ändern, kann die Änderung des ersten Prozesses durch den zweiten überschrieben werden — ohne dass einer der beiden davon weiss. Das ist das Lost-Update-Problem.

ETag mit If-Match löst dieses Problem. Der Server sendet mit jeder GET-Response einen `ETag`-Header — einen Hashwert der den aktuellen Zustand der Ressource repräsentiert:

```http
GET /v1/orders/ord_abc123

HTTP/1.1 200 OK
ETag: "a1b2c3d4e5f6"

{
  "id": "ord_abc123",
  "status": "OPEN",
  "etag": "a1b2c3d4e5f6"
}
```

Der ETag-Wert wird im Response-Body als `etag`-Feld (nach #174) und im `ETag`-Header gleichzeitig geliefert. Beim nächsten schreibenden Zugriff sendet der aufrufende Dienst den ETag im `If-Match`-Header zurück:

```http
PUT /v1/orders/ord_abc123
If-Match: "a1b2c3d4e5f6"
Content-Type: application/json

{
  "status": "CANCELLED",
  "cancellation_reason": "Customer request"
}
```

Der Server prüft ob der aktuelle ETag der Ressource mit dem übermittelten übereinstimmt. Stimmen sie überein, hat sich die Ressource seit dem letzten Lesen nicht verändert — die Änderung wird durchgeführt:

```http
HTTP/1.1 200 OK
ETag: "x7y8z9a0b1c2"

{
  "id": "ord_abc123",
  "status": "CANCELLED",
  "etag": "x7y8z9a0b1c2"
}
```

Hat ein anderer Prozess die Ressource zwischenzeitlich geändert, stimmt der ETag nicht mehr überein. Der Server antwortet mit `409 Conflict`:

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/optimistic-locking-conflict",
  "title": "Optimistic Locking Conflict",
  "status": 409,
  "detail": "The resource was modified since it was last read. Please fetch the current version and retry.",
  "instance": "/v1/orders/ord_abc123"
}
```

Der aufrufende Dienst muss in diesem Fall die Ressource erneut lesen, die Änderung auf die aktuelle Version anwenden und den Request mit dem neuen ETag wiederholen.

### If-None-Match für bedingte Reads

`If-None-Match` ist die Read-Variante des bedingten Zugriffs. Wenn der aufrufende Dienst bereits eine Version der Ressource kennt und nur eine aktualisierte Version benötigt, sendet er den bekannten ETag mit:

```http
GET /v1/orders/ord_abc123
If-None-Match: "a1b2c3d4e5f6"
```

Hat sich die Ressource nicht verändert, antwortet der Server mit `304 Not Modified` ohne Body — das spart Bandbreite:

```http
HTTP/1.1 304 Not Modified
ETag: "a1b2c3d4e5f6"
```

Hat sich die Ressource verändert, wird die vollständige aktuelle Version zurückgegeben:

```http
HTTP/1.1 200 OK
ETag: "x7y8z9a0b1c2"

{ "id": "ord_abc123", "status": "CANCELLED", ... }
```

-----

## Asynchrone Operationen und Idempotenz (#253)

Langläufige Operationen — Exports, Massenupdates, Berechnungen — werden asynchron verarbeitet. Das Muster:

```http
POST /v1/exports
Idempotency-Key: 9a8b7c6d-5e4f-3a2b-1c0d-e9f8a7b6c5d4
Content-Type: application/json

{
  "filter": { "status": "COMPLETED", "created_at": { "gte": "2024-01-01" } },
  "format": "CSV"
}

HTTP/1.1 202 Accepted
Location: /v1/exports/exp_xyz789

{
  "id": "exp_xyz789",
  "status": "PENDING"
}
```

Der `Idempotency-Key` ist auch bei asynchronen Operationen wichtig: Wird der POST wiederholt bevor das Ergebnis abgerufen wurde, wird kein zweiter Export-Job gestartet — der bestehende Job wird zurückgegeben.

Der Status des Jobs wird über den zurückgegebenen `Location`-Header abgerufen:

```http
GET /v1/exports/exp_xyz789

HTTP/1.1 200 OK

{
  "id": "exp_xyz789",
  "status": "COMPLETED",
  "download_url": "https://storage.example.com/exports/exp_xyz789.csv",
  "expires_at": "2024-01-16T10:30:00Z"
}
```

Der GET-Request auf den Job-Status ist safe und idempotent — er kann beliebig oft wiederholt werden ohne Nebeneffekte.

-----

## OpenAPI-Spezifikation

Idempotenz-Mechanismen müssen in der OpenAPI-Spezifikation dokumentiert sein. Der `Idempotency-Key` Header wird als optionaler Header-Parameter definiert:

```yaml
paths:
  /v1/orders:
    post:
      summary: Create an order
      parameters:
        - name: Idempotency-Key
          in: header
          required: false
          schema:
            type: string
            format: uuid
          description: |
            Optional UUID for idempotent request handling.
            If provided, repeated requests with the same key and body
            return the cached response instead of creating a new resource.
            Keys are stored for 24 hours.
          example: "7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderRequest'
      responses:
        '201':
          description: Order created
          headers:
            Location:
              schema:
                type: string
              description: URL of the created order
        '200':
          description: Existing order returned (idempotent repeat)
          headers:
            Idempotency-Key:
              schema:
                type: string
              description: Echo of the provided Idempotency-Key
        '409':
          description: Conflict — resource modified since last read (ETag mismatch)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
        '422':
          description: Idempotency-Key reused with different request body
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
```

-----

## Übersicht — Eigenschaften aller HTTP-Methoden

|Methode |Safe|Idempotent|Typischer Statuscode|Verwendung                    |
|--------|----|----------|--------------------|------------------------------|
|`GET`   |✓   |✓         |`200`               |Ressource lesen               |
|`HEAD`  |✓   |✓         |`200`               |Nur Header lesen              |
|`PUT`   |✗   |✓         |`200`, `204`        |Ressource vollständig ersetzen|
|`DELETE`|✗   |✓         |`204`, `404`        |Ressource löschen             |
|`POST`  |✗   |✗         |`201`, `202`        |Ressource erstellen           |
|`PATCH` |✗   |✗         |`200`               |Ressource partiell ändern     |

-----

## Caching — Safe-Methoden effizient nutzen (#227, #155, #156, #157, #158)

Caching ist die direkte Folge der Safe-Eigenschaft: Weil GET und HEAD den Serverzustand nicht verändern, darf ihre Response zwischen gespeichert und wiederverwendet werden. Das reduziert Latenz, entlastet den Server und verbessert die Skalierbarkeit — ohne dass der aufrufende Dienst etwas davon mitbekommt.

Nach Regel **#227** muss jeder Endpunkt dessen Responses gecacht werden können, explizit mit `Cache-Control`-Direktiven dokumentiert sein. Das gilt umgekehrt genauso: Endpunkte die nicht gecacht werden dürfen, müssen das ebenfalls explizit signalisieren.

### Cache-Control Direktiven

Der `Cache-Control`-Header steuert das Caching-Verhalten auf allen Ebenen — im aufrufenden Dienst, in Proxies, im API-Gateway (Gravitee) und in CDNs. Die wichtigsten Direktiven:

```http
Cache-Control: max-age=3600
```

Die Response darf 3600 Sekunden (1 Stunde) gecacht werden. Innerhalb dieser Zeit wird keine neue Anfrage an den Server gestellt.

```http
Cache-Control: max-age=3600, must-revalidate
```

Die Response darf 3600 Sekunden gecacht werden. Nach Ablauf muss der Cache beim Server validieren bevor die Response erneut ausgeliefert wird — auch wenn der Server nicht erreichbar ist.

```http
Cache-Control: no-cache
```

Die Response darf gecacht werden, aber muss vor jeder Auslieferung beim Server validiert werden. Das verhindert veraltete Daten ohne Caching komplett zu deaktivieren — sinnvoll in Kombination mit ETag.

```http
Cache-Control: no-store
```

Die Response darf nicht gespeichert werden. Für sensible Daten wie Authentifizierungs-Tokens oder persönliche Informationen.

```http
Cache-Control: private, max-age=300
```

Die Response ist benutzerspezifisch und darf nur vom aufrufenden Dienst selbst gecacht werden — nicht von gemeinsam genutzten Proxies oder Gateways.

### Welche Direktive für welchen Endpunkt?

Die Wahl richtet sich danach wie häufig sich die Daten ändern und wie kritisch veraltete Daten sind:

|Endpunkt-Typ                                |Direktive                     |Begründung                                        |
|--------------------------------------------|------------------------------|--------------------------------------------------|
|Statische Referenzdaten (Länder, Kategorien)|`max-age=86400`               |Ändern sich selten, 24h Cache sinnvoll            |
|Produktkatalog                              |`max-age=300, must-revalidate`|Ändern sich gelegentlich, 5min Cache akzeptabel   |
|Bestellstatus                               |`no-cache`                    |Muss aktuell sein, aber ETag-Validierung möglich  |
|Persönliche Daten                           |`private, max-age=60`         |Nur aufrufender Dienst darf cachen                |
|Zahlungsdaten                               |`no-store`                    |Darf nicht persistiert werden                     |
|POST /search                                |`no-store`                    |POST-Responses werden standardmässig nicht gecacht|

### Caching in OpenAPI dokumentieren (#227)

Caching-Verhalten wird in der OpenAPI-Spezifikation über Response-Header dokumentiert:

```yaml
paths:
  /v1/product-categories:
    get:
      summary: List product categories
      description: |
        Returns all product categories. Responses are cached for 24 hours.
        Use If-None-Match for conditional requests.
      responses:
        '200':
          description: List of categories
          headers:
            Cache-Control:
              schema:
                type: string
                example: "max-age=86400, must-revalidate"
            ETag:
              schema:
                type: string
                example: "\"a1b2c3d4\""
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CategoryList'
        '304':
          description: Not Modified — cached response is still valid
          headers:
            ETag:
              schema:
                type: string

  /v1/orders/{id}:
    get:
      summary: Get order by ID
      responses:
        '200':
          description: Order details
          headers:
            Cache-Control:
              schema:
                type: string
                example: "no-cache"
            ETag:
              schema:
                type: string
```

### ETag als Cache-Validierung (#182)

ETag und Caching arbeiten zusammen: `Cache-Control: no-cache` bedeutet nicht “nicht cachen” sondern “vor Auslieferung validieren”. Die Validierung erfolgt über den ETag:

```http
# Erster Request — Response wird gecacht
GET /v1/orders/ord_abc123

HTTP/1.1 200 OK
Cache-Control: no-cache
ETag: "a1b2c3d4e5f6"

{ "id": "ord_abc123", "status": "OPEN" }
```

```http
# Zweiter Request — Cache validiert mit ETag
GET /v1/orders/ord_abc123
If-None-Match: "a1b2c3d4e5f6"

HTTP/1.1 304 Not Modified
ETag: "a1b2c3d4e5f6"
# Kein Body — gecachte Response wird verwendet
```

```http
# Dritter Request — Ressource hat sich geändert
GET /v1/orders/ord_abc123
If-None-Match: "a1b2c3d4e5f6"

HTTP/1.1 200 OK
ETag: "x7y8z9a0b1c2"

{ "id": "ord_abc123", "status": "CANCELLED" }
# Neue Version wird zurückgegeben und gecacht
```

Das Ergebnis: Netzwerktraffic wird nur dann erzeugt wenn sich tatsächlich etwas geändert hat. Bei `304` wird kein Response-Body übertragen — nur der Header. Bei grossen Responses ist das ein erheblicher Bandbreitengewinn.

### Bandbreite zusätzlich reduzieren (#156, #157, #158)

Neben Caching gibt es drei weitere Mechanismen die den Datenverkehr reduzieren:

**gzip-Komprimierung (#156)** komprimiert Response-Bodies serverseitig. Der aufrufende Dienst signalisiert Unterstützung über den `Accept-Encoding`-Header:

```http
GET /v1/orders
Accept-Encoding: gzip

HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Type: application/json
```

JSON-Daten lassen sich durch gzip typischerweise auf 10–20% der ursprünglichen Grösse komprimieren. Bei Pagination-Responses mit vielen Einträgen ist das ein signifikanter Gewinn.

**Feldauswahl (#157)** erlaubt dem aufrufenden Dienst, nur benötigte Felder anzufordern. Das reduziert sowohl den übertragenen Datenumfang als auch den Serialisierungsaufwand serverseitig:

```http
GET /v1/orders?fields=id,status,created_at

HTTP/1.1 200 OK

{
  "items": [
    { "id": "ord_abc123", "status": "OPEN", "created_at": "2024-01-15T10:30:00Z" },
    { "id": "ord_def456", "status": "COMPLETED", "created_at": "2024-01-14T08:00:00Z" }
  ],
  "cursor": { "next": "eyJpZCI6...", "prev": null }
}
```

Ohne `?fields` würden alle Properties zurückgegeben — inkl. `delivery_address`, `line_items`, `payment_details` und weiterer Felder die für den jeweiligen Use Case nicht relevant sind.

**Einbetten von Sub-Ressourcen (#158)** reduziert die Anzahl der Requests indem verwandte Ressourcen in einer einzigen Response mitgeliefert werden:

```http
GET /v1/orders/ord_abc123?embed=items,customer

HTTP/1.1 200 OK

{
  "id": "ord_abc123",
  "status": "OPEN",
  "items": [
    { "id": "item_001", "product_id": "prod_xyz", "quantity": 2 }
  ],
  "customer": {
    "id": "cust_789",
    "name": "Max Muster"
  }
}
```

Ohne `embed` wären drei separate Requests nötig: `GET /v1/orders/ord_abc123`, `GET /v1/orders/ord_abc123/items` und `GET /v1/customers/cust_789`. Das Einbetten ist optional — der aufrufende Dienst entscheidet je nach Bedarf.

### Caching bei POST /search

POST-Requests werden von HTTP-Infrastruktur standardmässig nicht gecacht. Das ist ein Nachteil von `POST /v1/orders/search` gegenüber `GET /v1/orders` mit Query-Parametern. GET-Responses können gecacht werden, POST-Responses nicht.

Wenn Caching für Suchergebnisse relevant ist, empfiehlt sich `GET` mit Query-Parametern für den einfachen Fall — und `POST /search` nur wenn die Komplexität des Filters es wirklich erfordert. Das ist die in Regel #237 beschriebene Entscheidung: einfache Filter per GET, komplexe Filter per POST — mit dem bewussten Verzicht auf Caching beim POST-Endpunkt.

-----

**GET-Endpunkte mit Seiteneffekten implementieren.** Ein GET-Request sendet eine Benachrichtigung, inkrementiert einen Zähler oder ändert einen Datenbankwert. Das verletzt die Safe-Eigenschaft. Proxies cachen die Response, Monitoring-Systeme rufen den Endpunkt regelmässig ab — die Seiteneffekte treten unkontrolliert auf.

**PUT für partielle Updates verwenden.** PUT ersetzt die gesamte Ressource. Wird nur ein Teilbereich im Body übergeben, werden alle nicht übermittelten Felder gelöscht oder auf Default-Werte zurückgesetzt. Für partielle Updates ist PATCH zu verwenden.

**DELETE bei wiederholtem Aufruf mit `500` antworten.** Ein zweiter DELETE-Aufruf auf eine bereits gelöschte Ressource löst serverseitig eine Exception aus die als `500` zurückgegeben wird. Korrekt ist `404` oder `204` — nicht `500`. Idempotenz bedeutet dass der Fehlerfall “Ressource existiert nicht mehr” kein Serverfehler ist.

**POST ohne Idempotenz bei kritischen Operationen.** Eine Zahlung, eine Bestellung oder eine E-Mail-Versendung wird per POST ausgelöst ohne Sekundärschlüssel oder Idempotency-Key. Bei einem Netzwerkfehler und automatischem Retry entstehen Duplikate. Bei allen Operationen mit fachlichen Konsequenzen ist Idempotenz Pflicht.

**ETag-Werte selbst konstruieren.** Der `If-Match`-Wert wird nicht aus der vorherigen GET-Response übernommen, sondern aus dem eigenen Zustand berechnet. ETags sind serverinterne Werte — sie dürfen vom aufrufenden Dienst nicht konstruiert oder interpretiert werden, analog zu Pagination-Cursors.

**Idempotency-Key zwischen verschiedenen Endpunkten wiederverwenden.** Ein `Idempotency-Key` ist endpunktspezifisch. Derselbe Key für `POST /v1/orders` und `POST /v1/payments` zu verwenden führt zu unerwarteten Ergebnissen. Für jeden Request wird ein neuer Key generiert.

**`Cache-Control`-Header nicht setzen.** Ohne `Cache-Control` entscheidet der Cache selbst ob und wie lange gecacht wird — das Verhalten ist undefiniert und unterscheidet sich zwischen Proxies, Gateways und Clients. Jeder cacheable Endpunkt muss explizit mit `Cache-Control` dokumentiert sein (Regel #227).

**POST-Responses mit `Cache-Control` versehen und Caching erwarten.** POST ist nicht safe und wird von HTTP-Infrastruktur standardmässig nicht gecacht — unabhängig vom gesetzten `Cache-Control`-Header. Wer Caching benötigt, verwendet GET.

**`no-cache` mit “nicht cachen” gleichsetzen.** `Cache-Control: no-cache` bedeutet “vor jeder Auslieferung beim Server validieren” — nicht “niemals cachen”. Die Response wird gecacht, aber vor Verwendung mit ETag validiert. Wer wirklich verhindern will dass etwas gespeichert wird, verwendet `no-store`.

**ETag für Caching ignorieren.** ETag und `Cache-Control: no-cache` sind das effizienteste Caching-Muster: die Response wird lokal gecacht, aber nur ausgeliefert wenn der Server per `304 Not Modified` bestätigt dass sie noch aktuell ist. Ohne ETag muss bei `no-cache` der vollständige Response-Body bei jedem Request übertragen werden.

-----

## Zusammenfassung

|Regel|Kernaussage                                                                                                   |
|-----|--------------------------------------------------------------------------------------------------------------|
|#148 |HTTP-Methoden semantisch korrekt verwenden — GET liest, PUT ersetzt, PATCH ändert partiell                    |
|#149 |Safe: GET, HEAD ändern keinen Zustand. Idempotent: GET, PUT, DELETE liefern bei Wiederholung dasselbe Ergebnis|
|#229 |POST und PATCH SOLLTEN idempotent gestaltet werden — via Sekundärschlüssel oder Idempotency-Key               |
|#231 |Sekundärschlüssel im Request-Body für idempotentes POST — fachlicher Schlüssel des aufrufenden Dienstes       |
|#230 |`Idempotency-Key` Header als technische Alternative — UUID, 24 Stunden gültig                                 |
|#182 |ETag mit If-Match für optimistisches Locking — verhindert Lost-Update bei gleichzeitigen Zugriffen            |
|#253 |Asynchrone Operationen mit `202 Accepted` + `Location` — Idempotency-Key verhindert doppelte Jobs             |
|#227 |Cacheable Endpunkte mit `Cache-Control`-Direktiven in OpenAPI dokumentieren — MUSS                            |
|#155 |Bandbreite reduzieren — Kombination aus Caching, Komprimierung, Feldauswahl und Einbetten                     |
|#156 |gzip-Komprimierung via `Accept-Encoding` / `Content-Encoding` — typisch 80–90% Reduktion                      |
|#157 |Feldauswahl via `?fields=id,status` — nur benötigte Felder übertragen                                         |
|#158 |Sub-Ressourcen einbetten via `?embed=items` — mehrere Requests in einem zusammenfassen                        |

---
# Versionierung und Deprecation — Leitfaden für Entwickler

> Basierend auf den Regeln C-01, #116, #106, C-10, #107, #108, #109, #110, #111, #112, C-09, C-12, #185, #186, #187, #188, #189, #190, #191 des REST API Styleguides.

-----

## Zwei Ebenen der Versionierung

API-Versionierung findet auf zwei Ebenen statt, die unabhängig voneinander verwaltet werden und unterschiedliche Zwecke erfüllen:

**URL-Version** — die Major-Version im Pfad (`/v1/`, `/v2/`). Sie wird nur bei inkompatiblen Breaking Changes erhöht und bleibt über lange Zeiträume stabil. Eine neue URL-Version ist eine weitreichende Entscheidung mit Folgen für alle Konsumenten.

**Spec-Version** — die Versionsnummer in der OpenAPI-Spezifikation (`1.3.2`). Sie folgt Semantic Versioning und wird bei jeder Änderung der API-Beschreibung aktualisiert — auch bei kleinen Korrekturen oder neuen optionalen Feldern.

Diese Trennung ist bewusst: Die Spec-Version dokumentiert den Entwicklungsstand der API-Beschreibung. Die URL-Version signalisiert Konsumenten ob eine Migration erforderlich ist.

-----

## URL-Versionierung (C-01)

Nach Regel **C-01** enthält jeder API-Pfad die Major-Version als erstes Pfadsegment:

```
/v1/orders
/v1/order-items/{id}
/v1/customers/{id}/addresses
```

Dabei gelten drei feste Regeln: Nur Major Versions werden im Pfad geführt (`v1`, `v2`, `v3`). Media Type Versioning (`Accept: application/vnd.api+json;version=2`) wird nicht verwendet. Header-Versionierung wird nicht verwendet.

Eine neue Major Version wird ausschliesslich bei echten Breaking Changes eingeführt — nicht bei jeder grösseren Erweiterung. Solange Konsumenten keine Anpassungen an ihrem Code vornehmen müssen, bleibt die URL-Version unverändert.

-----

## Semantic Versioning der Spec (#116)

Die Versionsnummer in der OpenAPI-Spezifikation folgt dem Schema `MAJOR.MINOR.PATCH`:

|Änderung                           |Aktion       |Beispiel         |
|-----------------------------------|-------------|-----------------|
|Breaking Change — inkompatibel     |MAJOR erhöhen|`1.3.2` → `2.0.0`|
|Neue Funktion — rückwärtskompatibel|MINOR erhöhen|`1.3.2` → `1.4.0`|
|Korrektur oder Dokumentation       |PATCH erhöhen|`1.3.2` → `1.3.3`|

```yaml
info:
  title: Order Management API
  version: 1.4.2    # Spec-Version — unabhängig von der URL-Version
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70
```

Die `x-api-id` ist eine unveränderliche UUID die die API über alle Versionen hinweg eindeutig identifiziert. Sie ändert sich auch bei Breaking Changes oder Umbenennung der API nicht.

-----

## Was ist ein Breaking Change? (#106, C-10)

Breaking Changes erfordern eine neue Major Version in URL und Spec. Die vier Erweiterungsregeln nach **C-10** definieren was als Breaking Change gilt:

**1. Nichts wegnehmen** — keine Properties, Endpunkte oder Enum-Werte dürfen entfernt werden.

**2. Processing Rules nicht ändern** — die Semantik bestehender Felder bleibt stabil. Ein Feld das bisher den Nettobetrag enthielt darf nicht plötzlich den Bruttobetrag enthalten, auch wenn der Feldname gleich bleibt.

**3. Optionales nicht zu Pflicht machen** — ein bisher optionales Feld darf nicht zu einem Pflichtfeld werden. Konsumenten die das Feld bisher weggelassen haben, würden sonst mit `422`-Fehlern konfrontiert.

**4. Alles Neue muss optional sein** — neue Felder, neue Endpunkte und neue Enum-Werte sind immer optional. Konsumenten die sie nicht kennen, müssen sie ignorieren können.

Die vollständige Übersicht:

|Änderung                                             |Breaking?                                          |
|-----------------------------------------------------|---------------------------------------------------|
|Pflichtfeld in Request hinzufügen                    |✓ Breaking                                         |
|Feld entfernen oder umbenennen                       |✓ Breaking                                         |
|Ressource umbenennen (`/orders` → `/purchase-orders`)|✓ Breaking                                         |
|Typ ändern (`string` → `integer`)                    |✓ Breaking                                         |
|Semantik eines Feldes ändern ohne Umbenennung        |✓ Breaking                                         |
|Endpunkt entfernen                                   |✓ Breaking                                         |
|Statuscode ändern                                    |✓ Breaking                                         |
|Optionales Feld zum Request hinzufügen               |✗ Kompatibel                                       |
|Optionales Feld zur Response hinzufügen              |✗ Kompatibel                                       |
|Neuen Endpunkt hinzufügen                            |✗ Kompatibel                                       |
|Enum-Wert hinzufügen                                 |✗ Kompatibel (wenn Konsument tolerant — siehe #108)|
|Fehlermeldung in `detail` anpassen                   |✗ Kompatibel                                       |

-----

## Rückwärtskompatible Erweiterungen (#107, #111)

Das Ziel ist es, neue Funktionalität einzuführen ohne bestehende Konsumenten zu beeinträchtigen. Dazu gibt es bewährte Muster:

**Optionale Felder hinzufügen:**

```json
// Vorher — v1 Response
{
  "id": "ord_abc123",
  "status": "OPEN",
  "total_amount": 149.95
}

// Nachher — v1 Response, rückwärtskompatibel erweitert
{
  "id": "ord_abc123",
  "status": "OPEN",
  "total_amount": 149.95,
  "tax_amount": 23.99,          // neu, optional
  "currency_code": "EUR"         // neu, optional
}
```

Konsumenten die das neue Feld nicht kennen, ignorieren es. Konsumenten die es nutzen wollen, können es ab sofort verwenden — ohne eine neue Major Version.

**Neue Endpunkte hinzufügen:**

```
// Bestehend — unverändert
GET /v1/orders
GET /v1/orders/{id}

// Neu hinzugefügt — kein Breaking Change
GET /v1/orders/{id}/timeline
POST /v1/orders/export
```

**Enum-Werte erweitern:**

Nach Regel **#112** werden Enum-Listen als offen deklariert — neue Werte können jederzeit hinzugefügt werden. Konsumenten müssen unbekannte Werte tolerieren statt mit Fehler ablehnen:

```yaml
status:
  type: string
  x-extensible-enum:
    - OPEN
    - IN_PROGRESS
    - COMPLETED
    - CANCELLED
  description: |
    New values may be added in future versions.
    Clients must handle unknown values gracefully.
```

-----

## Tolerant Reader Pattern (#108, C-09)

Rückwärtskompatible Erweiterungen funktionieren nur wenn Konsumenten auf der Gegenseite robust implementiert sind. Nach Regel **#108** und Postel’s Law (C-09) gilt:

**Unbekannte Properties ignorieren:**

```javascript
// ✗ Fehleranfällig — bricht bei neuen Feldern
const { id, status, total_amount } = response;
if (Object.keys(response).some(k => !['id','status','total_amount'].includes(k))) {
  throw new Error('Unexpected field in response');
}

// ✓ Robust — unbekannte Felder werden ignoriert
const { id, status, total_amount } = response;
// Weitere Felder werden einfach nicht ausgelesen
```

**Unbekannte Enum-Werte tolerieren:**

```javascript
// ✗ Fehleranfällig — bricht bei neuen Enum-Werten
switch (order.status) {
  case 'OPEN': ...; break;
  case 'COMPLETED': ...; break;
  default: throw new Error(`Unknown status: ${order.status}`);
}

// ✓ Robust — unbekannte Werte werden toleriert
switch (order.status) {
  case 'OPEN': ...; break;
  case 'COMPLETED': ...; break;
  default:
    // Unbekannten Status ignorieren oder als "sonstiger Zustand" behandeln
    break;
}
```

**Nicht verwendete Response-Felder nicht validieren:**

Ein Konsument der nur `id` und `status` benötigt, sollte nicht prüfen ob die Response ausschliesslich diese Felder enthält. Zusätzliche Felder sind normale Erweiterungen, keine Fehler.

-----

## API-Spec in Git verwalten (C-12)

Nach Regel **C-12** werden OpenAPI-Spezifikationen in Git verwaltet — mit denselben Konventionen wie Code:

```
repository/
├── openapi.yaml          # Aktuelle Spec
├── CHANGELOG.md          # Alle Änderungen dokumentiert
└── ...
```

Für jede veröffentlichte API-Version wird ein Git Tag gesetzt:

```bash
git tag api/v1.4.2
git push origin api/v1.4.2
```

Alle Änderungen an der Spec laufen über Pull Requests — kein direktes Commit auf `main`. Das ermöglicht Code-Reviews für API-Änderungen und stellt sicher, dass Breaking Changes bewusst entschieden werden.

**CHANGELOG.md** dokumentiert jede Änderung mit Versionsreferenz:

```markdown
# Changelog

## [1.4.2] — 2024-06-15
### Fixed
- Corrected description of `delivery_date` field (#169)

## [1.4.0] — 2024-05-01
### Added
- Optional field `tax_amount` in Order response
- Optional field `currency_code` in Order response
- New endpoint GET /v1/orders/{id}/timeline

## [2.0.0] — 2024-03-01
### Breaking Changes
- Renamed field `price` to `total_amount` in Order response
- Removed deprecated endpoint GET /v1/legacy-orders
- Changed type of `quantity` from string to integer

### Migration
- Replace all references to `price` with `total_amount`
- Migrate from /v1/legacy-orders to /v2/orders
```

-----

## Wann eine neue Major Version nötig ist

Eine neue URL-Version (`/v2/`) ist nur bei echten Breaking Changes einzuführen. Drei Szenarien verdeutlichen die Entscheidung:

**Szenario 1 — Feldumbenennung:** Das Feld `price` soll in `total_amount` umbenannt werden. Das ist ein Breaking Change — alle Konsumenten die `price` lesen, erhalten `null` oder einen Fehler. Eine neue Major Version ist erforderlich.

**Szenario 2 — Neues Pflichtfeld:** Ein neues Pflichtfeld `warehouse_id` soll zum POST-Request hinzugefügt werden. Das ist ein Breaking Change — alle Konsumenten die das Feld nicht mitschicken, erhalten `422`. Eine neue Major Version ist erforderlich.

**Szenario 3 — Neues optionales Feld:** Ein neues optionales Feld `estimated_delivery_at` soll zur Response hinzugefügt werden. Das ist kein Breaking Change — Konsumenten ignorieren das Feld einfach. Keine neue Major Version nötig, nur MINOR in der Spec erhöhen.

-----

## Übergang auf eine neue Major Version

Wenn eine neue Major Version eingeführt wird, laufen beide Versionen für eine Übergangszeit parallel:

```
/v1/orders    ← Deprecated — läuft noch bis Sunset-Datum
/v2/orders    ← Aktuelle Version
```

Diese Parallelphase ist keine technische Empfehlung sondern eine Pflicht gegenüber Konsumenten: Sie brauchen Zeit um zu migrieren. Der Deprecation-Prozess (siehe nächstes Kapitel) regelt wie lange die alte Version verfügbar bleibt und wie Konsumenten informiert werden.

-----

## Deprecation-Prozess (#185, #186, #187, #188, #189, #190, #191)

Deprecation ist kein einmaliger Akt sondern ein strukturierter Prozess mit sechs Schritten. Kein Konsument darf unvorbereitet von einer API-Abschaltung betroffen sein.

### Schritt 0 — Spec markieren (#187)

Als erstes wird der betroffene Endpunkt in der OpenAPI-Spezifikation als deprecated markiert:

```yaml
paths:
  /v1/orders:
    get:
      deprecated: true
      description: |
        **Deprecated** — Migrate to /v2/orders.
        This endpoint will be decommissioned on 2025-06-30.
        See migration guide: https://developer.example.com/migration/v2
```

Das Sunset-Datum wird in der Beschreibung genannt. Es wird gleichzeitig im CHANGELOG.md dokumentiert.

### Schritt 1 — Response-Header setzen (#189)

Ab sofort enthalten alle Responses des deprecated Endpunkts zwei HTTP-Header:

```http
Deprecation: true
Sunset: Mon, 30 Jun 2025 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

`Deprecation: true` signalisiert maschinell dass dieser Endpunkt abgekündigt ist. `Sunset` gibt das genaue Abschaltdatum im RFC 7231 Format an. `Link` verweist auf den Nachfolge-Endpunkt. Automatisierte Monitoring-Systeme der Konsumenten können diese Header auswerten und Warnungen ausgeben.

### Schritt 2 — Ankündigung

Alle bekannten Konsumenten werden aktiv informiert — nicht nur über die Header, sondern direkt:

- Interne Konsumenten: E-Mail, Ticket im Issue-Tracker, persönliche Benachrichtigung
- Externe Partner: E-Mail an den definierten technischen Ansprechpartner
- Öffentliche APIs: Blogpost, Developer Portal, Newsletter, Changelog

Die Ankündigung enthält das Sunset-Datum, den Migrationspfad und einen Link zur Migrationsdokumentation. Datum, Kanal und Empfänger der Ankündigung werden dokumentiert.

### Schritt 3 — Migrationsfrist einräumen

Die Mindestfrist richtet sich nach der deklarierten `x-audience` der API:

|Audience                |Mindestfrist|
|------------------------|------------|
|`component-internal`    |2 Wochen    |
|`business-unit-internal`|4 Wochen    |
|`company-internal`      |3 Monate    |
|`external-partner`      |6 Monate    |
|`external-public`       |12 Monate   |

Diese Fristen sind nicht verhandelbar. Sie spiegeln die unterschiedlichen Rahmenbedingungen der Konsumenten wider: interne Teams können schnell reagieren, externe Partner haben eigene Release-Zyklen und Verträge.

### Schritt 4 — Nutzung überwachen (#188, #190)

Während der Migrationsfrist wird die tatsächliche Nutzung des deprecated Endpunkts gemessen. Drei Alert-Schwellen werden konfiguriert:

```
90 Tage vor Sunset  → Info-Alert: Noch X aktive Konsumenten
30 Tage vor Sunset  → Warning-Alert: Migration noch nicht abgeschlossen
 7 Tage vor Sunset  → Critical-Alert: Abschaltung unmittelbar bevorstehend
```

Solange aktive Aufrufe vorhanden sind, ist bekannt wer noch nicht migriert hat.

### Schritt 5 — Eskalation

Nach 80% der Migrationsfrist werden Konsumenten die noch nicht migriert haben, aktiv angesprochen:

```
Frist zu 0%  → Ankündigung an alle bekannten Konsumenten
Frist zu 50% → Erinnerung an nicht-migrierte Konsumenten
Frist zu 80% → Eskalation an Team-Lead / Management
Frist zu 100%→ Abschaltung
```

Eskalation bedeutet keine Verlängerung der Frist. Sie gibt Konsumenten eine letzte Gelegenheit, die Migration zu priorisieren.

### Schritt 6 — Abschaltung

Nach Ablauf der Migrationsfrist wird der Endpunkt abgeschaltet — auch wenn einzelne Konsumenten noch nicht migriert haben. Voraussetzung ist dass die Nachweise vorliegen:

```
✓ Ankündigung dokumentiert (Datum, Kanal, Empfänger)
✓ Sunset-Datum in Spec (#187) und Response-Header (#189) gesetzt
✓ Monitoring zeigt: Nutzung geht gegen null (#188)
✓ Eskalation für aktive Konsumenten dokumentiert
```

Kein Konsument kann die Abschaltung dauerhaft blockieren — das wäre operativ nicht tragbar. Die Verantwortung liegt bei Konsumenten die trotz Ankündigung und Migrationsfrist nicht reagiert haben.

### Deprecated Endpunkte nicht neu verwenden (#191)

Sobald ein Endpunkt als deprecated markiert ist, darf er von neuen Services oder neuen Integrationen nicht mehr verwendet werden. Wer heute eine neue Integration auf einen deprecated Endpunkt aufbaut, muss morgen sofort wieder migrieren.

-----

## Vollständiges Beispiel: Migration von v1 auf v2

Das folgende Beispiel zeigt den vollständigen Lebenszyklus einer Breaking Change — vom Entscheid über die Deprecation bis zur Abschaltung.

**Ausgangslage:** `GET /v1/orders` gibt Bestellungen zurück. Das Feld `price` soll in `total_amount` umbenannt werden — ein Breaking Change.

**Schritt 1 — v2 einführen, v1 weiter betreiben:**

```yaml
# v2 Spec — neues Feldschema
/v2/orders:
  get:
    responses:
      '200':
        content:
          application/json:
            schema:
              properties:
                id:
                  type: string
                total_amount:    # Neu — war vorher "price"
                  type: number
                  format: decimal
```

**Schritt 2 — v1 deprecaten:**

```yaml
# v1 Spec — deprecated
/v1/orders:
  get:
    deprecated: true
    description: |
      Deprecated — Migrate to /v2/orders.
      The field 'price' has been renamed to 'total_amount' in v2.
      Sunset date: 2026-03-31.
```

**Schritt 3 — v1 Response-Header setzen:**

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Tue, 31 Mar 2026 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
Content-Type: application/json

{
  "items": [...],
  "cursor": { "next": "...", "prev": null }
}
```

**Schritt 4 — CHANGELOG.md aktualisieren:**

```markdown
## [2.0.0] — 2025-10-01
### Breaking Changes
- Renamed field `price` to `total_amount` in Order response

### Deprecated
- GET /v1/orders — sunset date: 2026-03-31
- Migrate to GET /v2/orders
```

**Schritt 5 — Ankündigung und Monitoring bis Sunset-Datum.**

**Schritt 6 — Am 31. März 2026 — v1 abschalten:**

```http
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/api-version-sunset",
  "title": "API Version Sunset",
  "status": 410,
  "detail": "This API version was sunset on 2026-03-31. Please migrate to /v2/orders.",
  "instance": "/v1/orders"
}
```

Nach der Abschaltung gibt der Endpunkt `410 Gone` zurück — nicht `404`. `410` signalisiert explizit dass die Ressource dauerhaft entfernt wurde und nicht wiederkommt. Konsumenten die `404` und `410` unterscheiden, erkennen sofort dass eine Migration erforderlich ist.

-----

## Häufige Fehler

**Breaking Change ohne neue Major Version einführen.** Ein Pflichtfeld wird in der bestehenden Version hinzugefügt. Bestehende Konsumenten scheitern ohne Vorwarnung mit `422`. Die Lösung ist immer eine neue Major Version mit Deprecation der alten.

**Sunset-Datum zu knapp ansetzen.** Zwei Wochen Migrationsfrist für einen externen Partner ist nicht realistisch. Die Mindestfristen aus #185 gelten nicht nur als Empfehlung sondern als Untergrenze.

**Monitoring erst nach der Ankündigung aktivieren.** Ohne Baseline-Monitoring vor der Deprecation ist nicht bekannt wie viele Konsumenten den Endpunkt tatsächlich nutzen. Monitoring sollte immer laufen — nicht erst ab Deprecation.

**Beide Versionen für immer parallel betreiben.** `/v1/` und `/v2/` laufen nach Jahren noch parallel weil der Abschaltungsprozess nie gestartet wurde. Der Deprecation-Prozess beginnt mit der Einführung von v2 — nicht irgendwann danach.

**Enum-Werte ohne Vorwarnung entfernen.** Ein Enum-Wert der in Responses vorkam wird entfernt. Konsumenten die ihren Code auf alle bekannten Werte ausgelegt haben, verhalten sich jetzt unerwartet. Enum-Werte in Responses werden nie entfernt — sie werden deprecated und bleiben bis zur nächsten Major Version erhalten.

**`x-api-id` bei neuer Version ändern.** Die `x-api-id` ist unveränderlich. Eine neue Major Version bekommt dieselbe `x-api-id` wie alle vorherigen Versionen. Nur so ist die Continuity der API über Versionsgrenzen hinweg nachverfolgbar.

-----

## Zusammenfassung

|Regel|Kernaussage                                                                                               |
|-----|----------------------------------------------------------------------------------------------------------|
|C-01 |Major-Version im URL-Pfad: `/v1/`, `/v2/` — nur bei Breaking Changes erhöhen                              |
|#116 |Spec-Version nach Semantic Versioning: MAJOR.MINOR.PATCH                                                  |
|#106 |Keine Breaking Changes ohne neue Major Version                                                            |
|C-10 |Vier Erweiterungsregeln: nichts wegnehmen, Semantik stabil, optional bleibt optional, Neues immer optional|
|#108 |Tolerant Reader Pattern: unbekannte Felder und Enum-Werte ignorieren                                      |
|#112 |Offene Enum-Listen — neue Werte können jederzeit hinzugefügt werden                                       |
|C-12 |API-Spec in Git mit Tags, CHANGELOG und Pull Requests                                                     |
|#187 |Deprecated Endpunkte in der Spec markieren mit Sunset-Datum                                               |
|#189 |`Deprecation` und `Sunset` Response-Header setzen                                                         |
|#185 |Konsumenten informieren, Mindestfristen einhalten, strukturierter Prozess                                 |
|#186 |Externe Partner: Mindestfrist 6–12 Monate, offizielle Kanäle                                              |
|#188 |Nutzung deprecated Endpunkte aktiv monitoren                                                              |
|#190 |Alerts bei 90, 30 und 7 Tagen vor Sunset                                                                  |
|#191 |Deprecated Endpunkte nicht neu verwenden                                                                  |

---
# REST API Styleguide — Erklärungen auf Deutsch

> Version 2.0 — Basierend auf [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/), [Adidas API Guidelines](https://adidas.gitbook.io/api-guidelines/) und [Stripe API](https://docs.stripe.com/api).  
> Zalando-interne Regeln wurden entfernt. Quercheck mit Adidas und Stripe eingearbeitet.  
> Alle Regeln sind auf Deutsch erklärt. Eigene Regeln sind explizit gekennzeichnet.

## Inhaltsverzeichnis

- [1. Allgemeine Richtlinien](#1-allgemeine-richtlinien)
  - [100 MUSS API-First-Prinzip befolgen](#100-muss-api-first-prinzip-befolgen)
  - [101 MUSS API-Spezifikation mit OpenAPI bereitstellen](#101-muss-api-spezifikation-mit-openapi-bereitstellen)
  - [102 SOLLTE API-Benutzerhandbuch bereitstellen](#102-sollte-api-benutzerhandbuch-bereitstellen)
  - [103 MUSS APIs auf amerikanischem Englisch schreiben](#103-muss-apis-auf-amerikanischem-englisch-schreiben)
  - [C-08 MUSS Minimale API-Oberfläche (YAGNI-Prinzip)](#c-08-muss-minimale-api-oberfläche-yagni-prinzip)
  - [C-09 MUSS Robustheit nach Postel’s Law](#c-09-muss-robustheit-nach-postel-s-law)
  - [C-12 MUSS API-Spezifikationen in Git versionieren](#c-12-muss-api-spezifikationen-in-git-versionieren)
- [2. Meta-Informationen](#2-meta-informationen)
  - [218 MUSS API Meta-Informationen enthalten](#218-muss-api-meta-informationen-enthalten)
  - [116 MUSS Semantic Versioning verwenden](#116-muss-semantic-versioning-verwenden)
  - [215 MUSS API-Identifier bereitstellen](#215-muss-api-identifier-bereitstellen)
  - [219 MUSS API-Zielgruppe angeben](#219-muss-api-zielgruppe-angeben)
- [3. Sicherheit](#3-sicherheit)
  - [104 MUSS Alle Endpunkte absichern](#104-muss-alle-endpunkte-absichern)
  - [105 MUSS Berechtigungen (Scopes) definieren und zuweisen](#105-muss-berechtigungen-scopes-definieren-und-zuweisen)
  - [C-06 MUSS Einheitliche Scope-Namenskonvention](#c-06-muss-einheitliche-scope-namenskonvention)
- [4. Datenformate](#4-datenformate)
  - [238 MUSS Standarddatenformate verwenden](#238-muss-standarddatenformate-verwenden)
  - [171 MUSS Format für Zahlen und Integer definieren](#171-muss-format-für-zahlen-und-integer-definieren)
  - [169 MUSS Standardformate für Datum/Zeit verwenden](#169-muss-standardformate-für-datum-zeit-verwenden)
  - [255 SOLLTE Geeignete Datum/Zeit-Formate wählen](#255-sollte-geeignete-datum-zeit-formate-wählen)
  - [127 SOLLTE Standardformate für Zeitdauern verwenden](#127-sollte-standardformate-für-zeitdauern-verwenden)
  - [170 MUSS Standardformate für Land, Sprache, Währung](#170-muss-standardformate-für-land-sprache-währung)
  - [244 SOLLTE Content Negotiation unterstützen](#244-sollte-content-negotiation-unterstützen)
  - [144 SOLLTE UUIDs nur wenn notwendig verwenden](#144-sollte-uuids-nur-wenn-notwendig-verwenden)
- [5. URLs](#5-urls)
  - [C-01 MUSS URL-Versionierung verwenden](#c-01-muss-url-versionierung-verwenden)
  - [134 MUSS Ressourcennamen im Plural](#134-muss-ressourcennamen-im-plural)
  - [228 MUSS URL-kompatible Ressourcen-IDs](#228-muss-url-kompatible-ressourcen-ids)
  - [129 MUSS kebab-case für Pfadsegmente](#129-muss-kebab-case-für-pfadsegmente)
  - [136 MUSS Normalisierte Pfade ohne Trailing Slashes](#136-muss-normalisierte-pfade-ohne-trailing-slashes)
  - [141 MUSS URLs frei von Verben halten](#141-muss-urls-frei-von-verben-halten)
  - [138 MUSS Aktionen vermeiden — in Ressourcen denken](#138-muss-aktionen-vermeiden-—-in-ressourcen-denken)
  - [142 MUSS Domänenspezifische Ressourcennamen](#142-muss-domänenspezifische-ressourcennamen)
  - [143 MUSS Ressourcen via Pfadsegmente identifizieren](#143-muss-ressourcen-via-pfadsegmente-identifizieren)
  - [130 MUSS snake_case für Query-Parameter](#130-muss-snake_case-für-query-parameter)
  - [137 MUSS Konventionelle Query-Parameter verwenden](#137-muss-konventionelle-query-parameter-verwenden)
  - [135 SOLLTE `/api` nicht als Basispfad](#135-sollte-api-nicht-als-basispfad)
  - [140 SOLLTE Nützliche und notwendige Ressourcen definieren](#140-sollte-nützliche-und-notwendige-ressourcen-definieren)
  - [139 SOLLTE Vollständige Geschäftsprozesse modellieren](#139-sollte-vollständige-geschäftsprozesse-modellieren)
  - [146 SOLLTE Anzahl Ressourcentypen begrenzen](#146-sollte-anzahl-ressourcentypen-begrenzen)
  - [147 SOLLTE Sub-Ressource-Ebenen begrenzen](#147-sollte-sub-ressource-ebenen-begrenzen)
  - [145 KANN Verschachtelte URLs in Betracht ziehen](#145-kann-verschachtelte-urls-in-betracht-ziehen)
  - [241 KANN Zusammengesetzte Schlüssel als Ressourcen-ID](#241-kann-zusammengesetzte-schlüssel-als-ressourcen-id)
- [6. JSON Payload](#6-json-payload)
  - [167 MUSS JSON als Datenformat verwenden](#167-muss-json-als-datenformat-verwenden)
  - [118 MUSS snake_case für Property-Namen (niemals camelCase)](#118-muss-snake_case-für-property-namen-niemals-camelcase)
  - [C-02 MUSS NICHT Kein HATEOAS](#c-02-muss-nicht-kein-hateoas)
  - [174 MUSS Gemeinsame Feldnamen verwenden](#174-muss-gemeinsame-feldnamen-verwenden)
  - [C-07 SOLLTE `metadata`-Feld für erweiterbare Ressourcen](#c-07-sollte-metadata-feld-für-erweiterbare-ressourcen)
  - [C-11 SOLLTE `description` und `metadata` klar trennen](#c-11-sollte-description-und-metadata-klar-trennen)
  - [235 SOLLTE `_at`-Suffix für Datum/Zeit-Properties](#235-sollte-_at-suffix-für-datum-zeit-properties)
  - [240 SOLLTE Enum-Werte in UPPER_SNAKE_CASE](#240-sollte-enum-werte-in-upper_snake_case)
  - [120 SOLLTE Array-Namen im Plural](#120-sollte-array-namen-im-plural)
  - [123 MUSS Gleiche Semantik für `null` und fehlende Properties](#123-muss-gleiche-semantik-für-null-und-fehlende-properties)
  - [122 MUSS `null` nicht für Boolean-Properties](#122-muss-null-nicht-für-boolean-properties)
  - [124 SOLLTE `null` nicht für leere Arrays](#124-sollte-null-nicht-für-leere-arrays)
  - [216 SOLLTE Maps mit `additionalProperties` definieren](#216-sollte-maps-mit-additionalproperties-definieren)
  - [252 SOLLTE Einheitliches Schema für Lesen und Schreiben](#252-sollte-einheitliches-schema-für-lesen-und-schreiben)
  - [172 SOLLTE Standard-Medientypen verwenden](#172-sollte-standard-medientypen-verwenden)
- [7. HTTP-Anfragen](#7-http-anfragen)
  - [148 MUSS HTTP-Methoden korrekt verwenden](#148-muss-http-methoden-korrekt-verwenden)
  - [149 MUSS Gemeinsame Methoden-Eigenschaften einhalten](#149-muss-gemeinsame-methoden-eigenschaften-einhalten)
  - [229 SOLLTE POST und PATCH idempotent gestalten](#229-sollte-post-und-patch-idempotent-gestalten)
  - [231 SOLLTE Sekundärschlüssel für idempotentes POST](#231-sollte-sekundärschlüssel-für-idempotentes-post)
  - [253 KANN Asynchrone Anfrageverarbeitung](#253-kann-asynchrone-anfrageverarbeitung)
  - [154 MUSS Collection-Format für Header und Query-Parameter definieren](#154-muss-collection-format-für-header-und-query-parameter-definieren)
  - [236 SOLLTE Einfache Filter als Query-Parameter](#236-sollte-einfache-filter-als-query-parameter)
  - [237 SOLLTE Komplexe Filter als JSON-Body (POST)](#237-sollte-komplexe-filter-als-json-body-post)
  - [226 MUSS Implizite Response-Filterung dokumentieren](#226-muss-implizite-response-filterung-dokumentieren)
- [8. HTTP-Statuscodes](#8-http-statuscodes)
  - [243 MUSS Nur offizielle HTTP-Statuscodes](#243-muss-nur-offizielle-http-statuscodes)
  - [151 MUSS Alle Statuscodes spezifizieren](#151-muss-alle-statuscodes-spezifizieren)
  - [150 SOLLTE Nur gebräuchliche Statuscodes verwenden](#150-sollte-nur-gebräuchliche-statuscodes-verwenden)
  - [220 MUSS Spezifischsten Statuscode verwenden](#220-muss-spezifischsten-statuscode-verwenden)
  - [152 MUSS Code 207 für Batch/Bulk-Requests](#152-muss-code-207-für-batch-bulk-requests)
  - [153 MUSS Code 429 mit Retry-After bei Rate Limits](#153-muss-code-429-mit-retry-after-bei-rate-limits)
  - [176 MUSS Problem JSON für alle Fehler (RFC 7807)](#176-muss-problem-json-für-alle-fehler-rfc-7807)
  - [177 MUSS Keine Stack Traces in Fehler-Responses](#177-muss-keine-stack-traces-in-fehler-responses)
  - [251 SOLLTE Keine Weiterleitungs-Codes](#251-sollte-keine-weiterleitungs-codes)
- [9. HTTP-Header](#9-http-header)
  - [178 MUSS `Content-*` Header korrekt verwenden](#178-muss-content-header-korrekt-verwenden)
  - [✦ C-03 / C-04 · MUSS · W3C Trace Context (ersetzt #233 X-Flow-ID)](#c-03-c-04-muss-w3c-trace-context-ersetzt-233-x-flow-id)
  - [C-05 SOLLTE `trace_id` in 5xx Problem JSON Responses](#c-05-sollte-trace_id-in-5xx-problem-json-responses)
  - [132 SOLLTE kebab-case mit Grossbuchstaben für eigene HTTP-Header](#132-sollte-kebab-case-mit-grossbuchstaben-für-eigene-http-header)
  - [180 SOLLTE `Location` Header nach POST](#180-sollte-location-header-nach-post)
  - [182 KANN ETag mit If-Match / If-None-Match](#182-kann-etag-mit-if-match-if-none-match)
  - [230 KANN Idempotency-Key Header](#230-kann-idempotency-key-header)
  - [181 KANN Prefer Header](#181-kann-prefer-header)
- [10. Performance](#10-performance)
  - [155 SOLLTE Bandbreite reduzieren und Antwortzeiten verbessern](#155-sollte-bandbreite-reduzieren-und-antwortzeiten-verbessern)
  - [227 MUSS Cacheable Endpunkte dokumentieren](#227-muss-cacheable-endpunkte-dokumentieren)
  - [156 SOLLTE gzip-Komprimierung unterstützen](#156-sollte-gzip-komprimierung-unterstützen)
  - [157 SOLLTE Partial Responses via Feldauswahl](#157-sollte-partial-responses-via-feldauswahl)
  - [158 SOLLTE Einbetten von Sub-Ressourcen erlauben](#158-sollte-einbetten-von-sub-ressourcen-erlauben)
- [11. Paginierung](#11-paginierung)
  - [159 MUSS Paginierung für alle Collection-Ressourcen](#159-muss-paginierung-für-alle-collection-ressourcen)
  - [160 SOLLTE Cursor-basierte Paginierung bevorzugen](#160-sollte-cursor-basierte-paginierung-bevorzugen)
  - [248 SOLLTE Standard Pagination Response Object](#248-sollte-standard-pagination-response-object)
  - [254 SOLLTE Gesamtanzahl vermeiden](#254-sollte-gesamtanzahl-vermeiden)
- [12. Kompatibilität und Erweiterbarkeit](#12-kompatibilität-und-erweiterbarkeit)
  - [106 MUSS Keine Breaking Changes](#106-muss-keine-breaking-changes)
  - [C-10 MUSS Vier Erweiterungsregeln einhalten](#c-10-muss-vier-erweiterungsregeln-einhalten)
  - [108 MUSS Clients auf Erweiterungen vorbereiten](#108-muss-clients-auf-erweiterungen-vorbereiten)
  - [110 MUSS JSON-Objekte als Top-Level-Datenstruktur](#110-muss-json-objekte-als-top-level-datenstruktur)
  - [111 MUSS OpenAPI Spec als erweiterbar behandeln](#111-muss-openapi-spec-als-erweiterbar-behandeln)
  - [107 SOLLTE Kompatible Erweiterungen bevorzugen](#107-sollte-kompatible-erweiterungen-bevorzugen)
  - [109 SOLLTE APIs konservativ designen](#109-sollte-apis-konservativ-designen)
  - [112 SOLLTE Offene Enum-Listen verwenden](#112-sollte-offene-enum-listen-verwenden)
- [13. Deprecation](#13-deprecation)
  - [187 MUSS Deprecation in API-Spezifikation markieren](#187-muss-deprecation-in-api-spezifikation-markieren)
  - [185 MUSS Konsumenten vor API-Abschaltung informieren und Migrationsfrist gewähren](#185-muss-konsumenten-vor-api-abschaltung-informieren-und-migrationsfrist-gewähren)
  - [186 MUSS Externe Partner bei Deprecation besonders behandeln](#186-muss-externe-partner-bei-deprecation-besonders-behandeln)
  - [188 MUSS Nutzung der deprecated API monitoren](#188-muss-nutzung-der-deprecated-api-monitoren)
  - [191 MUSS Keine deprecated APIs neu verwenden](#191-muss-keine-deprecated-apis-neu-verwenden)
  - [189 SOLLTE Deprecation und Sunset Header](#189-sollte-deprecation-und-sunset-header)
  - [190 SOLLTE Monitoring für Deprecation und Sunset](#190-sollte-monitoring-für-deprecation-und-sunset)
- [14. Betrieb](#14-betrieb)
  - [192 MUSS OpenAPI-Spezifikation veröffentlichen](#192-muss-openapi-spezifikation-veröffentlichen)
  - [193 SOLLTE API-Nutzung monitoren](#193-sollte-api-nutzung-monitoren)
- [Übersicht: Alle eigenen Regeln](#übersicht-alle-eigenen-regeln)
- [Entfernte Zalando-interne Regeln](#entfernte-zalando-interne-regeln)

-----

## 1. Allgemeine Richtlinien

### 100 MUSS API-First-Prinzip befolgen

APIs müssen **vor** der Implementierung spezifiziert werden — nicht danach. Das bedeutet:

- Die API-Spezifikation (OpenAPI) wird als erstes erstellt, bevor Code geschrieben wird
- Kolleginnen und Kollegen sowie Client-Entwickler geben frühzeitig Feedback
- Die API ist stabil, auch wenn sich die Implementierung dahinter ändert

**Warum?** APIs sind Verträge mit Konsumenten. Wer zuerst implementiert und dann dokumentiert, baut Abhängigkeiten ein, die später schwer zu ändern sind.

-----

### 101 MUSS API-Spezifikation mit OpenAPI bereitstellen

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

#### Was “zusammen mit dem Service deployed” bedeutet

Die OpenAPI-Datei muss zur Laufzeit vom Service selbst erreichbar sein — nicht nur in Git liegen. Der Service exposed einen Endpunkt der die aktuelle Spezifikation zurückgibt:

```
GET /openapi.yaml     ← Maschinenlesbare Spezifikation
GET /openapi.json     ← Alternativ als JSON
GET /docs             ← Optional: Swagger UI für Menschen
```

**Warum?** Liegt die Spec nur in Git, kann sie veralten — jemand ändert den Code aber vergisst die Spec zu aktualisieren. Als Teil des Deployments ist sie per Definition immer synchron mit der laufenden Version. Ausserdem können API-Gateways (Gravitee), Testframeworks und Client-Generatoren die Spec direkt zur Laufzeit abrufen.

#### Erreichbarkeit der Spec — nach Zielgruppe

Die Erreichbarkeit des `/openapi.yaml`-Endpunkts richtet sich nach der deklarierten `x-audience`:

|Audience                |Erreichbarkeit               |Absicherung                              |
|------------------------|-----------------------------|-----------------------------------------|
|`external-public`       |Öffentlich über Ingress      |Kein Auth — bewusste Ausnahme zu #104    |
|`external-partner`      |Öffentlich über Ingress      |Kein Auth — bewusste Ausnahme zu #104    |
|`company-internal`      |Nur cluster-intern           |Netzwerkseitig eingeschränkt, kein OAuth2|
|`business-unit-internal`|Nur cluster-intern           |Netzwerkseitig eingeschränkt, kein OAuth2|
|`component-internal`    |Nur pod-intern / service mesh|Kein externer Zugriff                    |


> **Ausnahme zu #104:** Der `/openapi.yaml`-Endpunkt ist ein technischer Meta-Endpunkt und MUSS ohne OAuth2-Authentifizierung erreichbar sein. OAuth2 würde automatisierte Tool-Integration (Gravitee Import, Spektral Lint, Client-Generierung) verhindern. Die Erreichbarkeit wird stattdessen netzwerkseitig kontrolliert.

#### In Gravitee / AKS

```yaml
# Gravitee importiert die Spec direkt vom laufenden Service
source:
  url: https://order-service.internal/openapi.yaml

# Kubernetes Ingress — nur für externe APIs öffentlich
ingress:
  rules:
    - path: /openapi.yaml
      backend: order-service
    - path: /v1/
      backend: gravitee-gateway
```

-----

### 102 SOLLTE API-Benutzerhandbuch bereitstellen

Zusätzlich zur technischen Spezifikation sollte ein Benutzerhandbuch für API-Konsumenten existieren mit:

- Zweck und Anwendungsfällen der API
- Konkreten Beispielen zur Nutzung
- Typischen Fehlerfällen und wie man sie behebt
- Architekturkontext und wichtige Abhängigkeiten

Das Handbuch wird über `#/externalDocs/url` in der OpenAPI-Spezifikation verlinkt.

-----

### 103 MUSS APIs auf amerikanischem Englisch schreiben

Alle API-Bezeichnungen, Beschreibungen, Fehlermeldungen und Dokumentationen müssen auf **US-Englisch** verfasst sein. Das gilt für:

- Ressourcennamen (`/orders`, `/customers`)
- Property-Namen (`order_id`, `created_at`)
- OpenAPI `description`-Felder
- Fehlermeldungen und Problem-JSON-Texte

-----

### C-08 MUSS Minimale API-Oberfläche (YAGNI-Prinzip)

Jedes API-Design MUSS auf eine minimale API-Oberfläche abzielen, ohne Produktanforderungen zu vernachlässigen.

- Keine Ressourcen, Relationen, Aktionen oder Felder die noch nicht gebraucht werden
- Keine vorauseilende Generalisierung
- Neue Funktionalität wird erst hinzugefügt wenn ein konkreter Bedarf besteht

**YAGNI:** “You Ain’t Gonna Need It” — was heute nicht gebraucht wird, kommt auch nicht rein.

```
# Falsch: generische "items"-Ressource für alle Entitäten
GET /v1/items?type=order

# Richtig: spezifische Ressource nur wenn gebraucht
GET /v1/orders
```

-----

### C-09 MUSS Robustheit nach Postel’s Law

Jede API-Implementierung und jeder API-Konsument MUSS Postel’s Law befolgen:

> *“Be conservative in what you send, be liberal in what you accept.”*

**Als Server:**

- Nur notwendige Daten senden — niemals mehr als erforderlich
- Keine internen Details, Stack Traces oder Debug-Informationen exponieren

**Als Client:**

- Unbekannte Properties ignorieren (nicht mit Fehler ablehnen)
- Neue Enum-Werte tolerieren
- Zusätzliche HTTP-Header tolerieren

Dies stärkt Kompatibilität und Erweiterbarkeit des gesamten API-Ökosystems.

→ Die Client-seitige Umsetzung ist in #108 (Clients auf Erweiterungen vorbereiten) beschrieben.
→ Offene Enum-Listen als konkrete Anwendung von Postel’s Law: #112 (Offene Enum-Listen verwenden).

-----

### C-12 MUSS API-Spezifikationen in Git versionieren

OpenAPI-Spezifikationen MÜSSEN in einem Versionskontrollsystem (Git) verwaltet werden:

- Gleiche Repository-Konventionen wie Code
- Git Tags für jede veröffentlichte API-Version: `api/v1.2.0`
- `CHANGELOG.md` dokumentiert alle Breaking Changes und Deprecations
- Pull Requests für alle API-Änderungen — kein direktes Commit auf `main`

-----

## 2. Meta-Informationen

### 218 MUSS API Meta-Informationen enthalten

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

-----

### 116 MUSS Semantic Versioning verwenden

Die API-Spec-Version folgt dem Schema `MAJOR.MINOR.PATCH`:

|Änderung                           |Aktion                          |
|-----------------------------------|--------------------------------|
|Breaking Change (inkompatibel)     |MAJOR erhöhen: `1.x.x` → `2.0.0`|
|Neue Funktion (rückwärtskompatibel)|MINOR erhöhen: `1.2.x` → `1.3.0`|
|Bugfix / Typo in Doku              |PATCH erhöhen: `1.2.3` → `1.2.4`|

**Wichtig:** Diese Versionsnummer betrifft die API-Spezifikationsdatei — nicht die URL-Version (siehe ✦ C-01).

-----

### 215 MUSS API-Identifier bereitstellen

Jede API bekommt eine **global eindeutige, unveränderliche UUID** als `x-api-id`. Diese ändert sich nie — auch nicht bei Breaking Changes oder Umbenennung der API.

```yaml
info:
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70
```

**Warum?** Der Identifier erlaubt die lückenlose Nachverfolgung der API-Evolution über alle Versionen hinweg, unabhängig von Namensänderungen.

-----

### 219 MUSS API-Zielgruppe angeben

Jede API muss ihre Zielgruppe deklarieren. Dies steuert Qualitätsanforderungen, Review-Prozesse und Zugriffsrechte:

|Wert                    |Bedeutung                                   |
|------------------------|--------------------------------------------|
|`component-internal`    |Nur innerhalb derselben Anwendungskomponente|
|`business-unit-internal`|Innerhalb derselben Business Unit           |
|`company-internal`      |Alle internen Teams des Unternehmens        |
|`external-partner`      |Externe Geschäftspartner                    |
|`external-public`       |Öffentlich zugänglich für alle              |

```yaml
info:
  x-audience: external-partner
```

-----

## 3. Sicherheit

### 104 MUSS Alle Endpunkte absichern

Jeder fachliche API-Endpunkt muss durch Authentifizierung und Autorisierung geschützt sein. Empfohlene Methoden:

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

#### Endpunkt-Kategorien und Absicherung

Nicht alle Endpunkte eines Services sind fachliche API-Endpunkte. Es gibt drei Kategorien mit unterschiedlichen Anforderungen:

|Kategorie                    |Beispiele                               |Absicherung                            |
|-----------------------------|----------------------------------------|---------------------------------------|
|**Fachliche Endpunkte**      |`GET /v1/orders`, `POST /v1/orders`     |MUSS OAuth2/JWT — keine Ausnahme       |
|**Infrastruktur-Endpunkte**  |`/health`, `/ready`, `/live`, `/startup`|MUSS öffentlich sein — Ausnahme zu #104|
|**Technische Meta-Endpunkte**|`/openapi.yaml`, `/metrics`             |Situationsabhängig — siehe unten       |

#### Ausnahme: Infrastruktur-Endpunkte

Health-, Readiness-, Liveness- und Startup-Endpunkte MÜSSEN ohne Authentifizierung erreichbar sein:

```
GET /health    → Liveness:  Läuft der Prozess?
GET /ready     → Readiness: Kann der Service Traffic annehmen?
GET /live      → Synonym für Liveness (je nach Framework)
GET /startup   → Ist die Initialisierung abgeschlossen?
```

**Warum?** Kubernetes ruft diese Endpunkte ohne Token auf — es gibt keinen Mechanismus einen Bearer Token mitzugeben. Ein gesicherter Health-Check der `401` zurückgibt würde dazu führen, dass Kubernetes den Pod als unhealthy markiert und ihn neu startet.

**Was diese Endpunkte zurückgeben dürfen:**

```json
// ✓ Richtig — minimale Information
{ "status": "UP" }

// ✗ Falsch — zu viel interne Information exponiert
{
  "status": "UP",
  "database": "postgresql://internal-db:5432/orders",
  "build_version": "1.2.3-internal-456",
  "hostname": "pod-abc123-xyz"
}
```

Keine internen Details exponieren — auch öffentliche Endpunkte minimieren die Angriffsfläche.

#### Ausnahme: `/openapi.yaml`

Der Spezifikations-Endpunkt ist ein technischer Meta-Endpunkt ohne OAuth2. Die Erreichbarkeit wird stattdessen netzwerkseitig über die `x-audience` kontrolliert — siehe #101 für Details.

#### Ausnahme: `/metrics`

Prometheus-Metriken enthalten interne Laufzeitdaten (Speicherverbrauch, Request-Raten, Fehlerquoten) und DÜRFEN NICHT öffentlich erreichbar sein. Sie werden netzwerkseitig abgesichert — nur aus dem Cluster erreichbar, nicht über den öffentlichen Ingress:

```yaml
# Kubernetes NetworkPolicy — /metrics nur cluster-intern
ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring    # Nur Prometheus-Namespace
    ports:
    - port: 9090
```

#### In der OpenAPI-Spezifikation dokumentieren

Alle Ausnahmen MÜSSEN in der OpenAPI-Spezifikation explizit mit `security: []` (leeres Array = kein Auth) gekennzeichnet sein:

```yaml
paths:
  /health:
    get:
      summary: Health Check
      security: []            # Explizit kein Auth — bewusste Ausnahme
      tags: [Infrastructure]
      responses:
        '200':
          description: Service ist healthy
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [UP, DOWN]
  /ready:
    get:
      summary: Readiness Check
      security: []            # Explizit kein Auth
      tags: [Infrastructure]
      responses:
        '200':
          description: Service kann Traffic annehmen
        '503':
          description: Service ist noch nicht bereit
```

-----

### 105 MUSS Berechtigungen (Scopes) definieren und zuweisen

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

-----

### C-06 MUSS Einheitliche Scope-Namenskonvention

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

|Aktion |Bedeutung                                                         |Typische HTTP-Methoden               |
|-------|------------------------------------------------------------------|-------------------------------------|
|`read` |Lesender Zugriff                                                  |`GET`, `HEAD`                        |
|`write`|Schreibender Zugriff (anlegen, ändern, löschen)                   |`POST`, `PUT`, `PATCH`, `DELETE`     |
|`admin`|Administrative Operationen (z.B. Konfiguration, Massenoperationen)|`POST`, `DELETE` auf Admin-Endpunkten|

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

-----

## 4. Datenformate

### 238 MUSS Standarddatenformate verwenden

Für alle Datentypen müssen die OpenAPI-Standardformate verwendet werden:

|Typ      |Format                        |Beispiel                             |
|---------|------------------------------|-------------------------------------|
|`integer`|`int32` / `int64` / `bigint`  |`42`, `7721071004`                   |
|`number` |`float` / `double` / `decimal`|`3.14`, `99.95`                      |
|`string` |`date`                        |`"2024-01-15"`                       |
|`string` |`date-time`                   |`"2024-01-15T10:30:00Z"`             |
|`string` |`time`                        |`"10:30:00Z"`                        |
|`string` |`duration`                    |`"P1DT3H"` (1 Tag, 3 Stunden)        |
|`string` |`email`                       |`"user@example.com"`                 |
|`string` |`uri`                         |`"https://api.example.com/v1/orders"`|
|`string` |`uuid`                        |`"e2ab873e-b295-11e9-9c02-..."`      |
|`string` |`iso-639-1`                   |`"de"`, `"en"`                       |
|`string` |`iso-3166-alpha-2`            |`"DE"`, `"GB"`                       |
|`string` |`iso-4217`                    |`"EUR"`, `"USD"`                     |

-----

### 171 MUSS Format für Zahlen und Integer definieren

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

-----

### 169 MUSS Standardformate für Datum/Zeit verwenden

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

-----

### 255 SOLLTE Geeignete Datum/Zeit-Formate wählen

|Format           |Verwendung                          |Beispiel                         |
|-----------------|------------------------------------|---------------------------------|
|`date-time`      |Exakter Zeitpunkt (UTC)             |Bestellzeitpunkt, Lieferzeitpunkt|
|`date`           |Nur Datum ohne Uhrzeit              |Geburtstag, Lieferdatum          |
|`time-local`     |Lokale Uhrzeit (ohne UTC)           |Öffnungszeiten                   |
|`date-time-local`|Lokaler Zeitpunkt + Zeitzone separat|Kampagnenstartzeit               |

-----

### 127 SOLLTE Standardformate für Zeitdauern verwenden

Zeitdauern und Intervalle müssen als ISO 8601 Strings dargestellt werden:

```
"P1DT3H4S"          # 1 Tag, 3 Stunden, 4 Sekunden
"PT30M"             # 30 Minuten
"2024-01-01T00:00:00Z/2024-12-31T23:59:59Z"   # Intervall
"2024-01-01T00:00:00Z/P1Y"                    # Anfang + Dauer
```

Query-Parameter für Zeitintervalle: `{feld}_between` statt `{feld}_before` + `{feld}_after`.

-----

### 170 MUSS Standardformate für Land, Sprache, Währung

|Datentyp        |Standard          |Format            |Beispiel            |
|----------------|------------------|------------------|--------------------|
|Land            |ISO 3166-1 alpha-2|`iso-3166-alpha-2`|`"DE"`, `"GB"`      |
|Sprache         |ISO 639-1         |`iso-639-1`       |`"de"`, `"en"`      |
|Sprache + Region|BCP 47            |`bcp47`           |`"de-AT"`, `"en-GB"`|
|Währung         |ISO 4217          |`iso-4217`        |`"EUR"`, `"USD"`    |

-----

### 244 SOLLTE Content Negotiation unterstützen

Wenn eine Ressource in verschiedenen Formaten geliefert werden kann, soll Content Negotiation über Standard-HTTP-Header verwendet werden. Die erlaubten Medientypen sind in #172 (Standard-Medientypen) definiert.

```
Accept: application/json
Accept: application/pdf
Accept-Language: de
Accept-Encoding: gzip
```

→ Siehe #172 für die Liste der erlaubten Medientypen.

-----

### 144 SOLLTE UUIDs nur wenn notwendig verwenden

UUIDs sind sinnvoll für dezentrale ID-Generierung ohne Koordination. Sie haben aber Nachteile: schwer lesbar, nicht sortierbar, hoher Speicherverbrauch. Alternativen prüfen, z.B. serverseitige ID-Generierung via POST.

-----

## 5. URLs

### C-01 MUSS URL-Versionierung verwenden

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

-----

### 134 MUSS Ressourcennamen im Plural

Ressourcen sind immer im Plural. In Kombination mit #129 (kebab-case) gilt:

```
/v1/orders          ✓ Plural + kebab-case
/v1/order-items     ✓ Plural + kebab-case (mehrere Wörter mit Bindestrich)
/v1/order           ✗ Singular
/v1/orderItems      ✗ Plural ✓, aber camelCase — verletzt #129
/v1/order_items     ✗ Plural ✓, aber snake_case — verletzt #129
```

→ Siehe #129 für kebab-case Anforderung an Pfadsegmente.

-----

### 228 MUSS URL-kompatible Ressourcen-IDs

IDs in URLs dürfen nur enthalten: `[a-zA-Z0-9:._\-/]*`

Keine Sonderzeichen, keine Leerzeichen, keine leeren Werte.

-----

### 129 MUSS kebab-case für Pfadsegmente

Pfadsegmente bestehen nur aus Kleinbuchstaben und Bindestrichen:

```
/v1/order-items         ✓ kebab-case
/v1/orderItems          ✗ camelCase
/v1/order_items         ✗ snake_case
```

-----

### 136 MUSS Normalisierte Pfade ohne Trailing Slashes

```
/v1/orders/123          ✓
/v1/orders/123/         ✗ Trailing Slash verboten
/v1//orders/123         ✗ Leeres Segment verboten
```

-----

### 141 MUSS URLs frei von Verben halten

URLs beschreiben **Ressourcen**, nicht **Aktionen**:

```
GET  /v1/orders              ✓ Liste abrufen
POST /v1/orders              ✓ Erstellen
POST /v1/order-cancellations ✓ Stornierung als Ressource

GET  /v1/getOrders           ✗ Verb in URL
POST /v1/cancelOrder         ✗ Verb in URL
```

#### Ausnahme: Standardisierte Sub-Ressourcen-Suffixe

Drei Suffixe sind als bewusste Ausnahme erlaubt — wenn das jeweilige HTTP-Verb semantisch nicht ausreicht oder technisch nicht funktioniert:

|Suffix   |Methode|Wann verwenden                                                                               |
|---------|-------|---------------------------------------------------------------------------------------------|
|`/search`|`POST` |Komplexe Filter die nicht als Query-Parameter passen (Längenlimit, verschachtelte Strukturen)|
|`/batch` |`POST` |Mehrere Operationen in einem Request (Create, Update, Delete)                                |
|`/export`|`POST` |Asynchrone Generierung grosser Datenmengen                                                   |

```
POST /v1/orders/search    ✓ Standardisierte Ausnahme
POST /v1/orders/batch     ✓ Standardisierte Ausnahme
POST /v1/orders/export    ✓ Standardisierte Ausnahme

POST /v1/orders/find      ✗ Synonym — nicht standardisiert
POST /v1/orders/query     ✗ Synonym — nicht standardisiert
POST /v1/orders/lookup    ✗ Synonym — nicht standardisiert
POST /v1/searchOrders     ✗ Verb als Ressourcenname — verboten
```

**Warum ist `/search` keine echte Verb-Verletzung?** `/search` bezeichnet hier keine Aktion die ausgeführt wird, sondern eine **spezialisierte Sub-Ressource** des Collections-Endpunkts — ähnlich wie `/v1/orders/{id}` eine Einzelressource ist. Der Unterschied zu verbotenen Verben wie `/getOrders` liegt darin, dass `/search` eine eigene, stabile Ressource mit definiertem Verhalten ist — kein Prozeduraufruf.

**Wann `/search` verwenden vs. Query-Parameter:**

```
# Einfache Filter → Query-Parameter (GET)
GET /v1/orders?status=OPEN&customer_id=abc123

# Komplexe Filter → /search (POST)
POST /v1/orders/search
{
  "filter": {
    "status": ["OPEN", "IN_PROGRESS"],
    "created_at": { "gte": "2024-01-01T00:00:00Z" },
    "total_amount": { "gte": 100.00, "lte": 500.00 }
  },
  "sort": ["-created_at"],
  "limit": 20
}
```

-----

### 138 MUSS Aktionen vermeiden — in Ressourcen denken

REST modelliert Ressourcen, nicht Prozeduraufrufe:

```
PUT /v1/article-locks/{article-id}   ✓ Ressource
POST /v1/articles/{id}/lock          ✗ Aktion
```

-----

### 142 MUSS Domänenspezifische Ressourcennamen

Namen sollen den Geschäftskontext widerspiegeln:

```
/v1/sales-order-items    ✓ Klar und spezifisch
/v1/items                ✗ Zu generisch
```

-----

### 143 MUSS Ressourcen via Pfadsegmente identifizieren

```
/v1/orders/{order-id}/items/{item-id}
```

Jedes Teilsegment muss für sich allein eine gültige Ressource sein.

-----

### 130 MUSS snake_case für Query-Parameter

```
?page_size=20    ✓
?pageSize=20     ✗ camelCase verboten
```

-----

### 137 MUSS Konventionelle Query-Parameter verwenden

|Parameter|Bedeutung                                            |
|---------|-----------------------------------------------------|
|`q`      |Generische Suchanfrage                               |
|`sort`   |Sortierung: `+created_at` (asc), `-created_at` (desc)|
|`fields` |Feldauswahl: `?fields=id,status,created_at`          |
|`embed`  |Sub-Ressourcen einbetten                             |
|`cursor` |Cursor für Pagination                                |
|`limit`  |Maximale Anzahl Ergebnisse                           |

-----

### 135 SOLLTE `/api` nicht als Basispfad

```
/v1/orders          ✓
/api/v1/orders      ✗ Unnötiger /api Präfix
```

-----

### 140 SOLLTE Nützliche und notwendige Ressourcen definieren

Eine Ressource sollte 90% der Anwendungsfälle abdecken. Zu granulare oder zu generische Ressourcen vermeiden. Neue Ressource erst einführen wenn ein konkreter, aktueller Bedarf besteht — nicht auf Vorrat (YAGNI, siehe C-08).

-----

### 139 SOLLTE Vollständige Geschäftsprozesse modellieren

Eine API sollte alle Ressourcen eines Geschäftsprozesses enthalten, damit Clients den Ablauf nachvollziehen können.

-----

### 146 SOLLTE Anzahl Ressourcentypen begrenzen

Erfahrungswert: gut designte APIs haben 4–8 Ressourcentypen.

-----

### 147 SOLLTE Sub-Ressource-Ebenen begrenzen

Maximal **3 Ebenen** Verschachtelung:

```
/v1/orders/{id}/items/{item-id}/attachments/{att-id}    ✓ 3 Ebenen max.
/v1/a/{id}/b/{id}/c/{id}/d/{id}                        ✗ Zu tief
```

-----

### 145 KANN Verschachtelte URLs in Betracht ziehen

Nested URLs nur wenn die Sub-Ressource ohne Elternressource nicht existiert.

-----

### 241 KANN Zusammengesetzte Schlüssel als Ressourcen-ID

```
/v1/price-advices/{sku}/{sales-channel}
```

-----

## 6. JSON Payload

### 167 MUSS JSON als Datenformat verwenden

Alle Request- und Response-Bodies verwenden JSON (RFC 7159):

- UTF-8 Encoding
- Keine duplizierten Property-Namen
- Top-Level ist immer ein JSON-Objekt — niemals direkt ein Array (siehe #110)

-----

### 118 MUSS snake_case für Property-Namen (niemals camelCase)

```json
// ✓ Richtig
{ "order_id": "123", "created_at": "2024-01-15T10:30:00Z" }

// ✗ Falsch — camelCase wie bei Stripe
{ "orderId": "123", "createdAt": "2024-01-15T10:30:00Z" }
```

Regex: `^[a-z_][a-z_0-9]*$`

> **Hinweis:** Stripe verwendet konsequent camelCase, weil ihre Client-Libraries primär auf JavaScript ausgerichtet sind. Für enterprise B2B APIs ist snake_case der breitere Industrie-Standard — bestätigt von Adidas, GitHub, Twilio und AWS.

-----

### C-02 MUSS NICHT Kein HATEOAS

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

-----

### 174 MUSS Gemeinsame Feldnamen verwenden

|Feldname     |Typ                 |Bedeutung                                   |
|-------------|--------------------|--------------------------------------------|
|`id`         |`string`            |Eindeutiger, unveränderlicher Bezeichner    |
|`{entity}_id`|`string`            |Referenz auf andere Ressource (`partner_id`)|
|`created_at` |`string` (date-time)|Erstellungszeitpunkt                        |
|`updated_at` |`string` (date-time)|Letzter Änderungszeitpunkt                  |
|`etag`       |`string`            |ETag für optimistisches Locking             |

-----

### C-07 SOLLTE `metadata`-Feld für erweiterbare Ressourcen

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
  propertyNames:
    pattern: '^[a-z][a-z0-9_]{0,39}$'   # snake_case, max 40 Zeichen
  additionalProperties:
    type: string
    maxLength: 500
  maxProperties: 50
  description: |
    Optionale Key-Value-Paare für Zusatzdaten.
    Keys: snake_case, max. 40 Zeichen.
    Keine sensitiven Informationen speichern.
```

-----

### C-11 SOLLTE `description` und `metadata` klar trennen

|Feld         |Typ     |Zweck                           |Sichtbarkeit               |
|-------------|--------|--------------------------------|---------------------------|
|`description`|`string`|Menschenlesbarer Freitext       |Ggf. im UI/E-Mail angezeigt|
|`metadata`   |`object`|Maschinenlesbare Key-Value-Daten|Nur intern / API           |

```json
{
  "id": "ord_123",
  "description": "2 Shirts für Kundenbestellung Frühjahr",
  "metadata": { "erp_order_id": "ERP-456", "channel": "web" }
}
```

Nie Metadaten in `description` schreiben und nie `description` für maschinenlesbare Daten missbrauchen.

-----

### 235 SOLLTE `_at`-Suffix für Datum/Zeit-Properties

```json
{ "created_at": "2024-01-15T10:30:00Z", "shipped_at": "2024-01-16T08:00:00Z" }
```

-----

### 240 SOLLTE Enum-Werte in UPPER_SNAKE_CASE

```yaml
status:
  type: string
  enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]
```

-----

### 120 SOLLTE Array-Namen im Plural

```json
{ "items": [...], "addresses": [...] }
```

-----

### 123 MUSS Gleiche Semantik für `null` und fehlende Properties

Fehlendes Feld und explizites `null` MÜSSEN für den Konsumenten identisch behandelt werden — sie dürfen keine unterschiedliche fachliche Bedeutung haben.

```json
// Diese beiden müssen für den Konsumenten identisch sein
{ "id": "ord_123" }                          // Feld fehlt
{ "id": "ord_123", "discount": null }        // Feld ist null
```

**Warum?** Clients die eine API konsumieren sind in verschiedenen Sprachen und Frameworks implementiert. Manche deserialisieren fehlendes Feld als `null`, andere als `undefined`, andere als Default-Wert. Wenn `null` und “fehlt” unterschiedliche Semantiken hätten, wäre das Verhalten je nach Client-Implementierung verschieden — ein schwer debuggbarer Fehler.

**Konsequenz in OpenAPI:**

```yaml
# ✓ Richtig — optional, aber nie null
discount_percentage:
  type: number
  format: float
  # kein required → darf fehlen
  # kein nullable  → darf nicht null sein

# ✗ Vermeiden — darf fehlen UND null sein (doppelte Semantik)
discount_percentage:
  type: number
  nullable: true   # Nicht kombinieren mit fehlendem required
```

**Robuste Client-Implementierung:**

```javascript
// ✓ Fehlendes Feld und null gleich behandeln
const discount = order.discount_percentage ?? null;
```

-----

### 122 MUSS `null` nicht für Boolean-Properties

Ein Boolean kennt per Definition nur zwei Zustände: `true` oder `false`. Ein dritter Zustand `null` — meistens “unbekannt” oder “noch nicht entschieden” — ist ein **eigener fachlicher Zustand** der einen eigenen Typ verdient.

#### Zusammenspiel mit #123 — Boolean weglassen statt null

#122 und #123 zusammen ergeben eine klare Konsequenz:

```
null für Boolean verboten (#122)
+
null und fehlendes Feld sind semantisch gleich (#123)
=
Fehlendes Feld ist die null-freie Alternative — und die korrekte Form
```

Ein Boolean der `null` sein würde, wird **einfach weggelassen**. Das Fehlen des Feldes übernimmt die Semantik von “nicht gesetzt”.

#### Die drei Fälle

**Fall 1 — Nur true/false möglich, immer vorhanden:**

```yaml
# OpenAPI
is_active:
  type: boolean
  # required: true → immer vorhanden, nullable: false
```

```json
{ "is_active": true }     // ✓
{ "is_active": false }    // ✓
{ "is_active": null }     // ✗ verboten
```

**Fall 2 — Boolean optional, Feld kann fehlen:**

Wenn `false` und “nicht gesetzt” dieselbe fachliche Bedeutung haben, darf das Feld weggelassen werden:

```yaml
# OpenAPI
is_gift_wrapping_requested:
  type: boolean
  nullable: false    # null explizit verboten
  # kein required   # Feld darf fehlen — ersetzt null
  description: |
    Fehlt das Feld: noch keine Auswahl getroffen (equivalent zu null).
    true:  Geschenkverpackung gewünscht.
    false: Keine Geschenkverpackung gewünscht.
```

```json
// ✓ Kunde hat noch nicht gewählt — Feld fehlt
{ "id": "ord_123", "status": "OPEN" }

// ✓ Aktiv Ja gewählt
{ "id": "ord_123", "is_gift_wrapping_requested": true }

// ✓ Aktiv Nein gewählt
{ "id": "ord_123", "is_gift_wrapping_requested": false }

// ✗ Verboten — null statt weglassen
{ "id": "ord_123", "is_gift_wrapping_requested": null }
```

**Fall 3 — Drei fachlich unterschiedliche Zustände:**

Wenn `false`, `null` und “fehlt” **unterschiedliche** fachliche Bedeutungen hätten, reicht Boolean nicht aus — Enum verwenden:

```yaml
# ✗ Problematisch — false und "fehlt" haben verschiedene Bedeutungen
{ "is_terms_accepted": false }   // Aktiv abgelehnt
{}                                // Noch nicht entschieden — oder auch abgelehnt?

# ✓ Richtig — Enum macht alle Zustände explizit und benannt
terms_acceptance:
  type: string
  enum: [ACCEPTED, DECLINED, PENDING]
```

```json
{ "terms_acceptance": "ACCEPTED" }   // ✓ Aktiv zugestimmt
{ "terms_acceptance": "DECLINED" }   // ✓ Aktiv abgelehnt
{ "terms_acceptance": "PENDING" }    // ✓ Noch nicht entschieden
```

#### Entscheidungsbaum

```
Brauche ich einen dritten Zustand neben true/false?
├── Nein
│   ├── Feld immer vorhanden → Boolean required: true
│   └── Feld optional        → Boolean ohne required, ohne nullable
│                               (Fehlen = "nicht gesetzt")
└── Ja — drei Zustände fachlich unterschiedlich
    └── Enum verwenden (niemals nullable Boolean)
```

#### Übersicht

|Situation                       |Lösung                                     |Beispiel                                         |
|--------------------------------|-------------------------------------------|-------------------------------------------------|
|Immer true oder false           |`boolean`, `required: true`                |`is_active`                                      |
|Optional, Fehlen = nicht gesetzt|`boolean`, kein `required`, kein `nullable`|`is_gift_wrapping_requested`                     |
|Drei fachliche Zustände         |`string` Enum                              |`terms_acceptance: ACCEPTED / DECLINED / PENDING`|
|`nullable: true` Boolean        |✗ Verboten nach #122                       |—                                                |

-----

### 124 SOLLTE `null` nicht für leere Arrays

```json
{ "items": [] }    // ✓
{ "items": null }  // ✗
```

-----

### 216 SOLLTE Maps mit `additionalProperties` definieren

```yaml
translations:
  type: object
  additionalProperties:
    type: string
  description: Schlüssel sind BCP-47 Sprachcodes (z.B. "de", "en-GB")
```

-----

### 252 SOLLTE Einheitliches Schema für Lesen und Schreiben

Dasselbe Schema für GET und POST/PUT/PATCH — Unterschiede via `readOnly: true` / `writeOnly: true`.

-----

### 172 SOLLTE Standard-Medientypen verwenden

|Content-Type              |Verwendung       |
|--------------------------|-----------------|
|`application/json`        |Standard JSON    |
|`application/problem+json`|Fehler (RFC 7807)|
|`application/pdf`         |PDF-Dokumente    |
|`multipart/form-data`     |Datei-Uploads    |

-----

## 7. HTTP-Anfragen

### 148 MUSS HTTP-Methoden korrekt verwenden

|Methode |Semantik                      |Idempotent|Sicher|
|--------|------------------------------|----------|------|
|`GET`   |Ressource lesen               |✓         |✓     |
|`POST`  |Ressource erstellen           |✗         |✗     |
|`PUT`   |Ressource vollständig ersetzen|✓         |✗     |
|`PATCH` |Ressource partiell ändern     |✗         |✗     |
|`DELETE`|Ressource löschen             |✓         |✗     |
|`HEAD`  |Wie GET, nur Header           |✓         |✓     |

**GET** darf keinen Request-Body haben. Bei komplexen Suchanfragen die nicht als Query-Parameter passen → `POST /resource/search` mit JSON-Body verwenden (siehe #237).

-----

### 149 MUSS Gemeinsame Methoden-Eigenschaften einhalten

- **Sicher (Safe):** GET, HEAD — dürfen den Zustand nicht verändern
- **Idempotent:** GET, PUT, DELETE — mehrfache Ausführung hat denselben Effekt

-----

### 229 SOLLTE POST und PATCH idempotent gestalten

Idempotente POST/PATCH-Requests verhindern Duplikate bei Netzwerkfehlern — via `Idempotency-Key` Header oder Sekundärschlüssel.

-----

### 231 SOLLTE Sekundärschlüssel für idempotentes POST

```json
POST /v1/orders
{
  "external_order_id": "EXT-2024-001",
  "items": [...]
}
```

Bei Wiederholung mit gleichem `external_order_id` → dieselbe Bestellung zurückgeben, nicht neu anlegen.

-----

### 253 KANN Asynchrone Anfrageverarbeitung

Langläufige Operationen können asynchron verarbeitet werden:

1. `POST /v1/exports` → `202 Accepted` + `Location: /v1/exports/{job-id}`
1. `GET /v1/exports/{job-id}` → Status prüfen
1. Bei Fertigstellung → `200` mit fertigem Ergebnis, oder `303 See Other` als Redirect (bewusste Ausnahme zu #251 — Weiterleitungen vermeiden)

-----

### 154 MUSS Collection-Format für Header und Query-Parameter definieren

#### Kontext

Viele Parameter können mehrere Werte gleichzeitig annehmen — zum Beispiel mehrere Statuswerte filtern, mehrere Felder sortieren oder mehrere IDs abfragen. Ohne eine dokumentierte Konvention, wie mehrere Werte in einem Parameter übergeben werden, entscheiden Entwickler das spontan und inkonsistent. Das Ergebnis: APIs in denen `?status=OPEN,CANCELLED`, `?status=OPEN&status=CANCELLED` und `?status[]=OPEN&status[]=CANCELLED` gleichzeitig im Einsatz sind — alle leicht unterschiedlich.

Diese Regel verlangt: **Jeder Parameter der mehrere Werte annehmen kann, muss in der OpenAPI-Spezifikation explizit dokumentieren, welches Format verwendet wird.**

#### Die vier Collection-Formate in OpenAPI

OpenAPI 3.1 kennt vier Serialisierungsformate für Arrays in Query-Parametern, gesteuert durch `style` und `explode`:

|Format            |`style`         |`explode`|Beispiel für `?status=OPEN,CANCELLED`|
|------------------|----------------|---------|-------------------------------------|
|**csv** (Standard)|`form`          |`false`  |`?status=OPEN,CANCELLED`             |
|**multi**         |`form`          |`true`   |`?status=OPEN&status=CANCELLED`      |
|**ssv**           |`spaceDelimited`|`false`  |`?status=OPEN%20CANCELLED`           |
|**pipes**         |`pipeDelimited` |`false`  |`?status=OPEN|CANCELLED`             |

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

|Parameter|Format        |Beispiel                      |Erklärung                                     |
|---------|--------------|------------------------------|----------------------------------------------|
|`sort`   |csv mit Präfix|`?sort=+created_at,-status`   |`+` aufsteigend, `-` absteigend, kommagetrennt|
|`fields` |csv           |`?fields=id,status,created_at`|Feldauswahl, kommagetrennt                    |
|`embed`  |csv           |`?embed=items,customer`       |Sub-Ressourcen einbetten, kommagetrennt       |
|`cursor` |single        |`?cursor=eyJpZCI6IjEyMyJ9`    |Einzelwert, kein Array                        |
|`limit`  |single        |`?limit=20`                   |Einzelwert, kein Array                        |
|`q`      |single        |`?q=winter+jacket`            |Suchbegriff, Leerzeichen URL-encoded          |

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

-----

### 236 SOLLTE Einfache Filter als Query-Parameter

Für einfache, flache Filterausdrücke Query-Parameter verwenden. Sobald Filter verschachtelt werden, die URL-Länge ~1500 Zeichen überschreitet oder mehr als 3 Filterkriterien kombiniert werden → #237 (POST /search) verwenden.

```
GET /v1/orders?status=OPEN&customer_id=abc123
GET /v1/orders?created_at_between=2024-01-01/2024-12-31
```

→ Siehe #237 für komplexe Filter und die vollständige Entscheidungstabelle.

-----

### 237 SOLLTE Komplexe Filter als JSON-Body (POST)

Bei komplexen Filterausdrücken die als Query-Parameter nicht ausreichen, wird `POST` mit JSON-Body verwendet. Der Endpunkt heisst `/search` — als standardisierte Ausnahme zu #141 (Verb-freie URLs).

#### Wann GET mit Query-Parametern reicht (#236) vs. wann /search nötig ist

|Kriterium     |GET + Query-Parameter      |POST /search                                   |
|--------------|---------------------------|-----------------------------------------------|
|Anzahl Filter |1–3 einfache Filter        |Viele oder verschachtelte Filter               |
|URL-Länge     |Unter ~1500 Zeichen        |Würde Längenlimit überschreiten                |
|Filterstruktur|Flach (`?status=OPEN`)     |Verschachtelt (`gte`, `lte`, `in`, `and`, `or`)|
|Cacheability  |✓ Cachebar (GET)           |✗ Nicht cachebar (POST)                        |
|Lesbarkeit    |✓ Direkt im Browser testbar|Braucht HTTP-Client                            |

**Faustregel:** Erst Query-Parameter versuchen — nur wenn sie nicht ausreichen `/search` verwenden.

#### Vollständiges Beispiel

```
POST /v1/orders/search
```

```json
{
  "filter": {
    "status": ["OPEN", "IN_PROGRESS"],
    "created_at": {
      "gte": "2024-01-01T00:00:00Z",
      "lte": "2024-12-31T23:59:59Z"
    },
    "total_amount": { "gte": 100.00 },
    "customer_id": "cust_456"
  },
  "sort": ["-created_at", "+status"],
  "limit": 20,
  "cursor": "eyJpZCI6ImFiYzEyMyJ9"
}
```

Response — identisch mit GET `/v1/orders`:

```json
{
  "items": [
    { "id": "ord_123", "status": "OPEN", "created_at": "2024-06-15T10:30:00Z" }
  ],
  "cursor": {
    "next": "eyJpZCI6Im9yZF8xMjMifQ",
    "prev": null
  }
}
```

#### Hinweis zu #141 — kein Widerspruch

`/search` sieht aus wie ein Verb in der URL — ist aber eine **bewusste, standardisierte Ausnahme** zu #141. Nur diese drei Suffixe sind erlaubt: `/search`, `/batch`, `/export`. Synonyme wie `/find`, `/query` oder `/lookup` sind verboten. Siehe #141 für die vollständige Begründung.

-----

### 226 MUSS Implizite Response-Filterung dokumentieren

Wenn ein Endpunkt automatisch filtert (z.B. nur eigene Daten), muss dies in der Spec dokumentiert sein.

-----

## 8. HTTP-Statuscodes

### 243 MUSS Nur offizielle HTTP-Statuscodes

Nur Statuscodes aus offiziellen RFCs verwenden. Keine proprietären Codes.

-----

### 151 MUSS Alle Statuscodes spezifizieren

Jeder Endpunkt muss alle möglichen Statuscodes mit Beispiel-Responses in OpenAPI dokumentiert haben.

-----

### 150 SOLLTE Nur gebräuchliche Statuscodes verwenden

|Code                       |Bedeutung            |Verwendung             |
|---------------------------|---------------------|-----------------------|
|`200 OK`                   |Erfolg               |GET, PUT, PATCH        |
|`201 Created`              |Erstellt             |POST (neue Ressource)  |
|`202 Accepted`             |Angenommen           |Asynchrone Verarbeitung|
|`204 No Content`           |Kein Inhalt          |DELETE, PUT ohne Body  |
|`207 Multi-Status`         |Teilerfolg           |Batch-Operationen      |
|`400 Bad Request`          |Ungültige Anfrage    |Syntaxfehler           |
|`401 Unauthorized`         |Nicht authentifiziert|Kein/ungültiger Token  |
|`403 Forbidden`            |Keine Berechtigung   |Fehlende Scopes        |
|`404 Not Found`            |Nicht gefunden       |Unbekannte Ressource   |
|`409 Conflict`             |Konflikt             |Optimistic Locking     |
|`410 Gone`                 |Dauerhaft entfernt   |Gelöschte Ressource    |
|`422 Unprocessable Entity` |Semantischer Fehler  |Validierungsfehler     |
|`429 Too Many Requests`    |Rate Limit           |Mit `Retry-After`      |
|`500 Internal Server Error`|Serverfehler         |                       |
|`503 Service Unavailable`  |Nicht verfügbar      |Wartung / Überlast     |

-----

### 220 MUSS Spezifischsten Statuscode verwenden

`422 Unprocessable Entity` statt generischem `400 Bad Request`, wenn die Syntax korrekt aber die Semantik fehlerhaft ist.

-----

### 152 MUSS Code 207 für Batch/Bulk-Requests

Batch-Endpunkte verwenden das `/batch`-Suffix — als standardisierte Ausnahme zu #141 (Verb-freie URLs). Siehe #141 für die Begründung.

```json
POST /v1/orders/batch → 207 Multi-Status
{
  "items": [
    { "id": "1", "status": 201, "order": {...} },
    { "id": "2", "status": 422, "problem": {...} }
  ]
}
```

-----

### 153 MUSS Code 429 mit Retry-After bei Rate Limits

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

-----

### 176 MUSS Problem JSON für alle Fehler (RFC 7807)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Das Feld 'quantity' muss grösser als 0 sein.",
  "instance": "/v1/orders/abc123"
}
```

|Feld      |Pflicht|Beschreibung                                                   |
|----------|-------|---------------------------------------------------------------|
|`type`    |✓      |URI des Fehlertyps                                             |
|`title`   |✓      |Kurze, menschenlesbare Fehlerbeschreibung                      |
|`status`  |✓      |HTTP-Statuscode als Zahl                                       |
|`detail`  |✗      |Detaillierte Fehlerbeschreibung                                |
|`instance`|✗      |URI der betroffenen Ressource                                  |
|`trace_id`|Nur 5xx|Aus `traceparent` extrahiert — für Log-Korrelation. Siehe C-05.|

-----

### 177 MUSS Keine Stack Traces in Fehler-Responses

Stack Traces, Datenbankfehler oder interne Pfade dürfen niemals in Fehler-Responses erscheinen.

-----

### 251 SOLLTE Keine Weiterleitungs-Codes

`301`, `302`, `307`, `308` nach Möglichkeit vermeiden. Korrekte URLs direkt zurückgeben.

-----

## 9. HTTP-Header

### 178 MUSS `Content-*` Header korrekt verwenden

```http
Content-Type: application/json
Content-Type: application/problem+json
Content-Encoding: gzip
```

-----

### ✦ C-03 / C-04 · MUSS · W3C Trace Context (ersetzt #233 X-Flow-ID)

**Jeder Service MUSS den `traceparent`-Header propagieren:**

|Header       |Level   |Format                                          |
|-------------|--------|------------------------------------------------|
|`traceparent`|**MUSS**|`00-{32hex traceId}-{16hex spanId}-{8bit flags}`|
|`tracestate` |SOLLTE  |`vendor=value,other=data`                       |
|`baggage`    |KANN    |W3C Baggage für Kontext-Weitergabe              |

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate:  company=backend-service
```

**Verhalten im Gateway (Gravitee):**

1. Eingehender Request **mit** `traceparent` → propagieren, nicht überschreiben
1. Eingehender Request **ohne** `traceparent` → neuen Trace generieren
1. `trace_id` und `span_id` in Access Logs schreiben
1. Bei 5xx-Fehler: `trace_id` in Problem JSON einbauen (siehe C-05)

**Warum W3C statt X-Flow-ID?** Offener Standard, unterstützt von Jaeger, Zipkin, Azure Monitor, OpenTelemetry Collector und allen modernen Observability-Plattformen.

-----

### C-05 SOLLTE `trace_id` in 5xx Problem JSON Responses

```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

Die `trace_id` wird aus dem `traceparent`-Header extrahiert (die 32-stellige Hex-ID nach `00-`). Dies ermöglicht direkte Log-Korrelation beim Debugging.

-----

### 132 SOLLTE kebab-case mit Grossbuchstaben für eigene HTTP-Header

```
Content-Type        ✓ Standard
traceparent         ✓ W3C Standard (Kleinbuchstaben korrekt)
X-Request-Id        ✓ Eigene Header in Title-Case
```

-----

### 180 SOLLTE `Location` Header nach POST

```http
HTTP/1.1 201 Created
Location: /v1/orders/abc123
```

-----

### 182 KANN ETag mit If-Match / If-None-Match

Der ETag-Wert wird im Response-Body als `etag`-Feld mitgeliefert — siehe #174 für den Standard-Feldnamen. Im HTTP-Header wird er als `ETag` gesendet.

```http
GET /v1/orders/123 → ETag: "abc123def456"

PUT /v1/orders/123
If-Match: "abc123def456"   # Schlägt fehl wenn zwischenzeitlich geändert
→ 409 Conflict
```

-----

### 230 KANN Idempotency-Key Header

```http
POST /v1/orders
Idempotency-Key: 7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b
```

-----

### 181 KANN Prefer Header

```http
Prefer: return=minimal          # Nur Statuscode, kein Body
Prefer: return=representation   # Vollständige Ressource zurück
Prefer: respond-async           # Asynchrone Verarbeitung
```

-----

## 10. Performance

### 155 SOLLTE Bandbreite reduzieren und Antwortzeiten verbessern

Übergeordnetes Performance-Prinzip: Kombination aus Komprimierung (#156), Feldauswahl (#157), optionalem Einbetten (#158) und Caching (#227) verwenden, um unnötige Datenübertragung zu vermeiden. Die nachfolgenden Regeln sind konkrete Umsetzungen dieses Prinzips.

-----

### 227 MUSS Cacheable Endpunkte dokumentieren

```http
Cache-Control: max-age=3600, must-revalidate
Cache-Control: no-cache
```

-----

### 156 SOLLTE gzip-Komprimierung unterstützen

```http
Accept-Encoding: gzip       # Request
Content-Encoding: gzip      # Response
```

-----

### 157 SOLLTE Partial Responses via Feldauswahl

```
GET /v1/orders?fields=id,status,created_at
```

-----

### 158 SOLLTE Einbetten von Sub-Ressourcen erlauben

```
GET /v1/orders/123?embed=items
→ { "id": "123", "items": [...] }
```

-----

## 11. Paginierung

### 159 MUSS Paginierung für alle Collection-Ressourcen

Jeder Endpunkt, der eine Liste zurückgibt, muss Paginierung unterstützen. Keine unlimitierten Responses.

→ Siehe #160 für die empfohlene Cursor-basierte Methode und #248 für das Standard-Response-Format.

-----

### 160 SOLLTE Cursor-basierte Paginierung bevorzugen

Cursor-basierte Paginierung ist stabiler als Offset (keine doppelten/fehlenden Einträge bei gleichzeitigen Änderungen). Gilt für alle Endpunkte die #159 erfüllen müssen. Das konkrete Response-Format ist in #248 definiert.

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

-----

### 248 SOLLTE Standard Pagination Response Object

Dieses Format gilt für alle Endpunkte die #159 (Paginierung MUSS) erfüllen und #160 (Cursor bevorzugen) umsetzen:

```json
{
  "items": [...],
  "cursor": {
    "next": "eyJpZCI6ImFiYzEyMyJ9",   // null wenn letzte Seite
    "prev": null                        // null wenn erste Seite
  }
}
```

-----

### 254 SOLLTE Gesamtanzahl vermeiden

`total_count` vermeiden — teuer bei grossen Datensätzen. Cursor für Navigation verwenden.

-----

## 12. Kompatibilität und Erweiterbarkeit

### 106 MUSS Keine Breaking Changes

Bestehende API-Konsumenten dürfen nicht ohne Abstimmung brechen. Breaking Changes erfordern eine neue Major Version.

**Was ist ein Breaking Change?**

|Änderung                                             |Breaking?                          |
|-----------------------------------------------------|-----------------------------------|
|Pflichtfeld in Request hinzufügen                    |✓ Breaking                         |
|Feld entfernen oder umbenennen                       |✓ Breaking                         |
|Ressource umbenennen (`/orders` → `/purchase-orders`)|✓ Breaking                         |
|Typ ändern (`string` → `integer`)                    |✓ Breaking                         |
|Bedeutung eines Feldes ändern (ohne Umbenennung)     |✓ Breaking                         |
|Endpunkt entfernen                                   |✓ Breaking                         |
|Statuscode ändern                                    |✓ Breaking                         |
|Optionales Feld hinzufügen                           |✗ Kompatibel                       |
|Neuen Endpunkt hinzufügen                            |✗ Kompatibel                       |
|Enum-Wert hinzufügen                                 |✗ Kompatibel (wenn Client tolerant)|

-----

### C-10 MUSS Vier Erweiterungsregeln einhalten

Jede Änderung an einer bestehenden API MUSS diese vier Regeln einhalten:

1. **Du DARFST NICHT etwas wegnehmen** — keine Properties, Endpunkte oder Enum-Werte entfernen
1. **Du DARFST NICHT Processing Rules ändern** — Semantik von Feldern bleibt stabil
1. **Du DARFST NICHT Optionales zu Pflicht machen** — existing clients würden brechen
1. **Alles was du hinzufügst MUSS optional sein** — neue Felder niemals required

> Diese Regeln gelten auch für Umbenennungen und URI-Änderungen. Namen und IDs sollen über die Zeit stabil bleiben — inklusive ihrer Semantik.

-----

### 108 MUSS Clients auf Erweiterungen vorbereiten

Clients MÜSSEN das Tolerant Reader Pattern implementieren: unbekannte Properties, neue Enum-Werte und zusätzliche Header ignorieren statt mit Fehler ablehnen. Die vollständige Definition und Begründung ist in C-09 (Postel’s Law) beschrieben.

-----

### 110 MUSS JSON-Objekte als Top-Level-Datenstruktur

```json
// ✓ Richtig
{ "items": [1, 2, 3], "cursor": {...} }

// ✗ Falsch — Array direkt als Top-Level
[1, 2, 3]
```

**Warum?** Ermöglicht späteres Hinzufügen von Metadaten ohne Breaking Change.

-----

### 111 MUSS OpenAPI Spec als erweiterbar behandeln

Die OpenAPI-Spezifikation MUSS so gestaltet sein, dass sie jederzeit um optionale Properties, neue Endpunkte und neue Enum-Werte erweitert werden kann — ohne dass bestehende Konsumenten brechen. Das ist die spec-seitige Umsetzung von C-10 Regel 4 (“Alles was du hinzufügst MUSS optional sein”).

-----

### 107 SOLLTE Kompatible Erweiterungen bevorzugen

Neue Funktionalität als optionale Erweiterungen hinzufügen, die bestehende Konsumenten ignorieren können. Wenn es um konkrete Felder in der API geht, wird dieses SOLLTE durch C-10 Regel 4 zum MUSS: “Alles was du hinzufügst MUSS optional sein.”

-----

### 109 SOLLTE APIs konservativ designen

APIs SOLLTEN so wenig wie nötig exponieren — unnötige Flexibilität erhöht Komplexität und Wartungsaufwand. Die konkrete Umsetzung dieses Prinzips ist in C-08 (Minimale API-Oberfläche / YAGNI) definiert.

-----

### 112 SOLLTE Offene Enum-Listen verwenden

```yaml
status:
  type: string
  x-extensible-enum:
    - OPEN
    - COMPLETED
    - CANCELLED
  description: Neue Werte können hinzugefügt werden. Clients müssen unbekannte Werte tolerieren.
```

-----

## 13. Deprecation

### 187 MUSS Deprecation in API-Spezifikation markieren

```yaml
paths:
  /v1/legacy-orders:
    get:
      deprecated: true
      description: |
        **Deprecated** — Bitte auf /v2/orders migrieren.
        Sunset-Datum: 2025-06-30
```

-----

### 185 MUSS Konsumenten vor API-Abschaltung informieren und Migrationsfrist gewähren

Die ursprüngliche Formulierung “alle Konsumenten müssen zustimmen” ist bewusst präzisiert worden: ein absolutes Zustimmungsrecht würde bedeuten dass ein einzelner nicht-reagierender Konsument eine API-Abschaltung auf unbestimmte Zeit blockieren kann. Das ist weder praktikabel noch beabsichtigt.

**Der Kern der Regel ist:** Kein Konsument darf unvorbereitet von einer API-Abschaltung getroffen werden.

#### Der Deprecation-Prozess (6 Schritte)

```
0. SPEC MARKIEREN  deprecated: true + Sunset-Datum in OpenAPI setzen (#187)
        ↓
1. HEADER SETZEN   Deprecation + Sunset Header in Responses aktivieren (#189)
        ↓
2. ANKÜNDIGUNG     Alle bekannten Konsumenten aktiv informieren
        ↓
3. MIGRATIONSFRIST Angemessene Frist nach Zielgruppe einräumen
        ↓
4. MONITORING      Tatsächliche Nutzung überwachen (#188)
        ↓
5. ESKALATION      Nicht-migrierte Konsumenten nach 80% der Frist aktiv ansprechen
        ↓
6. ABSCHALTUNG     Nach Fristablauf — auch ohne explizite Zustimmung
```

**Was “Zustimmung” bedeutet:**
Zustimmung bedeutet nicht dass ein Konsument aktiv “Ja” sagen muss. Es bedeutet: der Konsument hat die Ankündigung nachweislich erhalten und die Migrationsfrist ist bekannt. Wer nach Fristablauf nicht migriert hat, trägt selbst die Verantwortung.

#### Mindest-Migrationsfristen nach Audience

|Audience                |Mindestfrist|Begründung                                   |
|------------------------|------------|---------------------------------------------|
|`component-internal`    |2 Wochen    |Dasselbe Team, kurze Abstimmung möglich      |
|`business-unit-internal`|4 Wochen    |Interne Teams, Planungszyklen berücksichtigen|
|`company-internal`      |3 Monate    |Verschiedene Teams, Budgetplanung nötig      |
|`external-partner`      |6 Monate    |Externe Verträge, Release-Zyklen der Partner |
|`external-public`       |12 Monate   |Unbekannte Konsumenten, maximale Fairness    |

#### Eskalationsstufen bei Nicht-Reaktion

```
Frist zu 0%  → Ankündigung per E-Mail / Ticket an alle bekannten Konsumenten
Frist zu 50% → Erinnerung — wer noch nicht migriert hat wird direkt kontaktiert
Frist zu 80% → Eskalation an Team-Lead / Management der betroffenen Konsumenten
Frist zu 100%→ Abschaltung — unabhängig vom Migrationsstand einzelner Konsumenten
```

#### Nachweis-Pflicht

Bevor die API abgeschaltet wird, müssen folgende Nachweise vorliegen:

```
✓ Ankündigung dokumentiert (Datum, Kanal, Empfänger)
✓ Sunset-Datum in API-Spec (#187) und Response-Header (#189) gesetzt
✓ Monitoring zeigt: Nutzung geht gegen null (#188)
✓ Eskalation für aktive Konsumenten dokumentiert
```

-----

### 186 MUSS Externe Partner bei Deprecation besonders behandeln

Externe Partner (`external-partner`, `external-public`) haben andere Rahmenbedingungen als interne Teams: externe Verträge, eigene Release-Zyklen, ggf. regulatorische Anforderungen. Für sie gilt:

- Mindestfrist 6 Monate (`external-partner`) bzw. 12 Monate (`external-public`) — nicht verhandelbar
- Ankündigung zusätzlich über offizielle Kanäle (Developer Portal, Newsletter, Changelog)
- Bei `external-public`: öffentliche Ankündigung reicht — direkte Zustimmung jedes Konsumenten ist nicht erforderlich und nicht möglich

**Sonderfall unbekannte Konsumenten (`external-public`):**

Bei öffentlichen APIs gibt es per Definition Konsumenten die nicht direkt erreichbar sind:

```
✓ Öffentliche Ankündigung (Blog, Changelog, Developer Portal)
✓ Deprecation + Sunset Header in allen Responses (#189)
✓ Sunset-Datum mindestens 12 Monate in der Zukunft
✓ Neue Registrierungen ab Ankündigung mit Warnung versehen
✓ Monitoring auf verbleibende aktive Aufrufe (#188)
```

-----

### 188 MUSS Nutzung der deprecated API monitoren

Tatsächliche Nutzung messen, um sicherzustellen dass alle Konsumenten migriert haben.

-----

### 191 MUSS Keine deprecated APIs neu verwenden

Neue Services dürfen keine als deprecated markierten Endpunkte verwenden.

-----

### 189 SOLLTE Deprecation und Sunset Header

```http
Deprecation: true
Sunset: Sat, 30 Jun 2025 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

-----

### 190 SOLLTE Monitoring für Deprecation und Sunset

Konkretisierung von #188: Alerts konfigurieren wenn das Sunset-Datum innerhalb von 30 Tagen liegt und noch aktive Konsumenten vorhanden sind. Empfohlene Alert-Schwellen: 90 Tage vor Sunset (Info), 30 Tage (Warning), 7 Tage (Critical).

-----

## 14. Betrieb

### 192 MUSS OpenAPI-Spezifikation veröffentlichen

Die OpenAPI-Spezifikation muss zur Laufzeit vom Service erreichbar sein. Welche Endpunkte zu verwenden sind, wie die Erreichbarkeit nach Audience gesteuert wird und wie Gravitee die Spec importiert, ist vollständig in #101 beschrieben.

```
GET /openapi.yaml    # Spezifikation abrufbar
GET /docs            # Optional: Swagger UI
```

→ Siehe #101 für vollständige Erreichbarkeits- und Audience-Regeln.

-----

### 193 SOLLTE API-Nutzung monitoren

- Requests pro Endpunkt und Statuscode
- Latenz (p50, p95, p99)
- Fehlerrate
- Nutzung pro API-Konsument (für Deprecation-Monitoring)

-----

## Übersicht: Alle eigenen Regeln

|ID  |Level         |Regel                                                        |Quelle|Ersetzt               |
|----|--------------|-------------------------------------------------------------|------|----------------------|
|C-01|**MUSS**      |URL-Versionierung: `/{version}/{resource}`                   |Eigene|#113, #114, #115      |
|C-02|**MUSS NICHT**|Kein HATEOAS — kein `_links`, `href`, `self`                 |Eigene|#163, #164, #165, #161|
|C-03|**MUSS**      |`traceparent` (W3C Trace Context) propagieren                |Eigene|#233                  |
|C-04|SOLLTE        |`tracestate` propagieren falls vorhanden                     |Eigene|#233                  |
|C-05|SOLLTE        |`trace_id` in Problem JSON 5xx Responses                     |Eigene|neu                   |
|C-06|**MUSS**      |Scope-Format: `read:<resource>`, `write:<resource>`          |Eigene|#225                  |
|C-07|SOLLTE        |`metadata`-Feld für erweiterbare Ressourcen                  |Stripe|neu                   |
|C-08|**MUSS**      |Minimale API-Oberfläche (YAGNI-Prinzip)                      |Adidas|ergänzt #109          |
|C-09|**MUSS**      |Postel’s Law für Server und Client                           |Adidas|ergänzt #108          |
|C-10|**MUSS**      |Vier Erweiterungsregeln (kein Wegnehmen, keine Pflichtfelder)|Adidas|schärft #106          |
|C-11|SOLLTE        |`description` vs. `metadata` klar trennen                    |Stripe|neu                   |
|C-12|**MUSS**      |API-Specs in Git mit CHANGELOG und Tags                      |Adidas|ergänzt #101          |

-----

## Entfernte Zalando-interne Regeln

|#ID |Grund                                                            |
|----|-----------------------------------------------------------------|
|#234|Verweist auf `zalandoapis.com` interne GitHub URLs               |
|#223|Functional Naming basiert auf Zalando-internem Component Registry|
|#224|Hostname-Convention für `.zalandoapis.com` / `.zalan.do` Domains |
|#183|Explizit Zalando-spezifische proprietäre Header-Liste            |
|#173|Zalando Money Object mit `jackson-datatype-money`                |
|#249|Zalando-spezifisches Adressformat                                |

-----

*Version 2.0 — Basiert auf Zalando, Adidas und Stripe API Guidelines*  
*Quercheck: [Stripe API](https://docs.stripe.com/api) · [Adidas Guidelines](https://adidas.gitbook.io/api-guidelines)*


# Pagination — Leitfaden für Entwickler

> Basierend auf den Regeln #159, #160, #248, #254, #110, #130, #137, #153 und #176 des REST API Styleguides.

-----

## Warum Pagination?

Ein Order-Service mit 2 Millionen Bestellungen wird ohne Pagination zum Problem: Bei einem Aufruf von `GET /v1/orders` würde versucht, alle 2 Millionen Datensätze in einer einzigen Response zurückzugeben. Der Request läuft in ein Timeout, der Server bricht unter der Last zusammen, und der aufrufende Dienst wartet minutenlang auf eine Antwort die nie eintrifft.

Durch Pagination werden grosse Datenmengen in handhabbare Seiten aufgeteilt. Deshalb gilt nach Regel **#159**: **Jeder Endpunkt der eine Liste zurückgibt, MUSS Pagination unterstützen — ohne Ausnahme.** Kein unlimitierter Response ist erlaubt, unabhängig davon wie klein der Datensatz heute noch ist.

-----

## Cursor vs. Offset — warum Cursor bevorzugt wird

Es gibt zwei grundlegende Ansätze für Pagination. Offset-Pagination ist aus SQL bekannt:

```
GET /v1/orders?offset=40&limit=20   ← Offset-Pagination
```

Das wirkt intuitiv — “20 Einträge ab Position 40”. Das Problem zeigt sich erst im Betrieb: Während die zweite Seite abgerufen wird, wird von einem anderen User eine neue Bestellung angelegt. Die neue Bestellung verschiebt alle nachfolgenden Einträge um eine Position. Ein Eintrag wird dadurch doppelt zurückgegeben oder übersprungen. Bei hohem Durchsatz tritt dieses Problem ständig auf.

Cursor-Pagination löst dieses Problem. Statt einer absoluten Position wird ein undurchsichtiger Zeiger auf den letzten gesehenen Eintrag verwendet:

```
GET /v1/orders?cursor=eyJpZCI6ImFiYzEyMyJ9&limit=20   ← Cursor-Pagination
```

Im Cursor wird intern der Ankerpunkt im Datensatz kodiert — beispielsweise die ID des letzten zurückgegebenen Eintrags. Neue Einträge die während der Navigation hinzukommen, verschieben keine relativen Positionen. Deshalb empfiehlt Regel **#160**: Cursor-basierte Pagination bevorzugen.

|                  |Offset                            |Cursor                  |
|------------------|----------------------------------|------------------------|
|Bekannte Position |✓ direkt berechenbar              |✗ nicht möglich         |
|Stabile Navigation|✗ Duplikate/Lücken möglich        |✓ immer konsistent      |
|Sprung zu Seite N |✓ trivial                         |✗ nicht unterstützt     |
|Geeignet für      |Admin-UIs mit Seitennavigation    |Feeds, Listen, APIs     |
|Performance       |✗ OFFSET wird langsamer je grösser|✓ gleichbleibend schnell|

Für die meisten API-Anwendungsfälle — Feeds, Listen, Exports — ist die stabile Navigation entscheidend. Das direkte Springen zu Seite N wird selten benötigt; in diesen Fällen empfiehlt sich eine Suchfunktion statt Pagination.

-----

## Das Standard-Response-Format (#248)

Nach Regel **#110** ist das Top-Level einer Response immer ein JSON-Objekt — niemals direkt ein Array. Dadurch können Pagination-Metadaten neben den eigentlichen Daten platziert werden, ohne später einen Breaking Change zu riskieren.

Das vorgeschriebene Format nach **#248** lautet:

```json
{
  "items": [
    {
      "id": "ord_abc123",
      "status": "OPEN",
      "total_amount": 149.95,
      "created_at": "2024-01-15T10:30:00Z"
    },
    {
      "id": "ord_def456",
      "status": "IN_PROGRESS",
      "total_amount": 89.00,
      "created_at": "2024-01-14T08:15:00Z"
    }
  ],
  "cursor": {
    "next": "eyJpZCI6Im9yZF9kZWY0NTYiLCJjcmVhdGVkX2F0IjoiMjAyNC0wMS0xNCJ9",
    "prev": null
  }
}
```

Zwei Felder sind immer vorhanden:

**`items`** enthält das Array der aktuellen Seite. Der Name ist bewusst generisch gewählt — er ist für alle Collection-Endpunkte gleich, was die Client-Implementierung vereinfacht. Nach Regel #120 werden Array-Namen immer im Plural geschrieben.

**`cursor`** enthält zwei Zeiger. `next` verweist auf die nächste Seite und ist `null` wenn die letzte Seite erreicht wurde. `prev` verweist auf die vorherige Seite und ist `null` wenn die erste Seite zurückgegeben wurde. Beide Felder sind opaque Strings — der Inhalt darf vom aufrufenden Dienst weder dekodiert noch konstruiert werden.

-----

## Der Cursor im Detail

Der Cursor ist für den aufrufenden Dienst ein schwarzes Brett: Er wird gespeichert und unverändert zurückgesendet — mehr nicht. Was serverseitig darin kodiert ist, bleibt eine Implementierungsentscheidung:

```
eyJpZCI6Im9yZF9kZWY0NTYiLCJjcmVhdGVkX2F0IjoiMjAyNC0wMS0xNCJ9
```

Dekodiert (Base64) ergibt das:

```json
{ "id": "ord_def456", "created_at": "2024-01-14" }
```

Diese Information wird serverseitig verwendet, um die nächste Seite ab genau diesem Punkt zu laden. Die interne Cursor-Struktur kann jederzeit geändert werden — ohne dass der aufrufende Dienst angepasst werden muss, solange der Cursor unverändert zurückgesendet wird.

Ein Cursor ist zeitlich begrenzt und keine permanente Ressource. Nach typischerweise 24 Stunden ist ein Cursor ungültig. Wird ein abgelaufener Cursor verwendet, antwortet der Server mit `400 Bad Request` und Problem JSON nach Regel **#176**:

```json
{
  "type": "https://api.example.com/errors/invalid-cursor",
  "title": "Invalid or Expired Cursor",
  "status": 400,
  "detail": "The provided cursor is invalid or has expired. Please start a new pagination from the beginning."
}
```

-----

## Query-Parameter für Pagination (#130, #137)

Nach Regel **#130** werden alle Query-Parameter in snake_case geschrieben. Regel **#137** definiert die Standard-Namen die für Pagination zu verwenden sind:

```
GET /v1/orders?limit=20&cursor=eyJpZCI6...
```

`limit` gibt die maximale Anzahl Einträge pro Seite an. `cursor` enthält den Wert aus der vorherigen Response. Beide Parameter sind optional — fehlen sie, wird eine sinnvolle Standardanzahl (typischerweise 20) ab dem Anfang der Liste zurückgegeben.

Filterparameter können mit Cursor und Limit kombiniert werden:

```
GET /v1/orders?status=OPEN&sort=-created_at&limit=20&cursor=eyJpZCI6...
```

Der Cursor muss immer konsistent mit den übrigen Parametern verwendet werden. Ein Cursor der mit `?status=OPEN` erzeugt wurde, darf nicht mit `?status=CANCELLED` kombiniert werden. Der Server validiert dies und gibt andernfalls `400 Bad Request` zurück.

-----

## Vollständiges Navigationsbeispiel

Alle offenen Bestellungen sollen durchlaufen werden. Die Navigation läuft wie folgt ab:

**Erster Request — kein Cursor, Anfang der Liste:**

```
GET /v1/orders?status=OPEN&sort=-created_at&limit=3
Authorization: Bearer eyJhbG...
```

```json
{
  "items": [
    { "id": "ord_001", "status": "OPEN", "created_at": "2024-01-15T10:30:00Z" },
    { "id": "ord_002", "status": "OPEN", "created_at": "2024-01-14T09:00:00Z" },
    { "id": "ord_003", "status": "OPEN", "created_at": "2024-01-13T14:20:00Z" }
  ],
  "cursor": {
    "next": "eyJpZCI6Im9yZF8wMDMifQ",
    "prev": null
  }
}
```

`prev` ist `null` — es handelt sich um die erste Seite. `next` enthält einen Wert — es gibt weitere Einträge.

**Zweiter Request — `next`-Cursor aus vorheriger Response:**

```
GET /v1/orders?status=OPEN&sort=-created_at&limit=3&cursor=eyJpZCI6Im9yZF8wMDMifQ
```

```json
{
  "items": [
    { "id": "ord_004", "status": "OPEN", "created_at": "2024-01-12T11:00:00Z" },
    { "id": "ord_005", "status": "OPEN", "created_at": "2024-01-11T08:30:00Z" }
  ],
  "cursor": {
    "next": null,
    "prev": "eyJpZCI6Im9yZF8wMDQiLCJkaXJlY3Rpb24iOiJwcmV2In0"
  }
}
```

`next` ist `null` — die letzte Seite wurde erreicht, es gibt keine weiteren Einträge. `prev` enthält einen Wert — Rückwärtsnavigation ist möglich.

**Navigation rückwärts — `prev`-Cursor:**

```
GET /v1/orders?status=OPEN&sort=-created_at&limit=3&cursor=eyJpZCI6Im9yZF8wMDQiLCJkaXJlY3Rpb24iOiJwcmV2In0
```

Als Antwort wird die erste Seite zurückgegeben.

-----

## Gesamtanzahl — warum sie vermieden wird (#254)

In der Response eine `total_count` mitzuliefern wirkt praktisch:

```json
// ✗ Vermeiden
{
  "items": [...],
  "total_count": 284710,
  "cursor": { "next": "...", "prev": null }
}
```

Das Problem: `SELECT COUNT(*)` über 2 Millionen Zeilen mit komplexen Filtern ist teuer. Bei jeder Pagination-Anfrage muss eine vollständige Count-Query ausgeführt werden — auch wenn die letzte Seite längst erreicht wurde und keine weiteren Einträge folgen.

Regel **#254** empfiehlt deshalb: Gesamtanzahl vermeiden. `cursor.next = null` signalisiert das Ende der Liste. Das reicht für die grosse Mehrheit der Anwendungsfälle.

Wird eine Gesamtanzahl tatsächlich benötigt — etwa für eine UI die “284.710 Ergebnisse” anzeigen soll — stehen bessere Alternativen zur Verfügung: eine approximierte Anzahl aus Datenbankstatistiken, ein separater Count-Endpunkt der nur bei Bedarf aufgerufen wird, oder ein gecachter Count der periodisch aktualisiert wird.

-----

## OpenAPI-Spezifikation

Pagination muss vollständig in der OpenAPI-Spezifikation dokumentiert sein. Nach Regel #151 muss jeder Endpunkt alle Statuscodes und sein Response-Schema spezifizieren.

```yaml
paths:
  /v1/orders:
    get:
      summary: List orders
      parameters:
        - name: limit
          in: query
          required: false
          schema:
            type: integer
            format: int32
            minimum: 1
            maximum: 100
            default: 20
          description: |
            Maximum number of items to return per page.
            Defaults to 20 if not specified.

        - name: cursor
          in: query
          required: false
          schema:
            type: string
          description: |
            Opaque cursor for pagination, obtained from a previous response.
            Do not construct or decode this value — treat it as an opaque string.
            Cursors expire after 24 hours.

        - name: sort
          in: query
          required: false
          style: form
          explode: false
          schema:
            type: array
            items:
              type: string
          description: |
            Sort order as comma-separated fields.
            Prefix with + for ascending (default), - for descending.
            Example: ?sort=-created_at,+status

      responses:
        '200':
          description: A page of orders
          content:
            application/json:
              schema:
                type: object
                required: [items, cursor]
                properties:
                  items:
                    type: array
                    items:
                      $ref: '#/components/schemas/Order'
                  cursor:
                    type: object
                    required: [next, prev]
                    properties:
                      next:
                        type: string
                        nullable: true
                        description: Cursor for the next page. null if this is the last page.
                        example: "eyJpZCI6Im9yZF9kZWY0NTYifQ"
                      prev:
                        type: string
                        nullable: true
                        description: Cursor for the previous page. null if this is the first page.
              example:
                items:
                  - id: "ord_abc123"
                    status: "OPEN"
                    created_at: "2024-01-15T10:30:00Z"
                cursor:
                  next: "eyJpZCI6Im9yZF9hYmMxMjMifQ"
                  prev: null

        '400':
          description: Invalid cursor or parameters
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
              example:
                type: "https://api.example.com/errors/invalid-cursor"
                title: "Invalid or Expired Cursor"
                status: 400
                detail: "The provided cursor is invalid or has expired."
```

-----

## Fehlerbehandlung

Drei Fehlerfälle sind bei Pagination relevant, alle nach Regel **#176** als Problem JSON zurückzugeben:

**Ungültiger oder abgelaufener Cursor:**

```json
{
  "type": "https://api.example.com/errors/invalid-cursor",
  "title": "Invalid or Expired Cursor",
  "status": 400,
  "detail": "The provided cursor is invalid or has expired. Please restart pagination."
}
```

**`limit` ausserhalb des erlaubten Bereichs:**

```json
{
  "type": "https://api.example.com/errors/invalid-parameter",
  "title": "Invalid Parameter",
  "status": 422,
  "detail": "Parameter 'limit' must be between 1 and 100. Provided value: 500."
}
```

**Cursor nicht kompatibel mit aktuellen Filterparametern:**

```json
{
  "type": "https://api.example.com/errors/cursor-filter-mismatch",
  "title": "Cursor Filter Mismatch",
  "status": 400,
  "detail": "The cursor was created with different filter parameters. Please restart pagination with consistent parameters."
}
```

-----

## Pagination bei POST /search (#237)

Pagination gilt nicht nur für `GET`-Endpunkte. Auch der `/search`-Endpunkt für komplexe Filter gibt paginierte Ergebnisse zurück — mit demselben `cursor`-Format im Response-Body. Der Cursor wird im Request-Body mitgesendet:

```json
POST /v1/orders/search

{
  "filter": {
    "status": ["OPEN", "IN_PROGRESS"],
    "total_amount": { "gte": 100.00 }
  },
  "sort": ["-created_at"],
  "limit": 20,
  "cursor": "eyJpZCI6Im9yZF9kZWY0NTYifQ"
}
```

Der Response ist identisch mit dem `GET`-Endpunkt. Dadurch kann nahtlos zwischen `GET` mit Filtern und `POST /search` gewechselt werden, ohne das Pagination-Muster anzupassen.

-----

## Häufige Fehler

**Cursor konstruieren statt kopieren.** Der Cursor wird dekodiert und selbst aufgebaut — `eyJpZCI6Im9yZF8xMjMifQ` → `{"id":"ord_123"}` → `eyJpZCI6Im9yZF8xMjQifQ`. Das funktioniert zufällig, bricht aber sobald das interne Format serverseitig geändert wird. Der Cursor muss immer unverändert aus der letzten Response übernommen werden.

**`total_count` implementieren und cachen.** `total_count` wird hinzugefügt “weil der aufrufende Dienst es vielleicht braucht”. Nach 5 Minuten ist der Cache veraltet. Das ist schlechter als keine Anzahl. `cursor.next === null` als Ende-Signal ist zuverlässiger.

**Pagination vergessen wenn die Liste klein ist.** “Es gibt nur 50 Kategorien, da wird keine Pagination benötigt.” Ein Jahr später sind es 500. Pagination nachzurüsten ist dann ein Breaking Change — der aufrufende Dienst hat bisher immer alle Einträge auf einmal erhalten. Deshalb gilt #159 ohne Ausnahme, auch für kleine Listen.

**Limit ohne Obergrenze.** `?limit=999999` darf serverseitig nicht akzeptiert werden. Eine sinnvolle Obergrenze (typischerweise 100) muss definiert und bei Überschreitung mit `422` quittiert werden.

**`prev`-Cursor weglassen.** Manche Implementierungen verzichten auf Rückwärtsnavigation. Das ist zulässig wenn der Anwendungsfall es nicht erfordert — aber `prev` muss dann explizit `null` sein und darf nicht einfach weggelassen werden. Nach Regel #123 haben fehlendes Feld und `null` dieselbe Semantik — `prev: null` ist für den aufrufenden Dienst dennoch klarer als ein fehlendes Feld.

-----

## Zusammenfassung

|Regel|Kernaussage                                                                      |
|-----|---------------------------------------------------------------------------------|
|#159 |Jede Collection MUSS paginiert sein — keine unlimitierten Responses              |
|#160 |Cursor-Pagination bevorzugen — stabiler als Offset bei gleichzeitigen Änderungen |
|#248 |Standard-Format: `{ "items": [...], "cursor": { "next": "...", "prev": "..." } }`|
|#254 |Keine `total_count` — teuer und selten wirklich nötig                            |
|#110 |Top-Level immer JSON-Objekt — nie direkt ein Array                               |
|#130 |Query-Parameter in snake_case: `limit`, `cursor`                                 |
|#137 |Standard-Namen verwenden: `cursor`, `limit`, `sort`, `fields`                    |
|#176 |Fehler als Problem JSON: ungültiger Cursor → `400`, falsches Limit → `422`       |


# Fehlerbehandlung — Leitfaden für Entwickler

> Basierend auf den Regeln #243, #151, #150, #220, #152, #153, #176, #177, C-03, C-04 und C-05 des REST API Styleguides.

-----

## Warum einheitliche Fehlerbehandlung?

Ein API-Konsument ruft `POST /v1/orders` auf und erhält eine Fehlerantwort. Ohne einheitliches Format muss in der Dokumentation nachgeschlagen werden, ob das Fehlerfeld `message`, `error`, `errorMessage` oder `description` heisst. Ob der HTTP-Statuscode `400` oder `422` zurückgegeben wird. Ob eine Trace-ID vorhanden ist und wo sie zu finden ist.

Einheitliche Fehlerbehandlung löst dieses Problem: Jede Fehlerantwort folgt demselben Format, denselben Statuscodes und denselben Konventionen — unabhängig davon welcher Endpunkt betroffen ist und welches Team ihn implementiert hat. Fehler werden dadurch maschinell verarbeitbar, Monitoring einfacher und Debugging schneller.

-----

## Das Problem JSON Format (#176)

Alle Fehlerantworten verwenden **Problem JSON** nach RFC 7807 mit dem Medientyp `application/problem+json`. Das ist kein optionales Format — nach Regel **#176** gilt es für alle Fehlerfälle ohne Ausnahme.

Ein vollständiges Beispiel:

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "The field 'quantity' must be greater than 0.",
  "instance": "/v1/orders/ord_abc123"
}
```

Die fünf Felder im Überblick:

**`type`** (Pflicht) ist eine URI die den Fehlertyp eindeutig identifiziert. Sie verweist idealerweise auf eine Dokumentationsseite die den Fehler beschreibt und mögliche Lösungsschritte nennt. Ist keine Dokumentation verfügbar, wird `about:blank` verwendet.

**`title`** (Pflicht) ist eine kurze, menschenlesbare Beschreibung des Fehlertyps. Der Titel ist für alle Instanzen desselben Fehlertyps identisch — er beschreibt die Klasse des Fehlers, nicht den konkreten Fall.

**`status`** (Pflicht) ist der HTTP-Statuscode als Zahl. Er muss mit dem tatsächlichen HTTP-Statuscode der Response übereinstimmen.

**`detail`** (optional) beschreibt den konkreten Fehlerfall. Hier werden spezifische Informationen angegeben — welches Feld fehlerhaft ist, welcher Wert erwartet wird, was konkret schiefgelaufen ist. Im Gegensatz zu `title` darf `detail` für jede Instanz unterschiedlich sein.

**`instance`** (optional) ist eine URI die die betroffene Ressource identifiziert. Typischerweise der Pfad der angeforderten Ressource.

Für 5xx-Fehler kommt ein sechstes Feld hinzu:

**`trace_id`** (bei 5xx empfohlen) enthält die Trace-ID aus dem W3C `traceparent`-Header. Sie ermöglicht die direkte Korrelation zwischen der Fehlerantwort und den Serverlog-Einträgen — unverzichtbar für das Debugging in verteilten Systemen. Siehe C-05 und C-03/C-04.

```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

-----

## HTTP-Statuscodes (#243, #150, #220)

Nach Regel **#243** werden ausschliesslich offizielle HTTP-Statuscodes aus RFCs verwendet. Proprietäre Codes sind nicht zulässig.

Regel **#150** empfiehlt die Verwendung der gebräuchlichsten Statuscodes. Weniger bekannte Codes sind nur einzusetzen wenn sie den Sachverhalt deutlich präziser beschreiben als die gebräuchlichen Alternativen.

### Erfolg-Statuscodes

|Code              |Bedeutung                         |Wann verwenden                         |
|------------------|----------------------------------|---------------------------------------|
|`200 OK`          |Erfolgreich                       |`GET`, `PUT`, `PATCH` mit Response-Body|
|`201 Created`     |Ressource erstellt                |`POST` bei neuer Ressource             |
|`202 Accepted`    |Angenommen, noch nicht verarbeitet|Asynchrone Verarbeitung                |
|`204 No Content`  |Erfolgreich, kein Body            |`DELETE`, `PUT` ohne Response-Body     |
|`207 Multi-Status`|Teilerfolg                        |Batch-Operationen — siehe #152         |

### Fehler-Statuscodes

|Code                       |Bedeutung                    |Wann verwenden                                        |
|---------------------------|-----------------------------|------------------------------------------------------|
|`400 Bad Request`          |Syntaktisch ungültige Anfrage|Ungültiges JSON, fehlende Pflichtfelder               |
|`401 Unauthorized`         |Nicht authentifiziert        |Kein oder ungültiger Token                            |
|`403 Forbidden`            |Nicht autorisiert            |Gültiger Token, aber fehlende Berechtigung            |
|`404 Not Found`            |Ressource nicht gefunden     |Unbekannte ID oder Pfad                               |
|`409 Conflict`             |Konflikt                     |Optimistic Locking, Duplikat                          |
|`410 Gone`                 |Dauerhaft entfernt           |Ressource wurde gelöscht und existiert nicht mehr     |
|`422 Unprocessable Entity` |Semantisch ungültige Anfrage |Syntaktisch korrektes JSON, aber inhaltlich fehlerhaft|
|`429 Too Many Requests`    |Rate Limit überschritten     |Immer mit `Retry-After` Header — siehe #153           |
|`500 Internal Server Error`|Unerwarteter Serverfehler    |Technische Fehler auf Serverseite                     |
|`503 Service Unavailable`  |Dienst nicht verfügbar       |Wartung oder Überlast                                 |

### 400 vs. 422 — der wichtigste Unterschied (#220)

Regel **#220** schreibt vor, den spezifischsten Statuscode zu verwenden. Die Unterscheidung zwischen `400` und `422` ist dabei am häufigsten unklar:

**`400 Bad Request`** — die Anfrage ist syntaktisch nicht verarbeitbar:

```
POST /v1/orders
Content-Type: application/json

{ "quantity": "zehn", "price": }    ← Ungültiges JSON
```

```json
{
  "type": "https://api.example.com/errors/invalid-json",
  "title": "Invalid JSON",
  "status": 400,
  "detail": "Unexpected token at position 32."
}
```

**`422 Unprocessable Entity`** — die Anfrage ist syntaktisch korrekt, aber semantisch fehlerhaft:

```json
POST /v1/orders
Content-Type: application/json

{
  "quantity": -5,
  "delivery_date": "2020-01-01"
}
```

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Field 'quantity' must be greater than 0. Field 'delivery_date' must be in the future.",
  "instance": "/v1/orders"
}
```

Die Faustregel: Kann der JSON-Parser die Anfrage nicht verarbeiten → `400`. Kann der Parser sie verarbeiten, aber die Geschäftslogik lehnt sie ab → `422`.

-----

## Validierungsfehler mit mehreren Feldern

Bei `422`-Fehlern mit mehreren ungültigen Feldern werden alle Fehler in einer einzigen Response zurückgegeben — nicht nacheinander. RFC 7807 erlaubt eigene Erweiterungsfelder im Problem JSON:

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Multiple validation errors occurred.",
  "instance": "/v1/orders",
  "errors": [
    {
      "field": "quantity",
      "message": "Must be greater than 0.",
      "rejected_value": -5
    },
    {
      "field": "delivery_date",
      "message": "Must be a future date.",
      "rejected_value": "2020-01-01"
    }
  ]
}
```

Das `errors`-Array ist ein Erweiterungsfeld das über den RFC-7807-Standard hinausgeht — es ist aber zulässig da RFC 7807 eigene Felder ausdrücklich erlaubt. Es wird im OpenAPI-Schema für den `422`-Statuscode definiert.

-----

## Rate Limiting (#153)

Wird das Rate Limit eines Endpunkts überschritten, wird `429 Too Many Requests` zurückgegeben — immer mit dem `Retry-After`-Header. Der Header gibt in Sekunden an, nach welcher Wartezeit eine erneute Anfrage gestellt werden kann:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "The rate limit of 1000 requests per minute has been exceeded.",
  "instance": "/v1/orders"
}
```

Der `Retry-After`-Header ist für automatisierte Clients unverzichtbar: Ohne ihn müssten aufrufende Dienste mit fixen Wartezeiten oder exponentiellem Backoff arbeiten, was zu unnötiger Latenz oder erneuten Überschreitungen führt. Mit dem Header kann die Wartezeit präzise eingehalten werden.

-----

## Batch-Fehler (#152)

Bei Batch-Operationen (`POST /v1/orders/batch`) kann ein Teil der Einträge erfolgreich sein, während andere fehlschlagen. In diesem Fall wird **nicht** `400` oder `500` zurückgegeben — stattdessen `207 Multi-Status` mit dem Ergebnis jedes einzelnen Eintrags:

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "items": [
    {
      "id": "req_1",
      "status": 201,
      "order": {
        "id": "ord_abc123",
        "status": "OPEN"
      }
    },
    {
      "id": "req_2",
      "status": 422,
      "problem": {
        "type": "https://api.example.com/errors/validation-error",
        "title": "Validation Error",
        "status": 422,
        "detail": "Field 'quantity' must be greater than 0."
      }
    },
    {
      "id": "req_3",
      "status": 201,
      "order": {
        "id": "ord_def456",
        "status": "OPEN"
      }
    }
  ]
}
```

Jeder Eintrag enthält einen eigenen `status`-Code und entweder das Ergebnisobjekt oder ein eingebettetes Problem JSON. Der aufrufende Dienst iteriert über die Einträge und behandelt Fehler auf Eintragsebene — nicht auf Response-Ebene. Ein `207` ist kein Fehler auf HTTP-Ebene; der Request selbst war erfolgreich, auch wenn einzelne Einträge fehlgeschlagen sind.

-----

## Keine internen Details in Fehlerantworten (#177)

Stack Traces, Datenbankfehlermeldungen, interne Pfade und Implementierungsdetails dürfen niemals in Fehlerantworten erscheinen. Das ist nicht nur eine Stilfrage — es ist ein Sicherheitserfordernis.

**Verboten:**

```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "NullPointerException at com.example.OrderService.create(OrderService.java:142)",
  "stacktrace": "at com.example.OrderService.create(OrderService.java:142)\nat com.example.OrderController...",
  "sql": "SELECT * FROM orders WHERE id = NULL"
}
```

**Korrekt:**

```json
{
  "type": "https://api.example.com/errors/internal-error",
  "title": "Internal Server Error",
  "status": 500,
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

Der vollständige Fehler wird serverseitig geloggt und ist über die `trace_id` auffindbar. Nach aussen wird nur das Minimum zurückgegeben: Fehlertyp, Statuscode und Trace-ID für die Korrelation.

-----

## Trace-ID für Debugging (C-03, C-04, C-05)

Bei 5xx-Fehlern ist die Verbindung zwischen der Fehlerantwort und den Serverlog-Einträgen entscheidend. Diese Verbindung wird über den W3C Trace Context hergestellt.

Jeder eingehende Request trägt einen `traceparent`-Header:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Der Header enthält:

- `4bf92f3577b34da6a3ce929d0e0e4736` — die 32-stellige `trace_id` die den gesamten Request-Kontext über alle Services hinweg identifiziert
- `00f067aa0ba902b7` — die `span_id` die den aktuellen Service-Aufruf identifiziert

Tritt ein 5xx-Fehler auf, wird die `trace_id` aus dem `traceparent`-Header extrahiert und in die Problem JSON Response eingebettet (Regel C-05). Der aufrufende Dienst kann diese ID direkt im Monitoring-System oder Log-Aggregator nachschlagen:

```
Schritt 1: Fehler tritt auf → Response enthält trace_id: "4bf92f3577b34da6a3ce929d0e0e4736"
Schritt 2: In Jaeger / Azure Monitor suchen: trace_id = 4bf92f3577b34da6a3ce929d0e0e4736
Schritt 3: Vollständiger Request-Verlauf über alle Services sichtbar
```

Ist kein `traceparent`-Header im eingehenden Request vorhanden, generiert das Gateway (Gravitee) einen neuen Trace. Die `trace_id` ist damit immer vorhanden — unabhängig davon ob der aufrufende Dienst Tracing unterstützt.

-----

## Alle Statuscodes in OpenAPI dokumentieren (#151)

Jeder Endpunkt muss alle möglichen Statuscodes in der OpenAPI-Spezifikation dokumentieren — nicht nur den Erfolgsfall. Nach Regel **#151** ist das kein optionaler Schritt.

```yaml
paths:
  /v1/orders:
    post:
      summary: Create an order
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderRequest'
      responses:
        '201':
          description: Order created successfully
          headers:
            Location:
              schema:
                type: string
              description: URL of the created order
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'

        '400':
          description: Syntactically invalid request body
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
              example:
                type: "https://api.example.com/errors/invalid-json"
                title: "Invalid JSON"
                status: 400
                detail: "Unexpected token at position 32."

        '401':
          description: Missing or invalid authentication token
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
              example:
                type: "https://api.example.com/errors/unauthorized"
                title: "Unauthorized"
                status: 401

        '403':
          description: Insufficient permissions — scope write:orders required
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
              example:
                type: "https://api.example.com/errors/forbidden"
                title: "Forbidden"
                status: 403
                detail: "The scope 'write:orders' is required for this operation."

        '422':
          description: Semantically invalid request — validation failed
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemWithErrors'
              example:
                type: "https://api.example.com/errors/validation-error"
                title: "Validation Error"
                status: 422
                detail: "Multiple validation errors occurred."
                errors:
                  - field: "quantity"
                    message: "Must be greater than 0."
                    rejected_value: -5

        '429':
          description: Rate limit exceeded
          headers:
            Retry-After:
              schema:
                type: integer
              description: Seconds to wait before retrying
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Problem'
              example:
                type: "https://api.example.com/errors/rate-limit-exceeded"
                title: "Rate Limit Exceeded"
                status: 429
                detail: "The rate limit of 1000 requests per minute has been exceeded."

        '500':
          description: Unexpected server error
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemWithTraceId'
              example:
                type: "https://api.example.com/errors/internal-error"
                title: "Internal Server Error"
                status: 500
                trace_id: "4bf92f3577b34da6a3ce929d0e0e4736"

components:
  schemas:
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type:
          type: string
          format: uri
          example: "https://api.example.com/errors/validation-error"
        title:
          type: string
          example: "Validation Error"
        status:
          type: integer
          format: int32
          example: 422
        detail:
          type: string
          example: "Field 'quantity' must be greater than 0."
        instance:
          type: string
          format: uri
          example: "/v1/orders/ord_abc123"

    ProblemWithErrors:
      allOf:
        - $ref: '#/components/schemas/Problem'
        - type: object
          properties:
            errors:
              type: array
              items:
                type: object
                properties:
                  field:
                    type: string
                  message:
                    type: string
                  rejected_value: {}

    ProblemWithTraceId:
      allOf:
        - $ref: '#/components/schemas/Problem'
        - type: object
          properties:
            trace_id:
              type: string
              example: "4bf92f3577b34da6a3ce929d0e0e4736"
```

-----

## Fehler-URI — eigene Fehlertypen definieren

Jeder `type`-Wert im Problem JSON ist eine URI. Diese URIs müssen konsistent und stabil sein — sie sind Teil des öffentlichen API-Vertrags. Ändert sich eine Fehler-URI, ist das ein Breaking Change.

Empfohlenes Schema:

```
https://api.{organisation}.com/errors/{fehler-slug}
```

Beispiele:

```
https://api.example.com/errors/validation-error
https://api.example.com/errors/rate-limit-exceeded
https://api.example.com/errors/resource-not-found
https://api.example.com/errors/optimistic-locking-conflict
https://api.example.com/errors/invalid-cursor
https://api.example.com/errors/insufficient-stock
```

Der `fehler-slug` ist in kebab-case, beschreibt den Fehler fachlich und ist nicht an interne Implementierungsdetails gebunden. `NullPointerException` ist kein gültiger Slug — `internal-error` schon.

-----

## Häufige Fehler

**`500` für Validierungsfehler zurückgeben.** Ein ungültiges Request-Feld löst intern eine Exception aus, die unbehandelt als `500` nach aussen gelangt. Validierungsfehler sind `422` — sie sind erwartete, normale Zustände, keine Serverfehler.

**`400` pauschal für alle Fehler verwenden.** `400` bedeutet syntaktisch ungültige Anfrage. Für semantische Fehler, fehlende Berechtigungen oder nicht gefundene Ressourcen gibt es spezifischere Statuscodes. Regel #220 schreibt vor, den spezifischsten Code zu verwenden.

**Stack Traces in `detail` schreiben.** Der `detail`-Wert ist für menschenlesbare Fehlerbeschreibungen gedacht, nicht für technische Fehlermeldungen. Stack Traces gehören ausschliesslich in Server-Logs.

**`type` als generischen String befüllen.** `"type": "error"` oder `"type": "BAD_REQUEST"` sind keine validen URIs. Jeder Fehlertyp bekommt eine eindeutige, stabile URI nach dem definierten Schema.

**`Retry-After` bei `429` weglassen.** Ohne `Retry-After` weiss der aufrufende Dienst nicht wie lange gewartet werden soll. Er wird entweder sofort erneut anfragen (und erneut `429` erhalten) oder mit willkürlichen Wartezeiten arbeiten.

**Fehler nicht in OpenAPI spezifizieren.** Fehlt der `422`-Response im OpenAPI-Schema, wissen API-Konsumenten nicht welche Fehler zu erwarten sind. Jeder mögliche Statuscode muss nach Regel #151 dokumentiert sein — auch Fehlerstatuscodes.

**Verschiedene Fehlerformate in derselben API mischen.** Manche Endpunkte geben Problem JSON zurück, andere eigene Formate. Das zwingt aufrufende Dienste dazu, verschiedene Fehlerformate zu unterscheiden. Nach Regel #176 wird Problem JSON überall ohne Ausnahme verwendet.

-----

## Zusammenfassung

|Regel    |Kernaussage                                                                    |
|---------|-------------------------------------------------------------------------------|
|#176     |Problem JSON (`application/problem+json`) für alle Fehler — RFC 7807           |
|#177     |Keine Stack Traces, Datenbankfehler oder interne Pfade in Responses            |
|#243     |Nur offizielle HTTP-Statuscodes aus RFCs                                       |
|#220     |Spezifischsten Statuscode verwenden — `422` statt `400` bei Validierungsfehlern|
|#150     |Gebräuchliche Statuscodes bevorzugen                                           |
|#151     |Alle Statuscodes in OpenAPI dokumentieren — auch Fehlerstatuscodes             |
|#152     |`207 Multi-Status` für Batch-Operationen — mit eingebettetem Problem JSON      |
|#153     |`429` immer mit `Retry-After` Header                                           |
|C-03/C-04|W3C `traceparent` Header propagieren                                           |
|C-05     |`trace_id` in alle 5xx Problem JSON Responses                                  |
