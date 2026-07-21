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


# Architekturbeschreibung: Logging & Monitoring für Azure SQL Database

Stand der Recherche: Juli 2026, basierend auf aktueller Microsoft-Learn-Dokumentation. Wo Informationen unsicher, unvollständig oder umgebungsabhängig sind, ist dies explizit gekennzeichnet. Diese Beschreibung gilt für **Azure SQL Database** (Single Database / Elastic Pool).

---

## 1. Grundprinzip: drei getrennte Datenpfade

Azure SQL Database liefert Observability- und Sicherheitsdaten über **drei technisch und organisatorisch unabhängige Mechanismen**:

| | Plattformmetriken (Azure Monitor Metrics) | Diagnostic Settings (Logs) | Auditing |
|---|---|---|---|
| Ressourcentyp/Namespace | `Microsoft.Sql/servers/databases` (bzw. `.../elasticpools`) | Diagnostic Setting am selben Ressourcentyp | Eigene Ressource `auditingSettings` am Server oder an der Datenbank |
| Aktivierung | Automatisch aktiv | Muss explizit konfiguriert werden | Muss explizit konfiguriert werden – eigene Portal-Blade „Auditing", getrennt von den Diagnoseeinstellungen |
| Ziel | Azure-Monitor-Metrikspeicher | Log Analytics Workspace, Storage Account, Event Hub | Storage Account, Log Analytics Workspace oder Event Hub |
| Inhalt | Ressourcenverbrauch (CPU, DTU, Storage, Verbindungen …) | Performance-/Wartestatistiken, Query Store, Fehler, Blockaden | Sicherheitsrelevante Ereignisse: wer hat wann welche Abfrage/welches Login ausgeführt |
| Typischer Zweck | Kurzfristiges operatives Monitoring, Kapazitätsplanung | Tiefergehende Performance-Diagnose, Fehleranalyse | Compliance, Security-Audit, forensische Nachvollziehbarkeit |

**Wichtiger, oft übersehener Zusammenhang zwischen Diagnostic Settings und Auditing:** Wenn Auditing über die Auditing-Blade auf ein Log-Analytics- oder Event-Hub-Ziel konfiguriert wird, legt Azure im Hintergrund automatisch eine eigene Diagnostic Setting mit der Kategorie `SQLSecurityAuditEvents` an (Namensmuster `SQLSecurityAuditEvents_<GUID>`). Löscht man diese Diagnostic Setting manuell, versagt das Auditing **lautlos** – ohne Fehlermeldung, aber ohne weitere Audit-Daten. Microsoft empfiehlt explizit, einen Alert auf das Löschen dieser Diagnostic Setting einzurichten. Mehrere Community-Quellen berichten außerdem übereinstimmend, dass das manuelle Aktivieren der Kategorie `SQLSecurityAuditEvents` direkt in den Diagnoseeinstellungen – ohne den Umweg über die Auditing-Blade – **nicht** zu befüllten Audit-Daten führt (das ist keine Aussage, die ich in der offiziellen Microsoft-Dokumentation selbst so explizit gefunden habe, sondern eine übereinstimmende Beobachtung mehrerer unabhängiger Community-Quellen – ich kennzeichne sie deshalb als nicht offiziell verifiziert, aber plausibel). Für Auditing ist also die Auditing-Blade bzw. die entsprechende dedizierte Terraform-Ressource der vorgesehene Weg.

---

## 2. Baustein A: Plattformmetriken (Azure Monitor Metrics)

### 2.1 Funktionsweise
Der Ressourcenprovider `Microsoft.Sql/servers/databases` (für Elastic Pools: `Microsoft.Sql/servers/elasticpools`) emittiert automatisch Metriken direkt in den Azure-Monitor-Metrikspeicher – ohne Zutun des Nutzers, unabhängig von jeder Diagnostic-Settings-Konfiguration.

- Die meisten Metriken werden im **1-Minuten-Takt (`PT1M`)** erfasst.
- Backup-bezogene Größen (`full_backup_size_bytes`, `diff_backup_size_bytes`, `log_backup_size_bytes`, `snapshot_backup_size_bytes`) werden nur **täglich (`P1D`)** erfasst.
- **Aufbewahrung vs. Abfragefenster:** Die Metrikdaten selbst werden bis zu **93 Tage** im Metrikspeicher vorgehalten. Ein einzelnes Diagramm im Metrics Explorer (Portal oder REST API) kann davon aber nur ein zusammenhängendes Zeitfenster von maximal **30 Tagen** gleichzeitig darstellen – eine reine Abfrage-/Darstellungsbeschränkung pro Chart, keine Reduktion der Speicherdauer. Die Daten außerhalb der letzten 30 Tage lassen sich durch Schwenken des Charts oder eine neue Abfrage mit anderem 30-Tage-Fenster erreichen. Für zusammenhängende Trend- oder Kapazitätsanalysen über mehr als 30 Tage empfiehlt sich der Export der Metriken über Diagnostic Settings (Kategorie `AllMetrics`) in einen Log Analytics Workspace.

### 2.2 Wichtige Metrikkategorien

Azure SQL Database unterscheidet zwei grundlegend verschiedene **Abrechnungs-/Kapazitätsmodelle** – DTU-basiert (gebündelte Ressourceneinheit) und vCore-basiert (granularer, mit separater CPU-/Speicher-Steuerung). Ein Teil der Metriken ist jeweils nur für eines der beiden Modelle relevant, ein anderer Teil nur für bestimmte Tarife (Serverless, Hyperscale, Data-Warehouse-Workloads).

| Kategorie | Beispielmetriken | Bemerkung |
|---|---|---|
| Ressourcenverbrauch (DTU-Modell) | `dtu_consumption_percent`, `dtu_limit`, `dtu_used` | Nur relevant bei DTU-basierten Datenbanken |
| Ressourcenverbrauch (vCore-Modell) | `cpu_percent`, `cpu_limit`, `cpu_used`, `sql_instance_cpu_percent`, `sql_instance_memory_percent` | Nur relevant bei vCore-basierten Datenbanken |
| Storage | `storage`, `storage_percent`, `allocated_data_storage`, `xtp_storage_percent` (In-Memory OLTP) | `storage_percent` nicht anwendbar bei Hyperscale-Datenbanken |
| I/O | `log_write_percent` (Log-IO), `physical_data_read_percent` (Data-IO) | Engpass-Indikatoren für I/O-lastige Workloads |
| Verbindungen | `connection_successful`, `connection_failed` (Systemfehler), `connection_failed_user_error` (z. B. falsches Passwort), `sessions_count`, `sessions_percent`, `workers_percent` | `connection_failed` und `connection_failed_user_error` sind bewusst getrennte Metriken – wichtig für die Unterscheidung Infrastruktur- vs. Anwendungsproblem |
| Verfügbarkeit | `availability` | Prozentsatz SLA-konformer Verfügbarkeit; pro 1-Minuten-Datenpunkt entweder 100 % (Verbindung(en) erfolgreich) oder 0 % (alle Verbindungen an Systemfehlern gescheitert) – Microsoft empfiehlt explizit die 1-Minuten-Granularität für eine korrekte SLA-Betrachtung |
| Fehler/Sperren | `deadlock` | Nicht anwendbar bei Data-Warehouse-Workloads |
| Tempdb | `tempdb_data_size`, `tempdb_log_size`, `tempdb_log_used_percent` | Relevant bei tempdb-intensiven Workloads (Sortierungen, temporäre Tabellen) |
| Serverless-spezifisch | `app_cpu_percent`, `app_cpu_billed`, `app_memory_percent` | Nur bei Serverless-Tarif |
| Backup-Speicher | `full_backup_size_bytes`, `diff_backup_size_bytes`, `log_backup_size_bytes`, `snapshot_backup_size_bytes` (Hyperscale) | Tägliches Sampling |
| Data-Warehouse-spezifisch (Kategorie WorkloadManagement) | `wlg_active_queries`, `wlg_queued_queries`, `dwu_consumption_percent`, `cache_hit_percent` u. a. | Nur relevant, wenn die Datenbank als Data-Warehouse-Workload betrieben wird |

*Hinweis:* Diese Auswahl ist kuratiert, nicht vollständig. Die verbindliche Gesamtliste inkl. exakter Aggregationstypen und Zeitgranularitäten steht in der Microsoft-Referenz „Supported metrics – Microsoft.Sql/servers/databases".

### 2.3 Bekannte Einschränkung bei geringer Auslastung
Microsoft dokumentiert ein Rundungsverhalten: Bei sehr niedriger Auslastung können Metrikwerte kleiner als 0,5 auf 0 gerundet werden, wodurch bei sehr kleinen/wenig genutzten Datenbanken oder Elastic Pools eine geringere tatsächliche Auslastung als real vorhanden angezeigt werden kann. Das ist bei der Interpretation von Metriken auf Testsystemen oder sehr kleinen SKUs zu beachten.

### 2.4 Nutzung/Konsumenten
- **Metrics Explorer** im Portal (Ad-hoc-Analyse)
- **Azure Monitor Alerts** (schwellwertbasiert)
- **Query Performance Insight**: eine in Azure SQL Database eingebaute Portal-Funktion, die direkt auf dem SQL-eigenen Query Store aufsetzt und Query-Tuning-Empfehlungen sowie Performance-Analysen liefert – funktioniert unabhängig von Diagnostic Settings, da sie nicht über Azure Monitor läuft
- **Automatic Tuning / Datenbank-Advisor**: ebenfalls direkt im Portal integriert, liefert Empfehlungen (z. B. fehlende oder überflüssige Indizes) auf Basis interner Telemetrie – ebenfalls unabhängig von Diagnostic Settings nutzbar
- **Export via `AllMetrics`** in Diagnostic Settings (Tabelle `AzureMetrics` im Log Analytics Workspace) für Trendanalysen über 30 Tage hinaus

---

## 3. Baustein B: Diagnostic Settings (Logs)

### 3.1 Funktionsweise
Diagnostic Settings sind eine separate Azure-Monitor-Ressource, die pro Datenbank (oder serverweit über die logische Servereinheit) angelegt werden muss. Ohne aktive Diagnostic Setting werden keine Logs erfasst.

### 3.2 Verfügbare Log-Kategorien

| Kategorie | Anzeigename | Inhalt |
|---|---|---|
| `SQLInsights` | SQL Insights | Grundlage für automatische Performance-Diagnose-Insights |
| `AutomaticTuning` | Automatic tuning | Ereignisse rund um automatische Tuning-Aktionen (z. B. automatisch erstellte/entfernte Indizes) |
| `QueryStoreRuntimeStatistics` | Query Store Runtime Statistics | Laufzeitstatistiken einzelner Queries aus dem SQL-eigenen Query Store |
| `QueryStoreWaitStatistics` | Query Store Wait Statistics | Wartestatistiken je Query aus dem Query Store |
| `Errors` | Errors | Datenbankfehler |
| `DatabaseWaitStatistics` | Database Wait Statistics | Aggregierte Wartestatistiken auf Datenbankebene |
| `Timeouts` | Timeouts | Zeitüberschreitungen |
| `Blocks` | Blocks | Blockierungssituationen |
| `Deadlocks` | Deadlocks | Deadlock-Ereignisse |
| `ExecRequests` | Exec Requests | Laufende/abgeschlossene Ausführungsanfragen |
| `RequestSteps` | Request Steps | Einzelschritte von Anfragen (v. a. relevant bei Data-Warehouse-Workloads) |
| `SqlRequests` | Sql Requests | SQL-Anfragen |
| `DmsWorkers` | Dms Workers | Data-Movement-Service-Worker (Data-Warehouse-Kontext) |
| `SQLSecurityAuditEvents` | SQL Security Audit Event | Sicherheits-/Audit-Ereignisse – wird normalerweise nicht manuell hier aktiviert, sondern automatisch über die Auditing-Blade angelegt (siehe Kapitel 1 und 4) |
| `DevOpsOperationsAudit` | Devops operations Audit Logs | Audit-Ereignisse zu Microsoft-Support-Operationen auf dem Server |
| (Metrics-Kategorie) `AllMetrics` | – | Export der Plattformmetriken aus Kapitel 2 |

