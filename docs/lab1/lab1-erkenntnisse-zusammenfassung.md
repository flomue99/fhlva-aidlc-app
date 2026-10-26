# SRE3, Lab 1 Bericht - Erkenntnisse und Zusammenfassung
### Gruppe: G2-Meds, health-notification-service
### Florian Bergthaler, Florian Mühleder, Jonas Mühleder
> Kurzpräsentation, max. 5 Minuten. Zeitangaben pro Abschnitt als Orientierung.

Aufgabe war die App lokal (podman compose, minikube) und auf Azure zum Laufen bringen, Probleme notieren und zusammenfassen, was unser Service macht.

---

## 1. Was macht unser Service? (~1 min)

- **health-notification-service** (Java 21, Spring Boot, Port `8083`, eigene DB `notifications_db`)
- Ein **Scheduler** prüft alle 5 min (compose: 5 s) den Konsum jedes Users beim `beer-tracking-service`
- Liegt der Konsum über dem Limit, legt er eine **Warnung** an: `> 2` Drinks/Tag oder `> 14` Drinks in 7 Tagen (UTC)
- Das Frontend holt die Warnungen über eine REST-API (JWT) ab und kann sie bestätigen
- Lebenszyklus: `ACTIVE → RESOLVED → ACKNOWLEDGED`
- Der Service verschickt selbst nichts. Er hängt von `auth-service` (JWKS), `beer-tracking-service` (`X-Internal-Key`) und PostgreSQL ab; ihn selbst ruft kein anderer Service auf

## 2. Deployment-Probleme und Lösungen (~2 min)

Lokal läuft der komplette Stack, sowohl per `podman compose` im Devcontainer als auch auf minikube. Das Azure-Deployment haben wir im Lab 2 gemacht. Im Folgenden eine Aufzählung der aufgetretenen Probleme und wie wir sie gelöst haben.

| Problem | Lösung |
|---|---|
| Podman startet unter Windows nicht | `wsl --update`, Machine neu anlegen (`--rootful`) |
| Container im Devcontainer ohne Netzwerk | `firewall_driver = "iptables"` in `.devcontainer/containers.conf` |
| DNS-Fehler beim Compose-Start nach Reboot | `rm -rf /run/libpod /run/containers/storage /run/containers/networks`, vor einem Reboot `minikube delete` |
| Rate-Limits bei Docker Hub und Maven Central | `podman login`, Services nacheinander bauen (`COMPOSE_PARALLEL_LIMIT=1`) |
| minikube nutzt die falsche Container-Engine | `minikube delete`, dann `--driver=podman` |
| Shell-Skripte laufen nicht unter PowerShell | Git Bash mit vollem Pfad aufrufen |
| Unerwartete Git-Diffs unter Windows | `.gitattributes` + `core.autocrlf input` |

Die meiste Zeit ging nicht in die App, sondern in die lokale Toolchain (WSL ↔ Podman ↔ Devcontainer ↔ minikube). Wer bei mehreren Engines nicht weiß, welche gerade aktiv ist, sucht lange an der falschen Stelle.

## 3. SRE-Befunde im Service-Code (~1,5 min)

Die wichtigsten Risiken aus der Architekturanalyse:

1. **Ein Ausfall des Tracking-Service löst Warnungen auf.** Ist er nicht erreichbar, rechnet der Scheduler mit 0 Drinks und setzt alle offenen Warnungen auf `RESOLVED`. Ohne `INTERNAL_API_KEY` passiert das dauerhaft und ohne Fehlermeldung.
2. **Kein Lock bei mehreren Replicas.** Laut Migration V2 ist ein Advisory Lock vorgesehen, im Code fehlt er. Mehrere Replicas können daher doppelte `ACTIVE`-Warnungen anlegen, die den User danach dauerhaft blockieren.
3. **`RestTemplate` ohne Timeout.** Ein hängender Tracking-Service blockiert den einzigen Scheduler-Thread, ohne dass ein Fehler geloggt wird.
4. **Keine Metriken.** `/actuator/prometheus` ist freigegeben, aber die Prometheus-Dependency fehlt, und der Endpunkt verlangt zusätzlich ein JWT.
5. Kleinere Punkte:
   - Bestätigt ein User eine Warnung, wird sie sofort wieder neu angelegt, solange er weiterhin über dem Limit liegt.
   - Fehlerhafte Eingaben liefern einen Serverfehler (500) statt einer Fehlermeldung „ungültige Anfrage“ (400). Etwa wenn die User-ID im JWT (Feld `sub`) keine gültige UUID ist oder die Paging-Parameter ungültig sind.

## 4. Ausblick Lab 2: SLIs/SLOs (~0,5 min)

- HTTP-Metriken allein reichen nicht. Wenn der Scheduler hängt, liefert die API weiter `200 OK`.
- Vorgeschlagene SLOs (28 Tage):
  - API-Verfügbarkeit **99,5 %**
  - Latenz **95 % ≤ 300 ms**
  - **Aktualität**: letzter erfolgreicher Lauf ≤ 15 min, 99 % der Zeit
  - **Abdeckung**: 99 % der User-Auswertungen mit vollständigen Daten
- Voraussetzung: Prometheus-Dependency, eigene Scheduler-Metriken, Timeouts, und die Auswertung bei Fehlern überspringen statt mit 0 zu rechnen

---

**Kernaussage:** Der Service läuft, aber er fällt **still** aus. Unsere Priorität ist, ihn messbar zu machen und Ausfälle sichtbar statt falsch zu machen.
