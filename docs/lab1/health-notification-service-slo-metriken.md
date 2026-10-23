# Health Notification Service – SLIs, SLOs & Metriken

Dieses Dokument baut auf der [Architekturübersicht](health-notification-service-architektur.md) auf. Es beschreibt:

- welche Kennzahlen (SLIs) für den `health-notification-service` wirklich aussagekräftig sind,
- welche Ziele (SLOs) wir dafür vorschlagen,
- wie man die Kennzahlen mit Prometheus/PromQL bzw. Dynatrace abfragt,
- was im Code und in der Konfiguration fehlt, damit die Metriken überhaupt existieren.

> **Ist-Zustand:** In der `application.yml` ist `prometheus` als Actuator-Endpunkt freigegeben. Die Dependency `micrometer-registry-prometheus` fehlt aber, deshalb existiert `/actuator/prometheus` nicht. Eigene Metriken gibt es keine. Alles in diesem Dokument ist ein **Vorschlag (Soll-Zustand)**.

---

## 1. Begriffe

| Begriff | Bedeutung | Beispiel für diesen Service |
|---|---|---|
| **SLI** (Service Level Indicator) | Messgröße für die Qualität aus Nutzersicht, meist als Verhältnis *gute Ereignisse / alle Ereignisse* | Anteil der API-Requests ohne 5xx |
| **SLO** (Service Level Objective) | Internes Ziel für einen SLI über ein Zeitfenster | 99,5 % der Requests ohne 5xx über 28 Tage |
| **Error Budget** | Erlaubte Fehlermenge: `1 − SLO` | 0,5 % der Requests in 28 Tagen dürfen fehlschlagen |
| **SLA** (Service Level Agreement) | Vertragliche Zusage mit Konsequenzen (z. B. Gutschrift). Immer **lockerer** als das SLO | 99,0 % Verfügbarkeit pro Monat |
| **Burn Rate** | Wie schnell das Error Budget verbraucht wird. 1 heißt: genau am Ende des Fensters ist es aufgebraucht | Burn Rate 14,4 → Budget in rund 2 Tagen weg |

---

## 2. Was muss dieser Service eigentlich leisten?

Der Service hat **zwei unterschiedliche Aufgaben**. Deshalb reichen klassische HTTP-SLIs allein nicht aus:

1. **Warnungen anzeigen und bestätigen** (REST-API, synchron). Hier passen die klassischen SLIs Verfügbarkeit und Latenz.
2. **Übermäßigen Konsum erkennen** (Scheduler, asynchron). Das ist der **eigentliche Zweck** des Service. Wenn der Scheduler hängt oder falsche Daten bekommt, liefert die API weiterhin `200 OK`, nur eben mit veralteten oder falschen Warnungen. Diesen Ausfall sieht **keine** HTTP-Metrik.

Die wichtigsten SLIs sind deshalb die des Schedulers: **Aktualität** (Freshness) und **Korrektheit/Abdeckung** (Coverage).

---

## 3. Vorgeschlagene SLIs & SLOs

| # | SLI | Definition (gut / gesamt) | SLO (28 Tage) | Error Budget |
|---|---|---|---|---|
| 1 | **API-Verfügbarkeit** | Requests auf `/notifications/**` ohne 5xx / alle Requests auf `/notifications/**` | **99,5 %** | 0,5 % der Requests |
| 2 | **API-Latenz** | Requests mit Antwortzeit ≤ 300 ms / alle Requests | **95 %** ≤ 300 ms, **99 %** ≤ 1 s | 5 % bzw. 1 % der Requests |
| 3 | **Auswertungs-Aktualität** | Minuten, in denen der letzte erfolgreiche Scheduler-Lauf ≤ 15 min zurückliegt / alle Minuten | **99 %** | ca. 6,7 h pro 28 Tage |
| 4 | **Auswertungs-Abdeckung** | User-Auswertungen mit vollständigen Daten vom Tracking-Service / alle User-Auswertungen | **99 %** | 1 % der Auswertungen |

