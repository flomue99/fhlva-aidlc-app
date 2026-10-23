# Lab 1 – Erkenntnisse (Gruppe „Meds“, health-notification-service)

> Kurzpräsentation, max. 5 Minuten. Zeitangaben pro Abschnitt als Orientierung.
> Details: [Architektur](health-notification-service-architektur.md) · [SLIs/SLOs](health-notification-service-slo-metriken.md) · [Setup Insights](setup_insights.md) · [Windows/Podman Troubleshooting](windows-podman-troubleshooting.md)

**Lab-1-Aufgabe:** App lokal (podman compose, minikube) und auf Azure zum Laufen bringen, Probleme notieren und zusammenfassen, was unser Service macht.

---

## 1. Was macht unser Service? (~1 min)

- **health-notification-service** (Java 21, Spring Boot, Port `8083`, eigene DB `notifications_db`)
- Ein **Scheduler** prüft alle 5 min (compose: 5 s) den Konsum jedes Users beim `beer-tracking-service`
- Liegt der Konsum über dem Limit, legt er eine **Warnung** an: `> 2` Drinks/Tag oder `> 14` Drinks in 7 Tagen (UTC)
- Das Frontend holt die Warnungen über eine REST-API (JWT) ab und kann sie bestätigen
- Lebenszyklus: `ACTIVE → RESOLVED → ACKNOWLEDGED`
- Der Service verschickt selbst nichts. Er hängt von `auth-service` (JWKS), `beer-tracking-service` (`X-Internal-Key`) und PostgreSQL ab; ihn selbst ruft kein anderer Service auf

## 2. Deployment-Probleme und Lösungen (~2 min)

| # | Problem | Ursache | Lösung |
|---|---|---|---|
| 1 | Podman-Machine wird nie „running“ / `Wsl/Service/E_UNEXPECTED` | WSL 2.2.4 zu alt für das Fedora-44-Image von Podman 6.1 | `wsl --update`, Machine neu anlegen (`--rootful`) |
| 2 | Container im Devcontainer haben kein Netzwerk | Podman-in-Podman: nftables kollidiert mit der äußeren VM | `firewall_driver = "iptables"` in `.devcontainer/containers.conf` |
| 3 | `aardvark-dns` kann nicht an `192.168.49.1:53` binden (nach Reboot) | Veralteter Podman-State unter `/run` mit verwaistem **minikube**-Netz; aardvark bindet an *alle* Gateways | `rm -rf /run/libpod /run/containers/storage /run/containers/networks`, vor einem Reboot `minikube delete` |
| 4 | `toomanyrequests` (Docker Hub) / `429` (Maven Central) | Rate-Limits pro IP; 4 Spring-Services laden parallel alle Dependencies | `podman login`, Services nacheinander bauen (`COMPOSE_PARALLEL_LIMIT=1`) |
| 5 | minikube nutzt weiter Docker Desktop | Treiber ist pro minikube-Profil gespeichert | `minikube delete`, dann `--driver=podman` |
| 6 | `.sh`-Skripte laufen nicht unter PowerShell | `bash` zeigt auf WSL-Debian ohne minikube/kubectl/helm | Git Bash mit vollem Pfad aufrufen |
| 7 | Phantom-Diffs unter Windows | CRLF/LF-Konvertierung | `.gitattributes` + `core.autocrlf input` |

**Gelernt:** Die meiste Zeit ging nicht in die App, sondern in die lokale Toolchain (WSL ↔ Podman ↔ Devcontainer ↔ minikube). Wer bei mehreren Engines nicht weiß, welche gerade aktiv ist, sucht lange an der falschen Stelle.

## 3. SRE-Befunde im Service-Code (~1,5 min)

Die wichtigsten Risiken aus der Architekturanalyse:

1. **Ein Ausfall des Tracking-Service löst Warnungen auf.** Ist er nicht erreichbar, rechnet der Scheduler mit 0 Drinks und setzt alle offenen Warnungen auf `RESOLVED`. Ohne `INTERNAL_API_KEY` passiert das dauerhaft und ohne Fehlermeldung.
2. **Kein Lock bei mehreren Replicas.** Laut Migration V2 ist ein Advisory Lock vorgesehen, im Code fehlt er. Mehrere Replicas können daher doppelte `ACTIVE`-Warnungen anlegen, die den User danach dauerhaft blockieren.
3. **`RestTemplate` ohne Timeout.** Ein hängender Tracking-Service blockiert den einzigen Scheduler-Thread, ohne dass ein Fehler geloggt wird.
4. **Keine Metriken.** `/actuator/prometheus` ist freigegeben, aber die Micrometer-Dependency fehlt, und der Endpunkt verlangt zusätzlich ein JWT.
5. Kleinere Punkte: Eine bestätigte Warnung wird sofort neu angelegt, solange der User über dem Limit liegt; ein `sub`, der keine UUID ist, oder ungültige Paging-Parameter führen zu 500 statt 400.

## 4. Ausblick Lab 2: SLIs/SLOs (~0,5 min)

- HTTP-Metriken allein reichen nicht. Wenn der Scheduler hängt, liefert die API weiter `200 OK`, nur mit veralteten Warnungen.
- Vorgeschlagene SLOs (28 Tage):
  - API-Verfügbarkeit **99,5 %**
  - Latenz **95 % ≤ 300 ms**
  - **Aktualität**: letzter erfolgreicher Lauf ≤ 15 min, 99 % der Zeit
  - **Abdeckung**: 99 % der User-Auswertungen mit vollständigen Daten
- Voraussetzung: Micrometer-Dependency, eigene Scheduler-Metriken, Timeouts, und die Auswertung bei Fehlern überspringen statt mit 0 zu rechnen

---

**Kernaussage:** Der Service läuft, aber er fällt **still** aus. Unsere Priorität ist, ihn messbar zu machen und Ausfälle sichtbar statt falsch zu machen.