### 3.3 Zielspeicher und Tabellenmodell

Laut der aktuellen Microsoft-Referenztabelle (Dokument zuletzt aktualisiert im April 2025) landen bei Ziel Log Analytics Workspace **alle** Log-Kategorien in der generischen **`AzureDiagnostics`**-Tabelle, unterscheidbar über die Spalte `Category`:

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.SQL"
| where Category == "Errors"
| where TimeGenerated > ago(1d)
```

*Einschränkung meiner Aussage:* Ich stütze mich hier auf die Spalte „Log table" der Microsoft-Referenztabelle, die für jede Kategorie `AzureDiagnostics` ausweist. Da sich Zieltabellenmodelle bei anderen Azure-Diensten in der Vergangenheit geändert haben, kann ich nicht ausschließen, dass sich das inzwischen ebenfalls geändert hat – das sollte vor der Implementierung anhand der aktuellen Diagnostic-Settings-Konfigurationsoberfläche geprüft werden.

Bei Ziel **Event Hub**: für Streaming an externe SIEM-/Log-Management-Lösungen oder eigene Verarbeitung.
Bei Ziel **Storage Account**: für kostengünstige Langzeitarchivierung/Compliance, keine native Abfragefunktion.

### 3.4 Nutzung/Konsumenten
- **Log Analytics / KQL** für Fehleranalyse, Wait-Statistik-Auswertung, Query-Store-Historie über die im Query Store selbst konfigurierte Aufbewahrung hinaus
- **Workbooks**
- Query Performance Insight (siehe Kapitel 2.4) arbeitet direkt auf dem Query Store, nicht auf den hierüber exportierten Logs – die beiden Wege ergänzen sich, sind aber technisch unabhängig

---

## 4. Baustein C: Auditing

### 4.1 Funktionsweise
Auditing ist eine eigenständige Azure-SQL-Funktion (Portal: Abschnitt „Security" → „Auditing", Ressourcentyp `Microsoft.Sql/servers/auditingSettings` bzw. `.../databases/auditingSettings`), keine Diagnostic Setting. Sie protokolliert sicherheitsrelevante Datenbankereignisse: erfolgreiche/fehlgeschlagene Logins, ausgeführte Abfragen und gespeicherte Prozeduren. Die Standard-Policy umfasst u. a. die Aktionsgruppen `BATCH_COMPLETED_GROUP`, `SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP` und `FAILED_DATABASE_AUTHENTICATION_GROUP`.

Auditing kann auf **Server-Ebene** (gilt für alle aktuellen und künftigen Datenbanken auf dem logischen Server – von Microsoft als empfohlener Standardweg genannt) oder auf **Datenbank-Ebene** konfiguriert werden; beide können parallel aktiv sein und protokollieren dann unabhängig voneinander in ihre jeweiligen Ziele.

Als Ziel stehen zur Wahl:
- Azure Storage Account
- Log Analytics Workspace (dabei automatische Diagnostic-Setting-Erzeugung, siehe Kapitel 1)
- Event Hub

### 4.2 Wichtige Betriebshinweise
- **Performance-Optimierung vor Vollständigkeit:** Microsoft dokumentiert ausdrücklich, dass Auditing auf Verfügbarkeit und Performance der Datenbank hin optimiert ist – bei sehr hoher Aktivität oder hoher Netzwerklast können einzelne Audit-Ereignisse **nicht** aufgezeichnet werden. Für ein forensisch lückenloses Audit-Log ist das relevant.
- **Verzögerung:** Eine Community-Quelle berichtet von einer Verzögerung von bis zu rund zwei Stunden, bis Security-Audit-Events in Log Analytics sichtbar werden. Das ist keine von mir bei Microsoft offiziell bestätigte Zeitangabe, sondern stammt aus einer Community-Quelle – ich nenne sie trotzdem, weil sie für die Erwartungshaltung bei der Fehlersuche relevant ist, markiere sie aber ausdrücklich als nicht offiziell verifiziert.
- **Stille Fehlerzustände:** Wird die automatisch angelegte Diagnostic Setting (`SQLSecurityAuditEvents_<GUID>`) gelöscht, versagt Auditing ohne sichtbare Fehlermeldung. Microsoft empfiehlt hier explizit einen Alert auf Löschungen von Diagnostic Settings (Activity-Log-Alert).
- **Read-Only-Replikate:** Auditing wird auf Lesereplikaten automatisch mit aktiviert.
- **Sensible Daten:** In Kombination mit Data Classification protokolliert Auditing zusätzlich ein Feld `data_sensitivity_information`, das die Sensitivitätskennzeichnung der zurückgegebenen Daten enthält – relevant für die Nachverfolgung des Zugriffs auf klassifizierte Spalten.

---

## 5. Architekturübersicht (Textdiagramm)

```
                    +-------------------------------------------+
                    |  Azure SQL Database                        |
                    |  (Microsoft.Sql/servers/databases)          |
                    +-------+---------------+---------------+----+
                            |               |               |
              (immer aktiv)|  (opt-in)     |  (opt-in,      |
                            |               |   eigene Blade)|
                            v               v                v
                +--------------------+ +------------------+ +---------------------+
                | Azure Monitor       | | Diagnostic        | | Auditing             |
                | Metrics              | | Settings (Logs)   | | (auditingSettings)   |
                | 93 Tage Speicherung, | | -> AzureDiagnostics| | -> erzeugt intern    |
                | 30-Tage-Chart-Fenster| |    (generische     | |    eine Diagnostic   |
                +---------+------------+ |    Tabelle)        | |    Setting mit       |
                          |              +---------+----------+ |    SQLSecurityAudit- |
                          |                        |            |    Events            |
                          v                        v            +----------+-----------+
                  Metrics Explorer,        Log Analytics /                 |
                  Alerts, Query            KQL, Workbooks                  v
                  Performance Insight,                            Storage Account /
                  Automatic Tuning                                 Log Analytics /
                  (portal-nativ, ohne                                Event Hub
                   Diagnostic Settings)
```

---

## 6. Zusammenfassende Entscheidungslogik

| Anforderung | Empfohlener Pfad |
|---|---|
| CPU/DTU/Storage-Überwachung, einfache Schwellwert-Alerts | Plattformmetriken, kein Diagnostic Setting nötig |
| Verfügbarkeits-Monitoring | Metrik `availability`, 1-Minuten-Granularität |
| Analyse langsamer Queries im Portal, ohne Log-Analytics-Setup | Query Performance Insight (nativ, kein Diagnostic Setting nötig) |
| Historische Query-Store-Auswertung über die native Query-Store-Aufbewahrung hinaus, KQL-Analysen | Diagnostic Settings mit `QueryStoreRuntimeStatistics`/`QueryStoreWaitStatistics` |
| Deadlock-/Blocking-Analyse | Diagnostic Settings mit `Deadlocks`, `Blocks`, `DatabaseWaitStatistics` |
| Compliance-/Security-Audit (wer hat was ausgeführt) | Auditing-Blade bzw. die dedizierte Auditing-Ressource konfigurieren, nicht die Diagnoseeinstellungen direkt |
| Microsoft-Support-Operationen nachvollziehen | Diagnostic Settings mit `DevOpsOperationsAudit` |

---

## 7. Empfohlene Mindeststandards: Start vs. Produktivbetrieb

**Wichtiger Hinweis vorab:** Microsoft veröffentlicht keinen offiziellen, verbindlichen „Minimal-Standard" für Logging, Monitoring und Auditing bei Azure SQL Database. Was folgt, ist meine fachliche Einschätzung/Empfehlung auf Basis der in Kapitel 1–6 beschriebenen, dokumentierten Mechanismen – keine von Microsoft vorgegebene Checkliste. Konkrete Schwellwerte sind begründete Vorschläge, keine Herstellervorgaben, und je nach Workload anzupassen.

### 7.1 Minimalstandard zum Start (Dev/Test/erste Inbetriebnahme)

| Bereich | Empfehlung |
|---|---|
| Plattformmetriken | Nichts zu konfigurieren – automatisch aktiv. Bei Bedarf ad hoc über Metrics Explorer ansehen. |
| Diagnostic Settings | Optional, kann in dieser Phase entfallen. Falls doch gewünscht: eine einzige Diagnostic Setting mit Ziel Log Analytics Workspace, nur Kategorie `Errors`. |
| Auditing | Nicht zwingend nötig in dieser Phase. |
| Alerts | Minimal zwei einfache Alerts: `availability` (Verfügbarkeit) und je nach Abrechnungsmodell `storage_percent` bzw. `dtu_consumption_percent`/`cpu_percent`. |

### 7.2 Minimalstandard für eine Produktivdatenbank

Das ist die untere Grenze dessen, was ich für eine produktiv genutzte Datenbank als verantwortbar ansehe – nicht die vollständige „Best Practice"-Ausstattung. Query-Store-Diagnose-Kategorien, Data-Warehouse-spezifische Metriken oder eine feingranulare Auditing-Aktionsgruppen-Konfiguration gehen darüber hinaus und sind je nach Anforderung zusätzlich sinnvoll.

**a) Plattformmetriken – Alerts mit Action Group**

Alerts ohne Benachrichtigungsziel sind wirkungslos: mindestens eine **Action Group** (E-Mail, Teams, PagerDuty o. ä.), gebunden an folgende Alerts:

| Metrik | Grund |
|---|---|
| `availability` | Basis-Verfügbarkeit |
| `storage_percent` (bzw. bei Hyperscale eine geeignete Alternative, da dort nicht anwendbar) | Speicherengpässe frühzeitig erkennen |
| `cpu_percent` (vCore) bzw. `dtu_consumption_percent` (DTU) | Anhaltend hohe Last als Frühindikator für Performance-Probleme |
| `sessions_percent` bzw. `workers_percent` | Drohende Erschöpfung von Sessions/Workern |
| `connection_failed` | Deutet auf Konfigurations- oder Kapazitätsprobleme hin |
| `deadlock` | Wiederkehrende Deadlocks als Hinweis auf Anwendungs-/Schema-Probleme |

**b) Diagnostic Settings – minimal produktiv-tauglicher Umfang**

- Ziel: Log Analytics Workspace
- Minimal sinnvolle Log-Kategorien: `Errors`, `Deadlocks`, `Blocks`, `DatabaseWaitStatistics`
- Nicht zwingend Teil des Minimalstandards, aber naheliegende nächste Ausbaustufe: `QueryStoreRuntimeStatistics`, `QueryStoreWaitStatistics` für tiefergehende Query-Performance-Diagnose
- `AllMetrics`-Export in denselben Workspace: sinnvoll, sobald Trendanalysen über mehr als 30 Tage gebraucht werden

**c) Auditing – für Produktivsysteme Teil des Minimalstandards**

Anders als in der Startphase stufe ich Auditing für eine produktive Datenbank als Minimalstandard ein, nicht als optionales Extra:
- Server-Level-Auditing aktivieren, Ziel Log Analytics Workspace
- Standard-Aktionsgruppen belassen (`BATCH_COMPLETED_GROUP`, `SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP`, `FAILED_DATABASE_AUTHENTICATION_GROUP`)
- Zusätzlich einen **Activity-Log-Alert auf das Löschen der automatisch erzeugten Diagnostic Setting** (`SQLSecurityAuditEvents_*`) einrichten, da Auditing sonst lautlos ausfällt (siehe Kapitel 4.2)

### 7.3 Kurzer Vergleich

| | Start-Minimalstandard | Produktiv-Minimalstandard |
|---|---|---|
| Diagnostic Settings | optional | verpflichtend (`Errors`, `Deadlocks`, `Blocks`, `DatabaseWaitStatistics`) |
| Auditing | nicht nötig | verpflichtend, Server-Level, mit Löschungs-Alert |
| Alerts | 2 (Verfügbarkeit, Storage/Auslastung) | 6 (siehe Tabelle 7.2a), mit Action Group |

---

## 8. Konfiguration als Code: Terraform

**Grundsätzliche Antwort: Ja**, sämtliche in dieser Architektur beschriebenen Bausteine lassen sich über den `azurerm`-Provider deklarativ verwalten.

### 8.1 Relevante Terraform-Ressourcen

| Zweck | Terraform-Ressource | Bemerkung |
|---|---|---|
| Logischer SQL-Server | `azurerm_mssql_server` | Basisressource |
| Datenbank | `azurerm_mssql_database` | SKU (DTU/vCore), Storage, etc. |
| Diagnostic Setting (Baustein B) | `azurerm_monitor_diagnostic_setting` | `enabled_log { category = "..." }` je Log-Kategorie, `enabled_metric { category = "AllMetrics" }` |
| Log Analytics Workspace | `azurerm_log_analytics_workspace` | Ziel für Diagnostic Settings und Auditing |
| Auditing – Server-Ebene | `azurerm_mssql_server_extended_auditing_policy` | Aktiviert Auditing für alle Datenbanken auf dem Server |
| Auditing – Datenbank-Ebene | `azurerm_mssql_database_extended_auditing_policy` | Aktiviert Auditing für eine einzelne Datenbank |
| Metrikbasierte Alerts | `azurerm_monitor_metric_alert` | Für die Schwellwert-Alerts aus Kapitel 7.2a |
| Alert auf Löschung der Diagnostic Setting | `azurerm_monitor_activity_log_alert` | Für den in Kapitel 7.2c empfohlenen Schutzmechanismus |

### 8.2 Bekannte Einschränkung bei Auditing mit Log-Analytics-Ziel

Für **Storage-Account-Ziele** funktioniert `azurerm_mssql_server_extended_auditing_policy`/`azurerm_mssql_database_extended_auditing_policy` unmittelbar über die eigenen Ressourcenattribute. Für ein **Log-Analytics-Ziel** ist die Lage laut mehreren, über mehrere Jahre verteilten GitHub-Issues zum `azurerm`-Provider unübersichtlicher: Die reine Aktivierung über `log_monitoring_enabled = true` an der Extended-Auditing-Policy-Ressource verknüpft den Log Analytics Workspace in der Praxis nicht zuverlässig im Portal sichtbar. Ein in einem GitHub-Issue dokumentiertes, funktionierendes Muster kombiniert beide Ressourcen:

```hcl
resource "azurerm_mssql_server_extended_auditing_policy" "main" {
  server_id               = azurerm_mssql_server.this.id
  retention_in_days       = 30
  log_monitoring_enabled  = true
}