Begründung der Zahlen:

- **15 min bei SLI 3:** Das ist das Dreifache des Scheduler-Intervalls (5 min). Ein einzelner langsamer oder ausgefallener Lauf verletzt das SLO also noch nicht.
- **99 % statt 99,9 %:** Der Service ist eine Hinweisfunktion, kein lebenskritisches System. Ein höheres Ziel würde außerdem mehr Replicas, Locking usw. erfordern (siehe Architektur-Risiko #1).
- **SLI 1 zählt nur 5xx als Fehler.** 4xx (401, 403, 404) sind Fehler des Clients, nicht des Service.

### Vorschlag für ein SLA (zur Diskussion)

| SLA-Zusage | Wert | Verhältnis zum SLO |
|---|---|---|
| API-Verfügbarkeit pro Kalendermonat | 99,0 % | lockerer als das SLO (99,5 %) |
| Warnungen spätestens nach | 30 min | lockerer als das SLO (15 min) |

Für eine Studien-App gibt es kein echtes SLA. Das Prinzip lautet: Das SLO schlägt Alarm, **bevor** das SLA verletzt wird.

---

## 4. Benötigte Metriken

### 4.1 Kommen automatisch von Spring Boot / Micrometer (nach Hinzufügen der Dependency)

| Metrik (Prometheus-Name) | Labels (Auswahl) | Verwendet für |
|---|---|---|
| `http_server_requests_seconds_count` / `_sum` / `_bucket` | `uri`, `method`, `status`, `outcome` | SLI 1, SLI 2 |
| `tasks_scheduled_execution_seconds_count` / `_sum` | `code_function`, `outcome` | Dauer der Scheduler-Läufe (ab Spring Boot 3.2 automatisch) |
| `hikaricp_connections_active`, `hikaricp_connections_pending` | `pool` | Engpässe bei der DB |
| `jvm_memory_used_bytes`, `jvm_gc_pause_seconds_*` | `area`, `id` | Ursachenanalyse |
| `up` (setzt Prometheus selbst) | `job`, `instance` | Ist der Pod überhaupt erreichbar? |

> `_bucket`-Werte für SLI 2 gibt es nur, wenn Histogramm-Grenzen konfiguriert sind (siehe §6.2).
>
> `http_client_requests_seconds_*` (ausgehende Aufrufe zum Tracking-Service) erscheint **nicht**, solange das `RestTemplate` mit `new RestTemplate()` statt über `RestTemplateBuilder` erzeugt wird (siehe §6.4).

### 4.2 Eigene Business-Metriken (müssen implementiert werden)

| Metrik (Prometheus-Name) | Typ | Labels | Verwendet für |
|---|---|---|---|
| `notification_evaluation_last_success_timestamp_seconds` | Gauge | – | **SLI 3** (Aktualität) |
| `notification_user_evaluations_total` | Counter | `outcome` = `success` \| `skipped` | **SLI 4** (Abdeckung) |
| `notification_tracking_fetch_failures_total` | Counter | `endpoint` = `users` \| `summary` | Ursachenanalyse (Architektur-Risiko #2) |
| `notification_warnings_created_total` | Counter | `type` | Auffälligkeiten erkennen |
| `notification_warnings_resolved_total` | Counter | `type` | Massen-Auflösung erkennen (Architektur-Risiko #2) |

---

## 5. Abfragen (PromQL)

### SLI 1 – API-Verfügbarkeit

```promql
# Anteil erfolgreicher Requests (letzte 5 min)
sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*", status!~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*"}[5m]))
```

```promql
# SLO-Erfüllung über 28 Tage
sum(increase(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*", status!~"5.."}[28d]))
/
sum(increase(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*"}[28d]))
```

### SLI 2 – API-Latenz

```promql
# Anteil der Requests ≤ 300 ms (benötigt die Histogramm-Grenze 300ms, siehe §6.2)
sum(rate(http_server_requests_seconds_bucket{job="health-notification-service", uri=~"/notifications.*", le="0.3"}[5m]))
/
sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*"}[5m]))
```

```promql
# p95-Latenz je Endpunkt (benötigt percentiles-histogram, siehe §6.2)
histogram_quantile(0.95,
  sum by (le, uri) (rate(http_server_requests_seconds_bucket{job="health-notification-service", uri=~"/notifications.*"}[5m])))
```

### SLI 3 – Auswertungs-Aktualität

```promql
# Sekunden seit dem letzten erfolgreichen Scheduler-Lauf
time() - max(notification_evaluation_last_success_timestamp_seconds{job="health-notification-service"})
```

```promql
# SLO-Erfüllung über 28 Tage: Anteil der Minuten mit Alter ≤ 900 s
avg_over_time(
  ((time() - max(notification_evaluation_last_success_timestamp_seconds{job="health-notification-service"})) <= bool 900)[28d:1m]
)
```

> `max(...)` über alle Pods: Es reicht, wenn **ein** Pod erfolgreich ausgewertet hat. Das passt zur Variante mit verteiltem Lock, bei der nur ein Pod den Job ausführt.

### SLI 4 – Auswertungs-Abdeckung

```promql
sum(rate(notification_user_evaluations_total{job="health-notification-service", outcome="success"}[1h]))
/
sum(rate(notification_user_evaluations_total{job="health-notification-service"}[1h]))
```

### Weitere nützliche Abfragen

```promql
# Dauer eines Scheduler-Laufs (Durchschnitt)
rate(tasks_scheduled_execution_seconds_sum{code_function="evaluateConsumption"}[15m])
/
rate(tasks_scheduled_execution_seconds_count{code_function="evaluateConsumption"}[15m])

# Fehlgeschlagene Aufrufe zum Tracking-Service pro Minute
sum by (endpoint) (rate(notification_tracking_fetch_failures_total[5m])) * 60

# Massen-Auflösung: mehr als 10 Warnungen in 10 min aufgelöst
sum(increase(notification_warnings_resolved_total[10m])) > 10

# Wartende DB-Connections
max(hikaricp_connections_pending{job="health-notification-service"}) > 0
```

---

## 6. Umsetzung im Service

### 6.1 Dependency ([pom.xml](../../services/health-notification-service/pom.xml))

```xml
<!-- Prometheus metrics endpoint -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Die Version wird vom Spring-Boot-Parent verwaltet.

### 6.2 Konfiguration ([application.yml](../../services/health-notification-service/src/main/resources/application.yml))

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      # Feste Bucket-Grenzen für die Latenz-SLIs (le="0.3", le="1.0")
      slo:
        http.server.requests: 300ms,1s
      # Zusätzliche Buckets, damit histogram_quantile() funktioniert
      percentiles-histogram:
        http.server.requests: true
```

### 6.3 Endpunkt für den Scraper freigeben ([SecurityConfig.java](../../services/health-notification-service/src/main/java/com/beertracker/notification/config/SecurityConfig.java))

Aktuell verlangt `/actuator/prometheus` ein JWT. Prometheus und Dynatrace scrapen aber ohne Token. Das sind die Optionen:

- **Einfach:** `.requestMatchers("/actuator/prometheus").permitAll()` ergänzen.
  - Ingress und nginx leiten nur `/notifications` weiter, der Endpunkt ist also von außen nicht erreichbar, nur cluster-intern.
  - Bei docker-compose ist Port 8083 aber direkt auf den Host gemappt (`ports: "8083:8083"`), dort ist der Endpunkt damit offen.
- **Sauberer:** Einen eigenen Management-Port setzen (`management.server.port: 9083`) und nur diesen Port für den Scraper öffnen.

### 6.4 `RestTemplate` über den Builder erzeugen (Client-Metriken und Timeouts)

```java
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
            .setConnectTimeout(Duration.ofSeconds(2))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();
}
```

Damit gibt es automatisch `http_client_requests_seconds_*`. Gleichzeitig ist Architektur-Risiko #3 behoben, weil ein hängender Tracking-Service den Scheduler nicht mehr blockieren kann.

### 6.5 Eigene Metriken im Scheduler (Skizze)

```java
// ConsumptionEvaluationScheduler – zusätzliche Felder
private final AtomicLong lastSuccessEpochSeconds = new AtomicLong(0);
private final Counter evaluationsSuccess;
private final Counter evaluationsSkipped;

// im Konstruktor (MeterRegistry wird von Spring injiziert)
Gauge.builder("notification.evaluation.last.success.timestamp", lastSuccessEpochSeconds, AtomicLong::get)
        .baseUnit("seconds")
        .register(meterRegistry);
this.evaluationsSuccess = Counter.builder("notification.user.evaluations")
        .tag("outcome", "success").register(meterRegistry);
this.evaluationsSkipped = Counter.builder("notification.user.evaluations")
        .tag("outcome", "skipped").register(meterRegistry);

// in evaluateConsumption(): Lauf nur als erfolgreich werten, wenn keine Auswertung übersprungen wurde
if (failures == 0) {
    lastSuccessEpochSeconds.set(Instant.now().getEpochSecond());
}
```

Wichtig ist der Zusammenhang mit **Architektur-Risiko #2**. Heute wird bei einem Fehler im Tracking-Service mit Konsum `0.0` weitergerechnet, und Warnungen werden fälschlich aufgelöst. Die Metrik `outcome="skipped"` ist nur aussagekräftig, wenn der Code in diesem Fall die Auswertung für den User **tatsächlich überspringt**, statt mit 0 weiterzurechnen.

Die Counter `notification.warnings.created` und `notification.warnings.resolved` (jeweils mit Tag `type`) werden analog im `NotificationService` hochgezählt, an den Stellen, an denen heute bereits geloggt wird.

---

## 7. Alarmierung (Multi-Window Burn Rate)

Ein Alarm „Fehlerrate > X %“ ist entweder zu laut oder zu träge. Besser ist es, auf die **Burn Rate** des Error Budgets zu alarmieren, und zwar über zwei Zeitfenster gleichzeitig. Das lange Fenster sorgt für Signifikanz, das kurze dafür, dass der Alarm wieder abklingt, sobald das Problem behoben ist. Das Verfahren stammt aus dem Google SRE Workbook.

| Schwere | Burn Rate | Fenster (lang / kurz) | Bedeutung bei 28-Tage-SLO |
|---|---|---|---|
| Page (sofort) | 14,4 | 1 h / 5 min | 2 % des Budgets in 1 h verbraucht |
| Page | 6 | 6 h / 30 min | 5 % des Budgets in 6 h verbraucht |
| Ticket | 1 | 3 d / 6 h | Budget wird bis zum Fensterende genau aufgebraucht |

Beispiel als Prometheus-Regeln für SLI 1 (SLO 99,5 %, also Fehlerbudget 0,005):

```yaml
groups:
  - name: health-notification-slo
    rules:
      # Recording Rules: Fehlerquote je Fenster
      - record: hns:api_error_ratio:rate5m
        expr: |
          sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*", status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*"}[5m]))
      - record: hns:api_error_ratio:rate1h
        expr: |
          sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*", status=~"5.."}[1h]))
          / sum(rate(http_server_requests_seconds_count{job="health-notification-service", uri=~"/notifications.*"}[1h]))

      # Alert: schneller Budget-Verbrauch
      - alert: HealthNotificationApiErrorBudgetBurn
        expr: |
          hns:api_error_ratio:rate1h > (14.4 * 0.005)
          and
          hns:api_error_ratio:rate5m > (14.4 * 0.005)
        labels:
          severity: page
        annotations:
          summary: "health-notification-service verbraucht Error Budget 14x zu schnell"

      # Alert: Auswertung veraltet (SLI 3) – direkt auf Schwellwert, da Zustand statt Ereignisrate
      - alert: HealthNotificationEvaluationStale
        expr: |
          time() - max(notification_evaluation_last_success_timestamp_seconds{job="health-notification-service"}) > 900
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "Seit über 15 min keine erfolgreiche Konsum-Auswertung"
```

Die Regeln für 6 h / 30 min und 3 d / 6 h werden nach demselben Muster ergänzt.

---

## 8. Umsetzung mit Dynatrace (Lab-Umgebung)

Im Cluster wird der Dynatrace OneAgent injiziert (siehe Kommentar im [Deployment-Template](../../charts/health-notification-service/templates/deployment.yaml)). Daraus ergeben sich zwei Wege:

**a) SLI 1 & 2 ohne Code-Änderung (Built-in-Metriken)**

Der OneAgent erkennt den Spring-Service selbst und misst Requests, Fehler und Antwortzeiten. Ein Dynatrace-SLO für die Verfügbarkeit (klassischer Metric Selector, z. B. über die SLO-Vorlage „Service availability“):

```
(100)*(builtin:service.errors.server.successCount:splitBy())/(builtin:service.requestCount.server:splitBy())
```

Gefiltert wird über die Entity-Auswahl auf den Service `health-notification-service`, idealerweise nur auf die Endpunkte unter `/notifications`. Für die Latenz gibt es `builtin:service.response.time` (Einheit: Mikrosekunden).

**b) SLI 3 & 4 nur mit eigenen Metriken**

Aktualität und Abdeckung des Schedulers kennt Dynatrace nicht von selbst. Die Metriken aus §4.2 müssen über den Prometheus-Endpunkt eingesammelt werden. Dazu bekommt der Pod Annotationen im Deployment-Template:

```yaml
template:
  metadata:
    annotations:
      metrics.dynatrace.com/scrape: "true"
      metrics.dynatrace.com/port: "8083"          # bzw. Management-Port, falls getrennt
      metrics.dynatrace.com/path: "/actuator/prometheus"
```

Zusätzlich muss in den Kubernetes-Einstellungen des Tenants das Scrapen annotierter Prometheus-Exporter aktiviert sein. Danach stehen die Metriken im Data Explorer zur Verfügung, und man kann darauf ein SLO bzw. ein Metric-Event (Alarm) anlegen.

> Die genauen Metrik-Schlüssel nach dem Import sowie die Menüpfade unterscheiden sich je nach Dynatrace-Version (klassisch vs. Grail/DQL). Bitte im Trial-Tenant prüfen.

---

## 9. Grenzen dieser SLIs

- **Serverseitige HTTP-Metriken sehen keine Requests, die den Pod nie erreichen**, z. B. wenn der Pod down ist oder der Ingress einen Fehler hat. Genauer wäre ein SLI am Ingress bzw. nginx oder ein synthetischer Check (Dynatrace Synthetic, Blackbox Exporter). `up == 0` deckt zumindest den Totalausfall ab.
- **SLI 3 misst, ob der Scheduler läuft, nicht ob die Warnungen fachlich richtig sind.** Fachliche Korrektheit (z. B. das rollende Wochenfenster vs. Kalenderwoche, Architektur-Risiko #6) muss über Tests abgesichert werden.
- **Mehrere Replicas ohne Lock** (Architektur-Risiko #1) verfälschen die Zähler `warnings_created_total`, weil Duplikate mitgezählt werden. Der Lock sollte daher vor der Skalierung umgesetzt sein.

## 10. Umsetzungsreihenfolge (Vorschlag)

1. Dependency, Konfiguration und Endpunktfreigabe (§6.1–6.3). Damit sind SLI 1 und 2 sofort messbar.
2. `RestTemplate` mit Timeouts (§6.4).
3. Auswertung bei Fehlern im Tracking-Service überspringen statt mit 0 zu rechnen, dazu die eigenen Metriken (§6.5). Damit werden SLI 3 und 4 messbar.
4. Scraping in Dynatrace bzw. Prometheus einrichten (§8), SLOs anlegen.
5. Burn-Rate-Alarme (§7).
