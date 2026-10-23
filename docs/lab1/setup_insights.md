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

---

## 3. Podman: `aardvark-dns` bind failure nach Devcontainer-Reboot

### Problem

Nach einem Devcontainer-Neustart baut `podman compose up` zwar alle Images, scheitert
aber beim Start des ersten Containers mit:

```
aardvark-dns failed to start: Error starting server failed to bind udp listener
on 192.168.49.1:53: IO error: Cannot assign requested address (os error 99)
```

`192.168.49.1` ist das Gateway des **minikube**-Netzes, nicht des Compose-Netzes. Der
Fehler bleibt auch nach dem Entfernen der Container bestehen.

### Ursache

1. Der Reboot ändert die boot ID der (WSL2-)VM. Podmans Laufzeit-State unter `/run`
   ist stale — alle `podman`-Befehle melden dann zunächst
   *"current system boot ID differs from cached boot ID; an unhandled reboot has occurred"*.
2. Im netavark-State (`/run/containers/networks`) liegt noch ein verwaister
   `minikube`-Eintrag mit Gateway `192.168.49.1`.
3. `aardvark-dns` ist **ein einziger Prozess**, der beim Start an die Gateway-IPs
   *aller* bekannten Netze bindet. Die tote minikube-Bridge `192.168.49.1` existiert
   nach dem Reboot nicht mehr → Bind schlägt fehl → aardvark-Start scheitert komplett →
   auch der eigentlich gesunde Compose-Container kommt nicht hoch.

### Lösung

Stale Laufzeit-State entfernen (die ersten beiden schlägt Podman selbst vor, das dritte
enthält den verwaisten minikube-Eintrag):

```bash
rm -rf /run/libpod /run/containers/storage /run/containers/networks
```

Gefahrlos, solange kein Container läuft und außer dem Default-Netz `podman` keine
persistente Netz-Definition existiert.

Prävention: minikube vor einem Reboot sauber beenden (`minikube delete`), damit kein
verwaister `192.168.49.1`-Eintrag zurückbleibt. Ausführliche Analyse siehe
`docs/lab1/2026.10.22 flober .../analyse.md`.