resource "azurerm_monitor_diagnostic_setting" "sql_audit" {
  name                       = "sql-audit-to-log-analytics"
  target_resource_id         = "${azurerm_mssql_server.this.id}/databases/master"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.this.id

  enabled_log {
    category = "SQLSecurityAuditEvents"
  }

  depends_on = [azurerm_mssql_server_extended_auditing_policy.main]
}
```

*Einschränkung meiner Aussage:* Dieses Muster stammt aus einem öffentlichen, mit „Bug" gekennzeichneten GitHub-Issue zum `azurerm`-Provider (Stand August 2024) und nicht aus der offiziellen Terraform-Registry-Dokumentation als der vorgesehene Standardweg. Ob dieses Verhalten in der aktuell von euch verwendeten Provider-Version noch genauso zutrifft, kann ich nicht mit Sicherheit sagen – das sollte vor der Umsetzung in einer Testumgebung verifiziert werden.

### 8.3 Was ich dazu nicht mit Sicherheit sagen kann

- Ob **jede** in Kapitel 3.2 gelistete Log-Kategorie zum aktuellen Zeitpunkt vollständig und ohne Provider-Lücken über `azurerm` abbildbar ist, kann ich nicht abschließend bestätigen.
- Konkrete Mindest-Versionsanforderungen an den `azurerm`-Provider nenne ich hier bewusst nicht, da sich das häufig ändert und ich das nicht zuverlässig aktuell verifizieren kann.

---

## 9. Offene Punkte / bewusst nicht beantwortet

- **Konkrete Kostenmodelle** für Diagnostic-Settings-Export und Auditing-Speicherung: keine gesicherten aktuellen Preisinformationen vorhanden – Azure-Preisrechner konsultieren.
- **Ob das in Kapitel 3.3 beschriebene, ausschließliche `AzureDiagnostics`-Tabellenmodell zum Zeitpunkt der Umsetzung noch aktuell ist** – das ist eine Momentaufnahme, keine dauerhafte Garantie.
- **Exakte Verzögerungszeit bei Auditing-Events** (Kapitel 4.2) – nur aus einer Community-Quelle, nicht offiziell von Microsoft bestätigt.
- **Terraform-Verhalten für Auditing mit Log-Analytics-Ziel** (Kapitel 8.2) – auf Basis eines GitHub-Issues, nicht der offiziellen Registry-Dokumentation als Standardweg.

## 10. Quellen (Microsoft Learn, abgerufen Juli 2026)

- Monitoring data reference for Azure SQL Database: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-sql-database-azure-monitor-reference?view=azuresql
- Supported metrics – Microsoft.Sql/servers/databases: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-sql-servers-databases-metrics
- Supported log categories – Microsoft.Sql/servers/databases: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-sql-servers-databases-logs
- Monitoring and performance tuning – Azure SQL Database & Managed Instance: https://learn.microsoft.com/en-us/azure/azure-sql/database/monitor-tune-overview?view=azuresql
- Auditing – Azure SQL Database and Azure Synapse Analytics (Übersicht): https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-overview?view=azuresql
- Set up Auditing – Azure SQL Database & Azure Synapse Analytics: https://learn.microsoft.com/en-us/azure/azure-sql/database/auditing-setup?view=azuresql
- Explore server and database audit (Microsoft Learn Training): https://learn.microsoft.com/training/modules/implement-compliance-controls-sensitive-data/3-explore-server-and-database-audit
- Terraform-Ressource `azurerm_mssql_server_extended_auditing_policy`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mssql_server_extended_auditing_policy
- Terraform-Ressource `azurerm_mssql_database_extended_auditing_policy`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/mssql_database_extended_auditing_policy
- GitHub-Issue zu Auditing mit Log-Analytics-Ziel (Terraform): https://github.com/hashicorp/terraform-provider-azurerm/issues/27025


---

# Architekturbeschreibung: Logging & Monitoring für Azure Database for PostgreSQL – Flexible Server

Stand der Recherche: Juli 2026, basierend auf aktueller Microsoft-Learn-Dokumentation. Wo Informationen unsicher, unvollständig oder umgebungsabhängig sind, ist dies explizit gekennzeichnet.

---

## 1. Grundprinzip: zwei getrennte Datenpfade

Azure Flexible Server für PostgreSQL liefert Observability-Daten über **zwei technisch und organisatorisch unabhängige Mechanismen**, die häufig verwechselt werden:

| | Plattformmetriken (Azure Monitor Metrics) | Diagnostic Settings (Logs, optional auch Metrics-Export) |
|---|---|---|
| Datenquelle | Azure-Ressourcenprovider `Microsoft.DBforPostgreSQL/flexibleServers` | PostgreSQL-Serverprozess (Log-Dateien, Query Store, Sessions, PgBouncer, Autovacuum-Statistiken) |
| Aktivierung | Standardmäßig aktiv, keine Konfiguration nötig | Muss explizit über eine Diagnostic Setting-Regel je Server konfiguriert werden |
| Speicherort | Azure Monitor Metrics-Datenbank (intern, zeitreihenoptimiert) | Ziel frei wählbar: Log Analytics Workspace, Storage Account, Event Hub |
| Aufbewahrung | Bis zu 93 Tage, im Portal-Chart max. 30 Tage abfragbar | Abhängig vom gewählten Ziel (z. B. Log Analytics Workspace-Aufbewahrung, standardmäßig oft 30 Tage, konfigurierbar) |
| Abfrage | Azure Monitor Metrics Explorer, REST API, Alerts, Workbooks | Kusto Query Language (KQL) in Log Analytics, bzw. Rohdaten in Storage/Event Hub |
| Typischer Zweck | Kurzfristiges operatives Monitoring, Schwellwert-Alerts, Kapazitätsplanung | Tiefergehende Diagnose, Query-Analyse, Audit, Troubleshooting-Guides, Langzeit-/Archivierungsszenarien |

Diese Trennung ist die zentrale Architekturentscheidung, die sich in Kosten, Latenz und Aufbewahrung niederschlägt.

---

## 2. Baustein A: Plattformmetriken (Azure Monitor Metrics)

### 2.1 Funktionsweise
Der Ressourcenprovider `Microsoft.DBforPostgreSQL/flexibleServers` emittiert kontinuierlich Metriken direkt in den Azure-Monitor-Metrikspeicher. Dies geschieht ohne Zutun des Nutzers, ist im Grundumfang kostenfrei und unabhängig von jeder Diagnostic-Setting-Konfiguration.

- Die meisten Metriken werden im **1-Minuten-Takt (PT1M)** erfasst.
- Einige Kategorien (z. B. Autovacuum-Statistiken) werden nur alle **30 Minuten** oder in größeren Intervallen (PT1H, PT6H, PT12H, P1D) erfasst.
- **Aufbewahrung vs. Abfragefenster (wichtiger Unterschied):** Die Metrikdaten selbst werden bis zu **93 Tage** im Metrikspeicher von Azure Monitor vorgehalten – das ist die reine Speicherdauer und gilt unabhängig davon, wie man die Daten betrachtet. Ein **einzelnes Diagramm** in der Metrics-Kachel (Portal oder REST API) kann davon aber nur ein zusammenhängendes Zeitfenster von maximal **30 Tagen** gleichzeitig darstellen. Das ist eine reine Abfrage-/Darstellungsbeschränkung pro Chart, keine Reduktion der Speicherdauer: Die Daten außerhalb der letzten 30 Tage sind nicht gelöscht, sondern lassen sich durch Schwenken ("pan") des Charts erreichen, oder man stellt eine neue Abfrage mit einem anderen 30-Tage-Fenster innerhalb der 93 Tage. Diese Log-basierte-Metriken-Ausnahme gilt nicht.
  - *Praktische Konsequenz:* Für zusammenhängende Trend- oder Kapazitätsanalysen über mehr als 30 Tage (z. B. Jahresvergleiche) reicht die reine Metrics-Explorer-Ansicht nicht aus. Microsoft empfiehlt dafür den Export der Metriken über Diagnostic Settings (Kategorie `AllMetrics`) in einen Log Analytics Workspace – dort gilt die 30-Tage-Chart-Grenze nicht, und die Aufbewahrung ist über die 93 Tage hinaus frei konfigurierbar (siehe Kapitel 3).
  - *Warum genau 30 Tage als Chart-Grenze gewählt wurden*, begründet Microsoft in der Dokumentation nicht explizit. Eine naheliegende Vermutung wäre eine Performance-/Rendering-Grenze für die interaktive Darstellung hochauflösender 1-Minuten-Zeitreihen – das ist jedoch **meine Einschätzung, keine dokumentierte Tatsache**.
- Ein Teil der Metriken (Autovacuum-Metriken, PgBouncer-Metriken, „Enhanced Metrics") ist **standardmäßig deaktiviert** und muss über dynamische Server-Parameter aktiviert werden (kein Neustart nötig), z. B.:
  - `metrics.autovacuum_diagnostics = ON`
  - `metrics.pgbouncer_diagnostics = ON` (zusätzlich `pgbouncer.enabled = ON`)
  - `metrics.collector_database_activity` für weitere granulare Metriken

### 2.2 Wichtige Metrikkategorien (Auswahl, nicht vollständig)

| Kategorie | Beispielmetriken | Bemerkung |
|---|---|---|
| Saturation | `cpu_percent`, `memory_percent`, `storage_percent`, `storage_used`, `storage_free`, `iops`, `read_iops`, `write_iops`, `disk_iops_consumed_percentage`, `disk_bandwidth_consumed_percentage`, `disk_queue_depth`, `backup_storage_used`, `cpu_credits_consumed/remaining` (nur Burstable-SKU) | Kern-Infrastrukturmetriken für Kapazitäts- und Performance-Monitoring |
| Traffic | `active_connections`, `connections_succeeded`, `max_connections`, `network_bytes_ingress/egress`, `tcp_connection_backlog` (ab 8 vCores) | Verbindungslast |
| Errors | `connections_failed` | Fehlgeschlagene Verbindungsversuche |
| Availability | `is_db_alive` (1 = verfügbar, 0 = nicht verfügbar) | Für Verfügbarkeits-Alerts geeignet |
| Database | `xact_commit`, `xact_rollback`, `deadlocks`, `tup_inserted/updated/deleted/fetched`, `temp_files`, `temp_bytes`, `database_size_bytes`, `numbackends`, `tps` (Preview) | Je Datenbank aufschlüsselbar über Dimension `DatabaseName` |
| Replication | `physical_replication_delay_in_bytes`, `physical_replication_delay_in_seconds` | Read-Replica-Lag |
| Logical Replication | `logical_replication_delay_in_bytes`, `logical_replication_slot_sync_status` (Preview) | Für logische Replikation/CDC-Szenarien |
| Activity | `longest_query_time_sec`, `longest_transaction_time_sec`, `oldest_backend_time_sec`, `sessions_by_state`, `sessions_by_wait_event_type` | Session- und Wait-Event-Übersicht |
| Autovacuum | `n_dead_tup_user_tables`, `n_live_tup_user_tables`, `bloat_percent` (Preview), `vacuum_count_user_tables` u. a. | Muss aktiviert werden, 30-Minuten-Takt |
| PgBouncer | `client_connections_active/waiting`, `server_connections_active/idle`, `num_pools`, `total_pooled_connections` | Nur relevant, wenn integrierter PgBouncer genutzt wird |

*Hinweis:* Die vollständige, verbindliche Liste inkl. exakter Einheiten und Aggregationstypen findet sich in der Microsoft-Referenz „Supported metrics – Microsoft.DBforPostgreSQL/flexibleServers". Ich gebe hier eine kuratierte Auswahl wieder, keine vollständige Kopie der Tabelle.

### 2.3 Vertiefung: Autovacuum-, PgBouncer- und „Enhanced"-Metriken

Diese drei Gruppen sind nicht Teil der Standard-Metriken und werden erfahrungsgemäß am häufigsten übersehen, obwohl sie für den produktiven Betrieb relevant sind.

**a) Autovacuum-Metriken – was und wozu**

PostgreSQL nutzt intern einen Hintergrundprozess namens *Autovacuum*, der zwei Aufgaben übernimmt:
- **VACUUM**: entfernt „tote" Zeilen (dead tuples), die durch UPDATE/DELETE entstehen (PostgreSQL überschreibt Zeilen nicht, sondern markiert alte Versionen als ungültig – MVCC-Modell), und gibt den Platz für Wiederverwendung frei.
- **ANALYZE**: aktualisiert die Tabellenstatistiken, die der Query-Planer für Ausführungspläne benötigt.

Läuft Autovacuum nicht ausreichend, wachsen Tabellen unnötig an (**Bloat**), Abfragen werden langsamer, und im schlimmsten Fall droht ein **Transaction-ID-Wraparound** – ein Zustand, der PostgreSQL zwingt, in einen reinen Lesemodus zu wechseln, bis manuell eingegriffen wird. Die Autovacuum-Metriken (`n_dead_tup_user_tables`, `n_live_tup_user_tables`, `bloat_percent`, `vacuum_count_user_tables`, `autovacuum_count_user_tables` u. a.) machen genau diesen Zustand sichtbar, bevor er kritisch wird.

*Warum standardmäßig deaktiviert:* Microsoft dokumentiert nur, dass diese Metriken „disabled by default" sind, nennt aber keinen expliziten Grund. Naheliegend – aber von mir nicht als gesicherte Tatsache, sondern als plausible Einschätzung markiert – ist, dass die Erhebung dieser Statistiken pro Tabelle und Datenbank zusätzliche Last auf den Systemkatalogen erzeugt und Microsoft sie daher als Opt-in-Feature für Nutzer anbietet, die aktives Autovacuum-Tuning betreiben, statt sie pauschal für alle Server zu erheben.

*Aktivierung:* Server-Parameter `metrics.autovacuum_diagnostics = ON` (dynamisch, kein Neustart nötig). Erfassungsintervall 30 Minuten. Dimension `DatabaseName` ist auf 30 Datenbanken begrenzt (10 bei Burstable-SKU).

**b) PgBouncer-Metriken – was und wozu**

PgBouncer ist ein leichtgewichtiger **Connection Pooler** für PostgreSQL. Er liegt logisch zwischen Anwendung und Datenbank und verwaltet einen Pool bestehender Datenbankverbindungen, die er an eingehende Client-Anfragen weiterreicht, statt für jede Anfrage eine neue physische Verbindung zum PostgreSQL-Server aufzubauen. Das ist relevant, weil jede neue PostgreSQL-Verbindung vergleichsweise teuer ist (eigener Serverprozess, Speicher-Overhead) – bei vielen kurzlebigen Verbindungen (z. B. serverlose Architekturen, Microservices mit vielen Instanzen) kann das den Server stark belasten. Azure Flexible Server bietet PgBouncer als **integriertes, optionales Feature** an.

Die PgBouncer-Metriken (`client_connections_active`, `client_connections_waiting`, `server_connections_active`, `server_connections_idle`, `num_pools`, `total_pooled_connections`) zeigen, ob der Pooler selbst zum Engpass wird – etwa wenn viele Client-Verbindungen auf einen freien Pool-Slot warten müssen (`client_connections_waiting` steigt).

*Was PgBouncer technisch ist:* PgBouncer ist ursprünglich ein eigenständiges, quelloffenes Open-Source-Projekt (nicht von Microsoft entwickelt). Bei Azure Flexible Server handelt es sich um eine **von Microsoft integrierte, mitgelieferte Instanz** dieser Software – kein separates Produkt, das man zusätzlich installieren, verwalten oder patchen müsste. Sie läuft laut Dokumentation auf derselben virtuellen Maschine wie der PostgreSQL-Serverprozess selbst, also nicht als separate, zusätzlich abgerechnete Infrastruktur.

*Kostenpflichtig?* Für die Nutzung des integrierten PgBouncer wird in der Dokumentation **keine separate Gebühr oder eigene SKU genannt** – es ist ein Server-Parameter am bestehenden, bereits bezahlten Flexible-Server, keine zusätzliche Azure-Ressource. Eine explizite Aussage von Microsoft der Form „PgBouncer ist kostenlos" habe ich allerdings nicht gefunden; meine Einschätzung „keine separaten Zusatzkosten" leite ich daraus ab, dass keine eigene abrechenbare Ressource entsteht – das ist eine Schlussfolgerung, keine wörtlich zitierte Garantie. Für eine verbindliche Aussage empfiehlt sich ein Blick in den Azure-Preisrechner bzw. Rückfrage beim Support.

*Was „aktivieren" konkret bedeutet:* Es wird keine neue Software installiert – der PgBouncer-Prozess ist bereits Teil der Server-Infrastruktur, standardmäßig aber inaktiv. „Aktivieren" heißt lediglich, den Server-Parameter `pgbouncer.enabled` im Bereich „Server-Parameter" (Portal, Azure CLI oder Terraform) von `false` auf `true` zu setzen – dynamisch, ohne Neustart. Danach lauscht PgBouncer zusätzlich auf **Port 6432** (statt des normalen PostgreSQL-Ports 5432) unter demselben Hostnamen. Damit eine Anwendung den Pooler tatsächlich nutzt, muss zusätzlich die **Verbindungskonfiguration der Anwendung** von Port 5432 auf 6432 umgestellt werden – reines Aktivieren des Parameters ändert also noch nicht automatisch, wie sich bestehende Anwendungen verbinden. Der direkte Weg über Port 5432 bleibt parallel weiterhin nutzbar.

*Praktische Einschränkungen, die für die Architekturentscheidung relevant sind:*
- Wird **nicht unterstützt auf der Burstable-Recheneinheit** (Compute Tier). Ein Wechsel von General Purpose/Memory Optimized zu Burstable deaktiviert PgBouncer automatisch mit.
- Bei jedem Server-Neustart (Skalierung, HA-Failover, Wartung) wird PgBouncer mit neu gestartet – bestehende gepoolte Verbindungen müssen dann neu aufgebaut werden, genau wie normale Datenbankverbindungen auch.
- Bei zonenredundanten HA-Servern läuft PgBouncer nur auf dem jeweils aktiven Primärserver; nach einem Failover wird es auf dem neu beförderten Server automatisch mit gestartet, der Verbindungsstring der Anwendung bleibt unverändert.

*Warum standardmäßig deaktiviert:* PgBouncer ist selbst ein optionales Feature (nicht jeder Server nutzt es), entsprechend sind auch die zugehörigen Metriken nur relevant und aktivierbar, wenn PgBouncer überhaupt läuft.

*Aktivierung – zweistufig:*
1. `pgbouncer.enabled = ON` (aktiviert den Pooler selbst)
2. `metrics.pgbouncer_diagnostics = ON` (aktiviert die Metriken dazu)

Beide Parameter sind dynamisch. Auch hier gilt das 30-Datenbanken-Limit (10 bei Burstable) für die `DatabaseName`-Dimension.

**c) „Enhanced Metrics" – Begriffsklärung**

„Enhanced Metrics" ist Microsofts Sammelbegriff für eine Gruppe **zusätzlicher, feingranularer Metriken**, die über die im Standard aktiven Infrastruktur-Metriken (CPU, Memory, Storage, IOPS etc.) hinausgehen und tiefere Einblicke je Datenbank oder Session ermöglichen. Autovacuum- und PgBouncer-Metriken lassen sich als Teilmengen dieser erweiterten Metrik-Welt verstehen; der zentrale Freischalt-Parameter für weitere „Enhanced"-Metriken (jenseits von Autovacuum/PgBouncer) ist:

- `metrics.collector_database_activity = ON`

Laut Dokumentation ist ein Teil dieser erweiterten Metriken bereits standardmäßig aktiv, der überwiegende Teil muss aber explizit eingeschaltet werden. Eine vollständige, verbindliche Aufschlüsselung, welche einzelnen Metriken darunterfallen und welche davon per Default an sind, gebe ich hier bewusst nicht wieder, da ich dafür keine abschließend sichere Quelle mit vollständiger Liste identifiziert habe – für eine verbindliche Zuordnung bitte die Microsoft-Referenztabelle „Supported metrics" konsultieren und dort die Spalte für den Default-Status prüfen.

### 2.4 Nutzung/Konsumenten
- **Metrics Explorer** im Portal (Ad-hoc-Analyse, Chart-Overlay mehrerer Metriken)
- **Azure Monitor Alerts** (schwellwertbasiert, z. B. `storage_percent > 85`)
- **Eingebettete Grafana-Dashboards** im Portal: Diese sind laut Dokumentation direkt im Azure-Portal integriert, ohne Zusatzkosten und ohne Einrichtungsaufwand, und visualisieren die Kern-Plattformmetriken; bei aktivierten Diagnostic Settings können sie Metriken und Logs korreliert darstellen.
- **Azure Monitor Workbooks** mit vorgefertigten Templates (u. a. „Enhanced Metrics"-Workbook)
- **Export** einzelner Metriken über Diagnostic Settings (Kategorie `AllMetrics`) in einen Log Analytics Workspace (Tabelle `AzureMetrics`) – das ist der einzige Berührungspunkt zwischen Baustein A und B.

---

## 3. Baustein B: Diagnostic Settings (Logs)

### 3.1 Funktionsweise
Diagnostic Settings sind eine separate Azure-Monitor-Ressource, die pro PostgreSQL-Flexible-Server-Instanz angelegt werden muss. Sie definiert:
1. **Welche Log-Kategorien** exportiert werden,
2. **Wohin** sie exportiert werden (Log Analytics Workspace, Storage Account, Event Hub, Partnerlösung),
3. optional zusätzlich den Export der Plattformmetriken (`AllMetrics`, siehe 2.3).

Ohne aktive Diagnostic Setting werden **keine** Logs erfasst – dies ist eine bewusste Opt-in-Architektur.

### 3.2 Verfügbare Log-Kategorien

| Kategorie (interner Name) | Anzeigename | Inhalt |
|---|---|---|
| `PostgreSQLLogs` | PostgreSQL Server Logs | Klassisches PostgreSQL-Server-Log (z. B. Fehler, Verbindungen, je nach `log_*`-Serverparametern) |
| `PostgreSQLFlexSessions` | PostgreSQL Sessions data | Sitzungsbezogene Daten |
| `PostgreSQLFlexQueryStoreRuntime` | PostgreSQL Query Store Runtime | Laufzeitstatistiken einzelner Queries (Query Store muss aktiviert sein) |
| `PostgreSQLFlexQueryStoreWaitStats` | PostgreSQL Query Store Wait Statistics | Wartestatistiken je Query |
| `PostgreSQLQueryStoreSqlText` | PostgreSQL Query Store SQL Text | SQL-Text zu den Query-Store-Einträgen |
| `PostgreSQLFlexTableStats` | PostgreSQL Autovacuum and schema statistics | Tabellen-/Schema-Statistiken |
| `PostgreSQLFlexDatabaseXacts` | PostgreSQL remaining transactions | Transaction-ID-Wraparound-relevante Daten |
| `PostgreSQLFlexPGBouncer` | PostgreSQL PgBouncer Logs | PgBouncer-Log-Einträge |
| (Metrics-Kategorie) `AllMetrics` | – | Export der unter Kapitel 2 beschriebenen Plattformmetriken in dasselbe Ziel |

Diese Liste stammt aus der aktuellen Microsoft-Referenzdokumentation (Stand des Abrufs: Juli 2026) und kann sich mit neuen Server-Features (z. B. neue PgBouncer- oder Query-Store-Funktionen) ändern.

### 3.3 Zielspeicher und Tabellenmodell

Bei Ziel **Log Analytics Workspace** gibt es zwei grundsätzlich unterschiedliche „Collection Modes", zwischen denen man sich beim Anlegen der Diagnostic Setting entscheidet. Das ist keine PostgreSQL-Spezifität, sondern ein Azure-Monitor-weites Konzept, das für alle Ressourcentypen gilt, die es unterstützen.

**a) „Azure diagnostics" (Legacy-Modell)**
Alle Log-Kategorien aller Ressourcen, die dieses Modell nutzen, landen in **einer einzigen, generischen Tabelle** namens `AzureDiagnostics`. Diese Tabelle hat ein sehr generisches Spaltenschema (u. a. eine Spalte `Category`, über die man die eigentliche Log-Kategorie – z. B. `PostgreSQLLogs` – herausfiltern muss) plus diverse, je nach Ressourcentyp unterschiedlich befüllte Freitext-/JSON-Spalten. Praktisch bedeutet das: Man muss beim Abfragen immer zuerst nach `Category` filtern, und die eigentlichen Nutzdaten liegen oft in einer generischen Spalte, die man erst parsen muss.

**b) „Resource specific" (auch „Dedicated" genannt, empfohlenes Modell)**
Jede Log-Kategorie bekommt ihre **eigene, sprechend benannte Tabelle** mit einem fest definierten, spezifischen Spaltenschema – für PostgreSQL Flexible Server z. B.:
- `PostgreSQLLogs` → Tabelle `PGSQLServerLogs`
- `PostgreSQLFlexTableStats` → Tabelle `PGSQLAutovacuumStats`
- (weitere Kategorien analog, jeweils mit eigenem Tabellennamen)

Vorteile laut Dokumentation: klar strukturierte Spalten ohne Generic-Parsing, schnellere und einfachere KQL-Abfragen, und die Möglichkeit, pro Tabelle unterschiedliche Aufbewahrungsfristen im selben Log Analytics Workspace zu setzen (z. B. Server-Logs 30 Tage, Audit-relevante Kategorien länger). Aus diesem Grund empfiehlt Microsoft „Resource specific" explizit als Standardwahl für neue Implementierungen; das Legacy-Modell `AzureDiagnostics` gilt als in Auslauf befindlich (laut Community-/Blogquellen als „wird langfristig abgelöst" beschrieben – für eine offizielle, verbindliche Abkündigungs-Timeline habe ich allerdings keine gesicherte Microsoft-Quelle gefunden, das ist daher nicht als bestätigte Tatsache zu verstehen).

Technisch wird die Wahl über den Parameter **`log_analytics_destination_type`** gesteuert: Wert `Dedicated` = Resource specific, `AzureDiagnostics` (bzw. kein Wert) = Legacy-Modell. Dieser Parameter wirkt sich **ausschließlich** auf das Ziel „Log Analytics Workspace" aus – bei Storage Account oder Event Hub gibt es dieses Konzept nicht in der gleichen Form.

- Bei Ziel **Event Hub**: für Streaming an externe SIEM-/Log-Management-Lösungen (z. B. Splunk, Sumo Logic) oder eigene Verarbeitung.
- Bei Ziel **Storage Account**: für kostengünstige Langzeitarchivierung/Compliance, keine native Abfragefunktion.

**Terraform-Unterstützung:** Ja, das lässt sich vollständig über Terraform steuern. Die Ressource `azurerm_monitor_diagnostic_setting` besitzt das optionale Attribut `log_analytics_destination_type`, das genau diesen Azure-API-Parameter abbildet:

```hcl
resource "azurerm_monitor_diagnostic_setting" "pg" {
  name                       = "pg-diagnostics"
  target_resource_id         = azurerm_postgresql_flexible_server.this.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.this.id
  log_analytics_destination_type = "Dedicated"   # = "Resource specific"

  enabled_log {
    category = "PostgreSQLLogs"
  }
  enabled_metric {
    category = "AllMetrics"
  }
}
```

Zwei praxisrelevante Einschränkungen dazu, die in öffentlichen GitHub-Issues zum `azurerm`-Provider dokumentiert sind:
- Das Attribut **wirkt nur, wenn gleichzeitig ein `log_analytics_workspace_id` gesetzt ist** – ohne Log-Analytics-Ziel hat es keine Wirkung.
- Bei manchen Azure-Ressourcentypen (in den Issues wird u. a. Key Vault genannt) gibt die Azure-API für dieses Feld inkonsistente oder leere Werte zurück, was in Terraform zu wiederkehrendem Plan-„Drift" führen kann (das Feld erscheint bei jedem `terraform plan` als vermeintliche Änderung, obwohl sich nichts geändert hat). Für PostgreSQL Flexible Server ist mir dieses spezifische Verhalten **nicht mit Sicherheit bekannt** – ich habe keine Quelle gefunden, die das Verhalten für genau diesen Ressourcentyp bestätigt oder ausschließt. Falls in der Praxis ein solcher Drift auftritt, ist der dokumentierte Workaround, das Attribut per `lifecycle { ignore_changes = [log_analytics_destination_type] }` von der Drift-Erkennung auszunehmen.

### 3.4 Abfrage (Beispiel KQL, Resource-specific-Tabelle)
```kql
PGSQLServerLogs
| where LogicalServerName == "example-flexible-server"
| where TimeGenerated > ago(1d)
```
Bei „Azure diagnostics"-Modell äquivalent über `AzureDiagnostics` mit Filter `Category == "PostgreSQLLogs"`.


### 3.5 Nutzung/Konsumenten
- **Log Analytics / KQL** für Ad-hoc-Diagnose, Reports, Alerts auf Log-Basis
- **Troubleshooting Guides** im Portal (sechs vordefinierte Problemfelder, z. B. hohe CPU, hohe IOPS): Diese benötigen laut Dokumentation zwingend Diagnostic Settings mit Ziel Log Analytics Workspace, zusätzlich Query Store und Enhanced Metrics als Datenquellen.
- **Workbooks**, die Metriken und Logs kombiniert darstellen
- **Externe SIEM/Log-Management-Systeme** über Event Hub (z. B. für Security-/Audit-Auswertungen, Compliance)

### 3.6 Abgrenzung: „Server Logs"-Feature vs. Diagnostic Settings
Zusätzlich zu Diagnostic Settings gibt es im Portal unter „Server-Parameter/Server-Logs" eine eigene, davon unabhängige Funktion: Rohe Log-Dateien können direkt am Server aktiviert und heruntergeladen werden (Portal oder Azure CLI). Diese Rohdateien haben eine **sehr kurze lokale Aufbewahrung von 1 bis 7 Tagen** und sind primär für schnelle, manuelle Diagnose gedacht – nicht als Ersatz für eine systematische Log-Pipeline über Diagnostic Settings.

---

## 4. Architekturübersicht (Textdiagramm)

```
                         ┌───────────────────────────────────────────┐
                         │  Azure Database for PostgreSQL             │
                         │  Flexible Server                           │
                         │  (Microsoft.DBforPostgreSQL/flexibleServers)│
                         └───────────────┬─────────────────┬─────────┘
                                         │                  │
                     (immer aktiv,      │                  │ (opt-in, muss konfiguriert werden)
                      kein Setup nötig) │                  │
                                         ▼                  ▼
                     ┌───────────────────────────┐   ┌─────────────────────────┐
                     │ Azure Monitor Metrics      │   │ Diagnostic Settings     │
                     │ (Plattformmetriken,        │   │ (Logs + optional        │
                     │  93 Tage Retention,        │   │  AllMetrics-Export)     │
                     │  PT1M/PT30M-Takt)           │   └────────┬────────────────┘
                     └───────────┬─────────────────┘            │
                                 │                               ├──────────────┬───────────────┐
             ┌───────────────────┼──────────────────┐            ▼              ▼               ▼
             ▼                   ▼                  ▼      Log Analytics   Storage Account   Event Hub
      Metrics Explorer    Azure Monitor        Embedded         Workspace    (Archivierung)  (SIEM/Streaming,
      (Ad-hoc-Analyse)    Alerts (Schwellwert)  Grafana-             │                          z. B. Splunk,
                                                Dashboards            ▼                          Sumo Logic)
                                                (Portal)        KQL-Abfragen,
                                                                 Workbooks,
                                                                 Troubleshooting
                                                                 Guides
