# Setup Insights

## 1. Podman-in-Podman: Netzwerkfehler durch nftables

### Problem

Nach dem Starten des Devcontainers schlugen sowohl `podman compose up` als auch `docker compose up` fehl — die Container konnten kein Netzwerk aufbauen. Services konnten sich untereinander nicht erreichen (z.B. kein Zugriff auf PostgreSQL), und der gesamte Compose-Stack startete nicht.

Das Problem trat bei beiden Compose-Varianten auf, weil im Devcontainer `podman-docker` installiert ist: der `docker`-Befehl ist dort nur ein Alias fuer Podman. Beide Aufrufe landen also beim selben Podman-Daemon und sind vom selben Netzwerk-Problem betroffen.

### Ursache

Das Devcontainer-Image basiert auf Fedora (`quay.io/podman/stable`), das standardmaessig **nftables** als Firewall-Backend verwendet. Der Devcontainer laeuft als privilegierter Container innerhalb der Podman-Machine (WSL2-VM). In dieser verschachtelten Umgebung (Podman-in-Podman) entsteht ein Konflikt:

- Die **aeussere** Podman-Machine (Host-VM) besitzt die nftables-Kernel-Module und verwaltet die Firewall-Regeln.
- Der **innere** Podman (im Devcontainer) versucht ebenfalls, eigene nftables-Regeln zu setzen.
- Das schlaegt fehl, weil der Kernel-nftables-Zustand vom aeusseren Host belegt ist — die verschachtelten nftables-Aufrufe kollidieren oder haben keine Berechtigung, eigene Chains anzulegen.

Ergebnis: Container innerhalb des Devcontainers bekommen keine funktionierenden Netzwerk-Bridges und koennen nicht miteinander kommunizieren.

### Loesung

In `.devcontainer/containers.conf` wurde folgender Eintrag ergaenzt:

```ini
[network]
# iptables-Workaround fuer Podman-in-Podman auf Fedora
firewall_driver = "iptables"
```

Damit verwendet der innere Podman die **iptables-Kompatibilitaetsschicht** (`iptables-nft`) statt des nativen nftables-Treibers. Das Paket `iptables-nft` ist im Dockerfile bereits installiert und bietet eine Legacy-iptables-API, die ueber nftables arbeitet, aber in verschachtelten Container-Umgebungen zuverlaessig funktioniert.

### Zusammenfassung

| | Ohne Fix | Mit Fix |
|---|---|---|
| Firewall-Treiber | `nftables` (Fedora-Standard) | `iptables` (Legacy-Kompatibilitaetsschicht) |
| Verschachtelte Container | Netzwerk-Setup schlaegt fehl | Funktioniert ueber `iptables-nft` |

---

## 2. Git: CRLF/LF Zeilenenden auf Windows

### Problem

Git zeigt unter Windows Dateien als geaendert an, obwohl kein inhaltlicher Unterschied sichtbar ist.

### Ursache

Ohne explizite Konfiguration konvertiert Git auf Windows beim Checkout LF zu CRLF (`core.autocrlf = true` ist der Standard). Shell-Skripte und Konfigurationsdateien, die im Repository mit LF gespeichert sind, erscheinen dann lokal als geaendert.

### Loesung

1. Das Repository enthaelt bereits eine `.gitattributes`, die fuer kritische Dateitypen (`*.sh`, `*.conf`, `Dockerfile`, `.env`) LF erzwingt.
2. Zusaetzlich sollte global gesetzt werden:
   ```bash
   git config --global core.autocrlf input
   ```
   Das bedeutet: beim Commit wird CRLF zu LF normalisiert, beim Checkout bleiben die Dateien unveraendert.
3. Falls trotzdem Phantom-Diffs auftreten, einmalig den Index neu normalisieren:
   ```bash
   git add --renormalize .
   ```