```

---

## 5. Zusammenfassende Entscheidungslogik für die Architektur

| Anforderung | Empfohlener Pfad |
|---|---|
| CPU/Memory/Storage/IOPS-Überwachung, einfache Schwellwert-Alerts | Plattformmetriken (Baustein A), kein Diagnostic Setting nötig |
| Verfügbarkeits-Monitoring | Metrik `is_db_alive` (Baustein A) |
| Analyse einzelner langsamer Queries, Query-Text | Diagnostic Settings mit Query-Store-Kategorien (Baustein B), zusätzlich Query Store am Server aktivieren |
| Autovacuum-Tuning | Sowohl Autovacuum-Plattformmetriken (Baustein A, muss aktiviert werden) als auch `PostgreSQLFlexTableStats`-Logs (Baustein B) je nach Detailtiefe |
| Security-/Compliance-Audit, Verbindungs-/DDL-/DML-Nachvollziehbarkeit | Diagnostic Settings, Audit-Log-Kategorien, Ziel Log Analytics oder Event Hub für SIEM |
| Langzeitarchivierung über Log-Analytics-Retention hinaus | Diagnostic Settings mit Ziel Storage Account |
| Troubleshooting Guides im Portal nutzen | Diagnostic Settings zwingend erforderlich (Ziel Log Analytics), plus Query Store und Enhanced Metrics |

---

## 6. Empfohlene Mindeststandards: Start vs. Produktivbetrieb

**Wichtiger Hinweis vorab:** Microsoft veröffentlicht keinen offiziellen, verbindlichen „Minimal-Standard" für Logging & Monitoring bei Flexible Server. Was folgt, ist daher **meine fachliche Einschätzung/Empfehlung** auf Basis der in Kapitel 1–5 beschriebenen, dokumentierten Mechanismen – keine von Microsoft vorgegebene Checkliste. Wo ich konkrete Werte nenne (z. B. Schwellwerte), sind das begründete Vorschläge, keine Herstellervorgaben; sie sind je nach Workload anzupassen.

### 6.1 Minimalstandard zum Start (Dev/Test/erste Inbetriebnahme)

Ziel in dieser Phase: möglichst wenig Setup-Aufwand und Zusatzkosten, aber genug Sichtbarkeit, um grobe Probleme nicht zu verpassen.

| Bereich | Empfehlung |
|---|---|
| Plattformmetriken (Baustein A) | Nichts zu konfigurieren – ist automatisch aktiv. Bei Bedarf ad hoc über Metrics Explorer ansehen. |
| Enhanced/Autovacuum/PgBouncer-Metriken | Nicht aktivieren. In der Startphase i. d. R. nicht nötig, spart Aufwand. |
| Diagnostic Settings (Baustein B) | Optional, kann in dieser Phase auch komplett entfallen. Falls doch gewünscht: eine einzige Diagnostic Setting mit Ziel Log Analytics Workspace (kleinste/günstigste Konfiguration), nur Kategorie `PostgreSQLLogs` (Server-Fehlerlog) – Query Store, Sessions, TableStats etc. weglassen. |
| Alerts | Minimal zwei einfache, kostenlose Alerts: `is_db_alive` (Verfügbarkeit) und `storage_percent` (z. B. Schwelle 85 %, da volle Storage zu Schreibsperren führen kann). |
| Server-Log-Inhalt (`log_*`-Parameter) | Defaults belassen. |

### 6.2 Minimalstandard für eine Produktivdatenbank

Das ist aus meiner Sicht die **untere Grenze dessen, was für eine produktiv genutzte Datenbank verantwortbar ist** – nicht die vollständige „Best Practice"-Ausstattung (die in Kapitel 5 skizzierten weitergehenden Szenarien wie Query-Store-Analyse oder SIEM-Anbindung gehen darüber hinaus und sind je nach Anforderung zusätzlich sinnvoll, aber nicht Teil dieses Minimalstandards).

**a) Plattformmetriken (Baustein A) – Alerts mit Action Group**

Alerts ohne Benachrichtigungsziel sind wirkungslos: Es sollte mindestens eine **Action Group** (E-Mail, Teams, PagerDuty o. ä.) konfiguriert sein, an die folgende Alerts gebunden sind:

| Metrik | Grund |
|---|---|
| `is_db_alive` | Basis-Verfügbarkeit |
| `storage_percent` | Volllaufender Storage kann zu Schreibsperren führen – kritisch |
| `cpu_percent` | Anhaltend hohe CPU-Last als Frühindikator für Performance-Probleme |
| `memory_percent` | Speicherdruck, potenzielle OOM-Situationen |
| `active_connections` (relativ zu `max_connections`) | Drohende Verbindungserschöpfung |
| `connections_failed` | Deutet auf Konfigurations- oder Kapazitätsprobleme hin |

**b) Autovacuum-Metriken aktivieren**

`metrics.autovacuum_diagnostics = ON`. Begründung: Bei einer produktiven Datenbank mit laufendem Schreibverkehr ist unbemerktes Table-Bloat bzw. ein sich näherndes Transaction-ID-Wraparound (siehe Kapitel 2.3) ein reales Betriebsrisiko, das ohne diese Metriken erst spät sichtbar wird. Ich stufe das als **Minimalstandard**, nicht als optionales Extra ein.

**c) PgBouncer-Metriken – bedingt**

Nur relevant, falls PgBouncer tatsächlich genutzt wird (`pgbouncer.enabled = ON`). Wenn ja: `metrics.pgbouncer_diagnostics = ON` ebenfalls als Minimalstandard, da sonst ein saturierter Pool unbemerkt bleibt.

**d) Diagnostic Settings (Baustein B) – minimal produktiv-tauglicher Umfang**

- Ziel: **Log Analytics Workspace**, Collection Mode **„Resource specific" (`Dedicated`)** (siehe Kapitel 3.3) – der Mehraufwand gegenüber „Azure diagnostics" ist beim Einrichten praktisch null, der Nutzen bei der späteren Abfrage aber deutlich höher.
- Minimal sinnvolle Log-Kategorien:
  - `PostgreSQLLogs` (Server-Fehler-/Verbindungslog) – Basis-Diagnose
  - `PostgreSQLFlexDatabaseXacts` (Transaction-ID-/Wraparound-Frühwarnung) – ergänzt die Autovacuum-Metriken um die konkrete Wraparound-Distanz
- **Nicht** zwingend Teil des Minimalstandards, aber naheliegende nächste Ausbaustufe: Query-Store-Kategorien (`PostgreSQLFlexQueryStoreRuntime`, `PostgreSQLFlexQueryStoreWaitStats`, `PostgreSQLQueryStoreSqlText`) für Performance-Diagnose einzelner Queries, sowie `PostgreSQLFlexSessions`. Diese würde ich als „empfehlenswert, aber nicht minimal" einordnen.
- `AllMetrics`-Export in denselben Workspace: sinnvoll, sobald Trendanalysen über mehr als 30 Tage gebraucht werden (siehe Kapitel 2.1); für den reinen Minimalstandard nicht zwingend.

**e) Server-Log-Inhalt (`log_*`-Parameter)**

Für eine produktive Datenbank sollten mindestens folgende, in PostgreSQL allgemein gebräuchliche Logging-Parameter gesetzt sein, damit `PostgreSQLLogs` überhaupt aussagekräftigen Inhalt liefert (Hinweis: Das sind Standard-PostgreSQL-Parameter, keine Azure-spezifischen Einstellungen):
- `log_connections` / `log_disconnections` = `ON` – wer verbindet sich wann
- `log_lock_waits` = `ON` – Hinweise auf Sperr-/Deadlock-Situationen
- `log_min_duration_statement` – langsame Queries protokollieren (konkreter Schwellwert ist workload-abhängig; ich nenne hier bewusst keinen pauschalen Millisekundenwert, da das ohne Kenntnis eurer Workload eine unbegründete Zahl wäre)
- `log_checkpoints` = `ON` – Checkpoint-Verhalten, relevant für I/O-Analyse

**f) Troubleshooting Guides**

Da diese laut Kapitel 3.5 zwingend Diagnostic Settings (Log Analytics) sowie Query Store voraussetzen, sind sie **kein Bestandteil des Minimalstandards**, sondern setzen bereits eine über den Minimalstandard hinausgehende Konfiguration voraus – wer sie nutzen möchte, muss die Query-Store-Kategorien aus Punkt (d) mit einrichten.

### 6.3 Kurzer Vergleich

| | Start-Minimalstandard | Produktiv-Minimalstandard |
|---|---|---|
| Diagnostic Settings | optional | verpflichtend (Log Analytics, Resource specific) |
| Log-Kategorien | ggf. nur `PostgreSQLLogs` | `PostgreSQLLogs` + `PostgreSQLFlexDatabaseXacts` |
| Autovacuum-Metriken | aus | an |
| Alerts | 2 (Verfügbarkeit, Storage) | 6 (siehe Tabelle 6.2a), mit Action Group |
| `log_*`-Serverparameter | Default | angepasst (Connections, Lock Waits, langsame Queries, Checkpoints) |

---

## 7. Konfiguration als Code: Terraform

**Grundsätzliche Antwort: Ja**, sämtliche in dieser Architektur beschriebenen Bausteine lassen sich über den `azurerm`-Provider (HashiCorp Terraform bzw. OpenTofu) deklarativ verwalten. Es gibt keinen Bestandteil dieser Architektur, der zwingend manuell im Portal konfiguriert werden müsste.

### 7.1 Relevante Terraform-Ressourcen

| Zweck | Terraform-Ressource | Bemerkung |
|---|---|---|
| PostgreSQL-Flexible-Server selbst | `azurerm_postgresql_flexible_server` | Basisressource (SKU, Storage, HA, Netzwerk, Backup) |
| Server-Parameter / Feature-Flags (die in Kapitel 2.3 genannten Schalter) | `azurerm_postgresql_flexible_server_configuration` | Ein Ressourcenblock je Parameter, z. B. `metrics.autovacuum_diagnostics`, `metrics.pgbouncer_diagnostics`, `pgbouncer.enabled`, `metrics.collector_database_activity`, sowie die `log_*`-Parameter, die den Inhalt von `PostgreSQLLogs` steuern |
| Diagnostic Settings (Baustein B) | `azurerm_monitor_diagnostic_setting` | Über `enabled_log { category = "..." }`-Blöcke je Log-Kategorie und `enabled_metric { category = "AllMetrics" }` für den Metrik-Export |
| Log Analytics Workspace als Ziel | `azurerm_log_analytics_workspace` | Inkl. `retention_in_days` für die Aufbewahrung jenseits der 93-Tage-Grenze aus Kapitel 2.1 |
| Metrikbasierte Alerts | `azurerm_monitor_metric_alert` | Für Schwellwert-Alerts auf Plattformmetriken (Baustein A) |
| Log-/KQL-basierte Alerts | `azurerm_monitor_scheduled_query_rules_alert_v2` | Für Alerts auf Basis von KQL-Abfragen gegen Log Analytics (Baustein B) |

Beispielhafter Ausschnitt (vereinfacht, zur Orientierung – kein vollständiges, produktionsreifes Modul):

```hcl
resource "azurerm_postgresql_flexible_server_configuration" "autovacuum_metrics" {
  name      = "metrics.autovacuum_diagnostics"
  server_id = azurerm_postgresql_flexible_server.this.id
  value     = "ON"
}

resource "azurerm_monitor_diagnostic_setting" "pg" {
  name                       = "pg-diagnostics"
  target_resource_id         = azurerm_postgresql_flexible_server.this.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.this.id

  enabled_log {
    category = "PostgreSQLLogs"
  }
  enabled_log {
    category = "PostgreSQLFlexTableStats"
  }
  enabled_metric {
    category = "AllMetrics"
  }
}
```

### 7.2 Bekannte Einschränkungen und Fallstricke (dokumentiert, nicht spekulativ)

- **Nebenläufige Änderungen an Server-Parametern:** In der Praxis wurde in einem öffentlichen GitHub-Issue zum `azurerm`-Provider berichtet, dass beim gleichzeitigen Setzen mehrerer `azurerm_postgresql_flexible_server_configuration`-Ressourcen über eine `for_each`-Schleife sporadisch ein `ServerBusy`-Fehler der Azure-API auftritt. Das ist kein grundsätzliches Terraform-Problem, sondern ein bekanntes, gemeldetes Verhalten bei parallelen Konfigurationsänderungen an derselben Server-Instanz. Empfehlenswert ist, entweder Wiederholungslogik (Retries) einzuplanen oder die Parameteränderungen zu serialisieren (z. B. über `depends_on`).
- **Fertige Module:** Es existieren sowohl ein von Microsoft mitgetragenes Azure-Verified-Module (`Azure/terraform-azurerm-avm-res-dbforpostgresql-flexibleserver`) als auch Community-Module (z. B. von Claranet), die Server, Konfiguration und Diagnostic Settings gebündelt bereitstellen. Ich habe diese Module nicht im Detail auf Vollständigkeit geprüft; vor Produktivnutzung empfiehlt sich ein eigener Blick in die jeweilige Modul-Dokumentation.

### 7.3 Was ich dazu nicht mit Sicherheit sagen kann

- Ob **jede einzelne** in Kapitel 3.2 gelistete Log-Kategorie und jeder Enhanced-Metrics-Parameter zum aktuellen Zeitpunkt bereits vollständig und ohne Provider-Lücken über `azurerm` abbildbar ist, kann ich nicht abschließend bestätigen – neue Azure-Features (insbesondere Preview-Funktionen wie `tps` oder `logical_replication_slot_sync_status`) werden erfahrungsgemäß mit einiger Verzögerung im Provider nachgezogen. Dazu habe ich keine vollständig gesicherte, aktuelle Information; im Zweifel vor der Planung die Provider-Dokumentation bzw. das Änderungsprotokoll (Changelog) des `azurerm`-Providers prüfen.
- Konkrete Versionsanforderungen an den `azurerm`-Provider für einzelne der genannten Parameter nenne ich hier bewusst nicht, da sich Mindestversionen häufig ändern und ich das nicht zuverlässig aktuell verifizieren kann.

### 7.4 Anpassung bei Private Endpoint / AMPLS

**Zentrale Erkenntnis:** Der Diagnostic-Settings-Traffic (PostgreSQL-Flexible-Server → Log Analytics Workspace) läuft laut Microsoft-Dokumentation über einen „secure private Microsoft channel" und wird von den AMPLS-Zugriffsmodi (`ingestionAccessMode`/`queryAccessMode`) **nicht** kontrolliert. Die Diagnostic Setting selbst (Kapitel 7.1) bleibt deshalb **unverändert** – sie zeigt weiterhin direkt auf `log_analytics_workspace_id`, unabhängig davon, ob Private Endpoint/AMPLS im Einsatz sind.

Relevant wird die private Netzwerk-Architektur nur für die **Abfrage-Seite**: KQL-Abfragen, Workbooks und Troubleshooting Guides sollen ebenfalls privat laufen, statt über die öffentliche Internetanbindung des Log Analytics Workspace. Dafür ist eine zusätzliche Ressource nötig – die Aufnahme des Workspace als „Scoped Service" in die bestehende AMPLS.

**Betroffen, aber nicht Teil dieses Logging-Moduls:**

| Ressource | Betroffen? | Anpassung nötig? |
|---|---|---|
| `azurerm_postgresql_flexible_server` (der DB-Server mit eigenem Private Endpoint) | Netzwerktechnisch ja, für Logging **nein** | Keine Änderung nötig – die Diagnostic Setting funktioniert unabhängig vom Networking-Modus des Servers. **Unabhängiger Nebenhinweis:** Laut Dokumentation werden Private Endpoints aktuell **nicht unterstützt bei Servern, die mit VNet-Integration (delegiertes Subnetz) erstellt wurden** – nur bei Servern mit Networking-Modus „Public access" (öffentlicher Zugriff deaktiviert) plus zusätzlichem Private Endpoint. Falls der Server ursprünglich mit VNet-Integration angelegt wurde, ist reiner Private-Endpoint-Betrieb nur über eine laut den gefundenen Quellen noch als Preview markierte Migration möglich – unabhängig vom Logging-Thema zu prüfen. |
| `azurerm_private_endpoint` (Ziel: AMPLS, Subresource `azuremonitor`) | Ja, aber vermutlich bereits vorhanden | Da die AMPLS laut Aussage bereits im Einsatz ist, gehe ich davon aus, dass der zugehörige Private Endpoint schon existiert. Falls nicht: Diese Ressource fehlt dann noch und wurde hier bewusst nicht neu angelegt, um keine Dopplung zu riskieren. |

**Vollständiges Codebeispiel: Minimaler Produktivstandard (Kapitel 6.2) mit Private-Endpoint-/AMPLS-Anpassung**

```hcl
############################################
# Variablen
############################################

variable "enable_pgbouncer" {
  type    = bool
  default = false
  # Nur auf true setzen, wenn PgBouncer tatsaechlich genutzt wird (Kapitel 6.2c)
}

variable "log_min_duration_statement_ms" {
  type        = number
  default     = 1000
  description = "Schwellwert in ms fuer 'langsame Queries'. Workload-abhaengig, unbedingt anpassen -- 1000 ms ist nur ein Startwert, kein Microsoft-Vorgabewert."
}

variable "storage_alert_threshold_percent" {
  type    = number
  default = 85
}

variable "cpu_alert_threshold_percent" {
  type    = number
  default = 90
}

variable "memory_alert_threshold_percent" {
  type    = number
  default = 90
}

variable "active_connections_alert_threshold" {
  type        = number
  default     = 0
  description = "Absoluter Schwellwert fuer active_connections. MUSS manuell anhand von max_connections der gewaehlten SKU gesetzt werden -- 0 ist ein Platzhalter, kein Produktivwert."
}

############################################
# Datenquellen (vorausgesetzt vorhanden)
############################################

data "azurerm_postgresql_flexible_server" "this" {
  name                = "mein-pg-server"
  resource_group_name = "rg-database"
}

data "azurerm_monitor_private_link_scope" "this" {
  name                = "ampls-shared"
  resource_group_name = "rg-networking-shared"
}

############################################
# 1) Log Analytics Workspace
############################################

resource "azurerm_log_analytics_workspace" "pg" {
  name                = "log-pg-prod"
  location            = "westeurope"
  resource_group_name = "rg-monitoring"
  sku                 = "PerGB2018"
  retention_in_days   = 30

  internet_ingestion_enabled = true   # siehe Unsicherheitshinweis unten
  internet_query_enabled     = false  # Abfragen nur noch privat ueber die AMPLS
}

############################################
# 2) Diagnostic Setting -- unveraendert durch Private Endpoint/AMPLS
############################################

resource "azurerm_monitor_diagnostic_setting" "pg" {
  name                            = "pg-diagnostics-prod"
  target_resource_id              = data.azurerm_postgresql_flexible_server.this.id
  log_analytics_workspace_id      = azurerm_log_analytics_workspace.pg.id
  log_analytics_destination_type  = "Dedicated"

  enabled_log {
    category = "PostgreSQLLogs"
  }
  enabled_log {
    category = "PostgreSQLFlexDatabaseXacts"
  }
  enabled_metric {
    category = "AllMetrics"
  }
}

############################################
# 3) NEU wegen AMPLS: Workspace als Scoped Service eintragen
############################################

resource "azurerm_monitor_private_link_scoped_service" "pg_law" {
  name                = "pg-law-scoped-service"
  resource_group_name = "rg-networking-shared"
  scope_name          = data.azurerm_monitor_private_link_scope.this.name
  linked_resource_id  = azurerm_log_analytics_workspace.pg.id
}

############################################
# 4) Server-Parameter: Autovacuum-Metriken (Kapitel 6.2b)
############################################

resource "azurerm_postgresql_flexible_server_configuration" "autovacuum_metrics" {
  name      = "metrics.autovacuum_diagnostics"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "ON"
}

############################################
# 5) Server-Parameter: log_*-Einstellungen (Kapitel 6.2e)
############################################

resource "azurerm_postgresql_flexible_server_configuration" "log_connections" {
  name      = "log_connections"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_disconnections" {
  name      = "log_disconnections"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_lock_waits" {
  name      = "log_lock_waits"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_checkpoints" {
  name      = "log_checkpoints"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_min_duration_statement" {
  name      = "log_min_duration_statement"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = tostring(var.log_min_duration_statement_ms)
}

############################################
# 6) PgBouncer -- bedingt (Kapitel 6.2c), nur falls genutzt
############################################

resource "azurerm_postgresql_flexible_server_configuration" "pgbouncer_enabled" {
  count     = var.enable_pgbouncer ? 1 : 0
  name      = "pgbouncer.enabled"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "true"
}

resource "azurerm_postgresql_flexible_server_configuration" "pgbouncer_metrics" {
  count     = var.enable_pgbouncer ? 1 : 0
  name      = "metrics.pgbouncer_diagnostics"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "ON"

  depends_on = [azurerm_postgresql_flexible_server_configuration.pgbouncer_enabled]
}

############################################
# 7) Action Group
############################################

resource "azurerm_monitor_action_group" "pg" {
  name                = "ag-pg-prod"
  resource_group_name = "rg-monitoring"
  short_name          = "pg-prod"

  email_receiver {
    name          = "ops-team"
    email_address = "ops@example.com"
  }
}

############################################
# 8) Alerts -- alle 6 aus Kapitel 6.2a
#    Aggregation-Werte vor dem Apply gegen die Microsoft-Referenz
#    "Supported metrics" pruefen (Kapitel 2.2).
############################################

resource "azurerm_monitor_metric_alert" "is_db_alive" {
  name                = "alert-pg-availability"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 0
  frequency           = "PT1M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "is_db_alive"
    aggregation      = "Average"
    operator         = "LessThan"
    threshold        = 1
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "storage" {
  name                = "alert-pg-storage"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 1
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "storage_percent"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.storage_alert_threshold_percent
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "cpu" {
  name                = "alert-pg-cpu"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "cpu_percent"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.cpu_alert_threshold_percent
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "memory" {
  name                = "alert-pg-memory"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "memory_percent"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.memory_alert_threshold_percent
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "active_connections" {
  name                = "alert-pg-active-connections"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "active_connections"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.active_connections_alert_threshold
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }

  # Hinweis: absoluter Schwellwert, keine echte Ratio
  # active_connections/max_connections -- ein Standard-Metrik-Alert
  # kann keine Ratio zwischen zwei Metriken bilden. Fuer eine echte
  # relative Schwelle waere ein log-/KQL-basierter Alert
  # (azurerm_monitor_scheduled_query_rules_alert_v2) noetig -- das
  # geht ueber den hier definierten Minimalstandard hinaus.
}

resource "azurerm_monitor_metric_alert" "connections_failed" {
  name                = "alert-pg-connections-failed"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "connections_failed"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = 5
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}
```

**Unsicherheitspunkt, der bewusst nicht verschwiegen wird:** Für `internet_ingestion_enabled` wurde hier `true` gewählt statt `false`. Es gibt eine klare Quelle dafür, dass die **AMPLS-Zugriffsmodi** die Diagnostic-Settings-Ingestion nicht beeinflussen. Ob das Setzen von `internet_ingestion_enabled = false` **direkt am Workspace** (eine andere, workspace-eigene Netzwerkeinstellung) die Diagnostic-Settings-Ingestion ebenfalls unberührt lässt, konnte ich **nicht mit einer eindeutigen Quelle bestätigen**. Vor einer vollständigen Umstellung auf privat sollte das in einer Testumgebung verifiziert werden.

---

## 8. Offene Punkte / bewusst nicht beantwortet

Zu folgenden Aspekten liegen mir **keine gesicherten, aktuellen Informationen** vor und ich habe daher nichts erfunden:

- **Konkrete Kostenmodelle**: Wie viel der Export bestimmter Log-Kategorien über Diagnostic Settings pro GB kostet, hängt vom gewählten Log-Analytics-Tarif, Region und aktuellem Preismodell ab. Dazu habe ich keine gesicherten aktuellen Preisinformationen – bitte den Azure-Preisrechner konsultieren.
- **Genaue Default-Aufbewahrungsdauer eines neu angelegten Log Analytics Workspace** in eurer konkreten Umgebung (abhängig von Tarif/Konfiguration) – dazu kann ich ohne Kenntnis eurer Workspace-Konfiguration keine verbindliche Aussage treffen.
- **Verhalten/Vollständigkeit bei sehr neuen Preview-Metriken** (z. B. `tps`, `bloat_percent`, `logical_replication_slot_sync_status`) – diese sind als Preview gekennzeichnet und können sich in Verhalten oder Verfügbarkeit noch ändern.
- **Terraform-Provider-Abdeckung im Detail** (siehe Kapitel 7.3).

## 9. Quellen (Microsoft Learn, abgerufen Juli 2026)

- Monitor using Metrics and Logs in Azure Database for PostgreSQL flexible server: https://learn.microsoft.com/en-us/azure/postgresql/monitor/concepts-monitoring
- Configure and access logs: https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs
- Supported metrics – Microsoft.DBforPostgreSQL/flexibleServers: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/microsoft-dbforpostgresql-flexibleservers-metrics
- Supported logs – Microsoft.DBforPostgreSQL/flexibleServers: https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-dbforpostgresql-flexibleservers-logs
- Monitor by using Azure Monitor workbooks: https://learn.microsoft.com/en-us/azure/postgresql/monitor/concepts-workbooks
- Troubleshooting guides – Azure portal: https://docs.azure.cn/en-us/postgresql/flexible-server/how-to-troubleshooting-guides
- Autovacuum Tuning – Azure Database for PostgreSQL: https://learn.microsoft.com/en-us/azure/postgresql/troubleshoot/how-to-autovacuum-tuning
- Terraform-Provider `azurerm_postgresql_flexible_server`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/postgresql_flexible_server
- Azure Verified Module für PostgreSQL Flexible Server (Terraform): https://github.com/Azure/terraform-azurerm-avm-res-dbforpostgresql-flexibleserver
- Bekanntes Issue zu paralleler Konfiguration (`ServerBusy`): https://github.com/hashicorp/terraform-provider-azurerm/issues/27332
- Diagnostic Settings in Azure Monitor (Collection Mode, Resource specific vs. Azure diagnostics): https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings
- Terraform-Ressource `azurerm_monitor_diagnostic_setting`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/monitor_diagnostic_setting
- Bekanntes Issue zu `log_analytics_destination_type`-Drift: https://github.com/hashicorp/terraform-provider-azurerm/pull/20203
- PgBouncer in Azure Database for PostgreSQL flexible server (Funktionsweise, Port, Einschränkungen): https://learn.microsoft.com/en-us/azure/postgresql/connectivity/concepts-pgbouncer
- Design Azure Monitor Private Link configuration (Access Modes, Ausnahme für Diagnostic Settings): https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/private-link-design
- Use Azure Private Link to connect networks to Azure Monitor: https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/private-link-security
- Terraform-Ressource `azurerm_monitor_private_link_scope`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/monitor_private_link_scope
- Terraform-Ressource `azurerm_monitor_private_link_scoped_service`: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/monitor_private_link_scoped_service
- Networking Overview with Private Link Connectivity – Azure Database for PostgreSQL (Einschränkung VNet-Integration vs. Private Endpoint): https://learn.microsoft.com/en-us/azure/postgresql/network/concepts-networking-private-link






############################################
# Variablen
############################################

variable "enable_pgbouncer" {
  type    = bool
  default = false
  # Nur auf true setzen, wenn PgBouncer tatsächlich genutzt wird (Kapitel 6.2c)
}

variable "log_min_duration_statement_ms" {
  type        = number
  default     = 1000
  description = "Schwellwert in ms fuer 'langsame Queries' im PostgreSQLLogs-Log. Workload-abhaengig, unbedingt anpassen -- 1000 ms ist nur ein Startwert, kein Microsoft-Vorgabewert."
}

variable "storage_alert_threshold_percent" {
  type    = number
  default = 85
}

variable "cpu_alert_threshold_percent" {
  type    = number
  default = 90
}

variable "memory_alert_threshold_percent" {
  type    = number
  default = 90
}

variable "active_connections_alert_threshold" {
  type        = number
  default     = 0
  description = "Absoluter Schwellwert fuer active_connections. MUSS manuell anhand von max_connections eurer gewaehlten SKU gesetzt werden (siehe Hinweis unten) -- 0 ist ein Platzhalter, kein sinnvoller Produktivwert."
}

############################################
# Datenquellen (vorausgesetzt vorhanden)
############################################

data "azurerm_postgresql_flexible_server" "this" {
  name                = "mein-pg-server"
  resource_group_name = "rg-database"
}

data "azurerm_monitor_private_link_scope" "this" {
  name                = "ampls-shared"
  resource_group_name = "rg-networking-shared"
}

############################################
# 1) Log Analytics Workspace
############################################

resource "azurerm_log_analytics_workspace" "pg" {
  name                = "log-pg-prod"
  location            = "westeurope"
  resource_group_name = "rg-monitoring"
  sku                 = "PerGB2018"
  retention_in_days   = 30

  internet_ingestion_enabled = true   # siehe Unsicherheitshinweis in Kapitel 7.4
  internet_query_enabled     = false  # Abfragen nur noch privat ueber die AMPLS
}

############################################
# 2) Diagnostic Setting (unveraendert durch Private Endpoint/AMPLS)
############################################

resource "azurerm_monitor_diagnostic_setting" "pg" {
  name                            = "pg-diagnostics-prod"
  target_resource_id              = data.azurerm_postgresql_flexible_server.this.id
  log_analytics_workspace_id      = azurerm_log_analytics_workspace.pg.id
  log_analytics_destination_type  = "Dedicated"

  enabled_log {
    category = "PostgreSQLLogs"
  }
  enabled_log {
    category = "PostgreSQLFlexDatabaseXacts"
  }
  enabled_metric {
    category = "AllMetrics"
  }
}

############################################
# 3) AMPLS-Scoped-Service fuer den Workspace
############################################

resource "azurerm_monitor_private_link_scoped_service" "pg_law" {
  name                = "pg-law-scoped-service"
  resource_group_name = "rg-networking-shared"
  scope_name          = data.azurerm_monitor_private_link_scope.this.name
  linked_resource_id  = azurerm_log_analytics_workspace.pg.id
}

############################################
# 4) Server-Parameter: Autovacuum-Metriken (Kapitel 6.2b)
############################################

resource "azurerm_postgresql_flexible_server_configuration" "autovacuum_metrics" {
  name      = "metrics.autovacuum_diagnostics"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "ON"
}

############################################
# 5) Server-Parameter: log_*-Einstellungen (Kapitel 6.2e)
############################################

resource "azurerm_postgresql_flexible_server_configuration" "log_connections" {
  name      = "log_connections"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_disconnections" {
  name      = "log_disconnections"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_lock_waits" {
  name      = "log_lock_waits"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_checkpoints" {
  name      = "log_checkpoints"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "log_min_duration_statement" {
  name      = "log_min_duration_statement"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = tostring(var.log_min_duration_statement_ms)
}

############################################
# 6) PgBouncer -- bedingt (Kapitel 6.2c), nur falls genutzt
############################################

resource "azurerm_postgresql_flexible_server_configuration" "pgbouncer_enabled" {
  count     = var.enable_pgbouncer ? 1 : 0
  name      = "pgbouncer.enabled"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "true"
}

resource "azurerm_postgresql_flexible_server_configuration" "pgbouncer_metrics" {
  count     = var.enable_pgbouncer ? 1 : 0
  name      = "metrics.pgbouncer_diagnostics"
  server_id = data.azurerm_postgresql_flexible_server.this.id
  value     = "ON"

  depends_on = [azurerm_postgresql_flexible_server_configuration.pgbouncer_enabled]
}

############################################
# 7) Action Group
############################################

resource "azurerm_monitor_action_group" "pg" {
  name                = "ag-pg-prod"
  resource_group_name = "rg-monitoring"
  short_name          = "pg-prod"

  email_receiver {
    name          = "ops-team"
    email_address = "ops@example.com"
  }
}

############################################
# 8) Alerts -- alle 6 aus Kapitel 6.2a
#    Hinweis: aggregation-Werte vor dem Apply gegen die
#    Microsoft-Referenz "Supported metrics" pruefen (Kapitel 2.2) --
#    eine falsche Metrik/Aggregations-Kombination wird von der
#    Azure-API abgelehnt.
############################################

resource "azurerm_monitor_metric_alert" "is_db_alive" {
  name                = "alert-pg-availability"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 0
  frequency           = "PT1M"
  window_size         = "PT5M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "is_db_alive"
    aggregation      = "Average"
    operator         = "LessThan"
    threshold        = 1
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "storage" {
  name                = "alert-pg-storage"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 1
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "storage_percent"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.storage_alert_threshold_percent
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "cpu" {
  name                = "alert-pg-cpu"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "cpu_percent"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.cpu_alert_threshold_percent
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "memory" {
  name                = "alert-pg-memory"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "memory_percent"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.memory_alert_threshold_percent
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}

resource "azurerm_monitor_metric_alert" "active_connections" {
  name                = "alert-pg-active-connections"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "active_connections"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.active_connections_alert_threshold
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }

  # WICHTIGER HINWEIS: Das ist ein absoluter Schwellwert, keine echte
  # Relation "active_connections / max_connections". Ein Standard-
  # Metrik-Alert kann keine Ratio zwischen zwei verschiedenen Metriken
  # bilden. Fuer eine echte relative Schwelle waere ein log-/KQL-
  # basierter Alert (azurerm_monitor_scheduled_query_rules_alert_v2)
  # gegen die exportierten AllMetrics-Daten noetig -- das geht ueber
  # den hier definierten Minimalstandard hinaus.
}

resource "azurerm_monitor_metric_alert" "connections_failed" {
  name                = "alert-pg-connections-failed"
  resource_group_name = "rg-monitoring"
  scopes              = [data.azurerm_postgresql_flexible_server.this.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Microsoft.DBforPostgreSQL/flexibleServers"
    metric_name      = "connections_failed"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = 5
  }

  action {
    action_group_id = azurerm_monitor_action_group.pg.id
  }
}



Hier eine Übersicht, wo du die einzelnen deployten Terraform-Ressourcen im Azure Portal (oder per CLI) wiederfindest:

| Terraform-Ressource | Wo im Portal | Was du dort siehst |
|---|---|---|
| `azurerm_postgresql_flexible_server_configuration` (autovacuum, log_*, pgbouncer) | Dein Postgres-Server → **Einstellungen → Server-Parameter** (Suche nach `metrics.autovacuum_diagnostics`, `log_connections` etc.) | Den aktuell gesetzten Wert – so verifizierst du, dass Terraform die Werte wirklich übernommen hat |
| `azurerm_monitor_diagnostic_setting` | Dein Postgres-Server → **Überwachung → Diagnoseeinstellungen** (Diagnostic settings) | Den Namen deiner Diagnostic Setting (`pg-diagnostics-prod`), die aktivierten Kategorien und das Ziel (dein Log Analytics Workspace) |
| Plattformmetriken (automatisch, unabhängig von Terraform) | Dein Postgres-Server → **Überwachung → Metriken** (Metrics Explorer) | `storage_percent`, `cpu_percent`, `is_db_alive` etc. sofort; Autovacuum-Metriken (`n_dead_tup_user_tables` etc.) erst nach bis zu 30 Minuten, da 30-Minuten-Erfassungsintervall |
| Log-Daten (`PostgreSQLLogs`, `PostgreSQLFlexDatabaseXacts`) | Log Analytics Workspace (`log-pg-prod`) → **Protokolle** (Logs, KQL-Editor) | Die Tabelle `PGSQLServerLogs` (bestätigt, siehe Kapitel 3.3/3.4 im Dokument) für die Server-Logs. **Für `PostgreSQLFlexDatabaseXacts` kenne ich den exakten Tabellennamen nicht mit Sicherheit** – schau im Workspace links unter **Tabellen** nach Einträgen mit Präfix `PGSQL`, dort findest du den tatsächlichen Namen, statt dass ich ihn hier rate. |
| `AllMetrics`-Export | Log Analytics Workspace → **Protokolle** | Tabelle `AzureMetrics` – dieselben Werte wie im Metrics Explorer, aber mit Historie über 30 Tage hinaus |
| `azurerm_monitor_metric_alert` (alle 6) | Dein Postgres-Server → **Überwachung → Warnungen** oder global unter **Monitor → Warnungen** | Regelname, Status (aktiv/ausgelöst), letzte Auswertung |
| `azurerm_monitor_action_group` | **Monitor → Warnungen → Aktionsgruppen** | Die konfigurierte E-Mail-Adresse; hier testest du am besten sofort per „Test-Benachrichtigung senden", ob die Zustellung funktioniert |
| `azurerm_monitor_private_link_scoped_service` | Deine AMPLS-Ressource → **Konfigurieren → Azure Monitor-Ressourcen** | Deinen Log Analytics Workspace als verknüpfte Ressource in der Liste |
| `internet_query_enabled = false` am Workspace | Log Analytics Workspace → **Netzwerkzugriffskonfiguration** | Ob "Öffentlicher Netzwerkzugriff für Abfragen" auf "Deaktiviert" bzw. "Nur ausgewählte Netzwerke" steht |

**Praktischer erster Check-Ablauf, den ich empfehlen würde:**
1. Server-Parameter prüfen (sind die `ON`/`on`-Werte wirklich angekommen?)
2. Diagnoseeinstellungen-Blade am Server öffnen (ist die Setting da, zeigt sie „Aktiv"?)
3. Ein paar Minuten warten, dann im Log Analytics Workspace `PGSQLServerLogs | take 10` ausführen – kommt überhaupt Log-Content an?
4. Metrics Explorer öffnen, `is_db_alive` und `storage_percent` als Chart anzeigen
5. In den Aktionsgruppen eine Testbenachrichtigung auslösen, um die Zustellkette zu verifizieren

Ein Hinweis zur Erwartungshaltung bei den Logs: Zwischen Aktivierung der Diagnostic Setting und dem ersten sichtbaren Logeintrag in Log Analytics kann es **einige Minuten Verzögerung** geben – das ist normal, ich habe aber keine exakte, garantierte Zeitangabe dafür gefunden, also keine feste Wartezeit als "Fakt" nennen.

Soll ich diesen Verifikations-Ablauf als kurzes neues Kapitel „Nach dem Deployment: Verifikation" ans Dokument anhängen?
