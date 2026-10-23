# Analyse: `podman compose up` scheitert an `aardvark-dns` UDP-Listener-Binding

**Datum:** 2026-10-23
**Symptom-Log:** `podman compose up log.txt` (dieses Verzeichnis)

## 1. Symptom

`podman compose up` baut alle Images erfolgreich, scheitert dann aber beim Start des
ersten Containers (`postgres`) mit:

```
[postgres] | [WARN  netavark::dns::aardvark] Failed to delete aardvark-dns entries after failed start: aardvark-dns failed to start: Error from child process
[postgres] |     Error starting server failed to bind udp listener on 192.168.49.1:53: IO error: Cannot assign requested address (os error 99)
[postgres] | Error: unable to start container ...: netavark: error while applying dns entries: aardvark-dns failed to start: ...
[postgres] |     Error starting server failed to bind udp listener on 192.168.49.1:53: IO error: Cannot assign requested address (os error 99)
```

Auffällig: Der Fehler nennt `192.168.49.1` — das ist **nicht** das Gateway des
Compose-Netzes (`beer-tracker-network`), sondern das Default-Gateway des
**minikube**-Netzwerks aus dem minikube-Teil des Labs.

Das Problem blieb auch bestehen, nachdem die Container entfernt wurden
(`after container removed`) — ein Hinweis darauf, dass der Auslöser nicht in den
Containern, sondern im Netzwerk-/Laufzeit-State von Podman liegt.

## 2. Diagnose

Beim Versuch, den Podman-State zu inspizieren, kam die entscheidende Meldung:

```
Error: current system boot ID differs from cached boot ID; an unhandled reboot has occurred.
Please delete directories "/run/containers/storage" and "/run/libpod" and re-run Podman
```

Daraus ergibt sich die vollständige Ursachenkette:

1. **Devcontainer-Neustart / Reboot.** Die boot ID der (WSL2-)VM ändert sich. Podmans
   Laufzeit-Verzeichnisse unter `/run` (`/run/libpod`, `/run/containers/storage`)
   enthalten noch den State von vor dem Reboot und gelten als stale. Solange dieser
   Zustand besteht, blockieren **alle** `podman`-Aufrufe mit der boot-ID-Meldung.

2. **Verwaister minikube-Netzeintrag.** Im netavark-Laufzeit-State
   (`/run/containers/networks`) lag noch ein alter Eintrag für das `minikube`-Netz:

   ```
   $ cat /run/containers/networks/aardvark-dns/minikube
   192.168.49.1
   60d9...  192.168.49.2  minikube,60d9ce1488bf,minikube
   ```

   Die zugehörige Netzwerk-Definition war bereits weg (`podman network ls` zeigte nur
   noch das Default-Netz `podman`, `/etc/containers/networks` war leer), aber der
   Laufzeit-Eintrag mit Gateway `192.168.49.1` existierte noch.

3. **`aardvark-dns` bindet an alle Gateways gleichzeitig.** `aardvark-dns` ist ein
   **einziger** Prozess, der beim Start an die Gateway-IPs *aller* ihm bekannten
   Netzwerke bindet. Beim `podman compose up` versuchte er deshalb — zusätzlich zum
   frischen Compose-Netz — auch an `192.168.49.1` zu binden. Nach dem Reboot existierte
   die minikube-Bridge aber nicht mehr, diese IP war keinem Interface mehr zugewiesen:

   ```
   failed to bind udp listener on 192.168.49.1:53: Cannot assign requested address (os error 99)
   ```

   Da der aardvark-Start als Ganzes scheitert, kommt auch der Postgres-Container nicht
   hoch — obwohl dessen eigenes Netz völlig in Ordnung ist.

**Kernaussage:** Der Fehler lag nicht am Postgres- oder Compose-Netz, sondern an einem
toten minikube-Netz-Eintrag im Laufzeit-State, der den gemeinsamen DNS-Dienst
`aardvark-dns` beim Start mitriss.

## 3. Lösung

Den stale Laufzeit-State von Podman entfernen (die ersten beiden Verzeichnisse schlägt
Podman selbst vor; das dritte enthält den verwaisten minikube-Eintrag):

```bash
rm -rf /run/libpod /run/containers/storage /run/containers/networks
```

Voraussetzungen, unter denen das gefahrlos ist (alle hier erfüllt):

- Es läuft kein Container (`podman ps -a` leer bzw. durch stale boot blockiert).
- Es existiert keine persistente Netz-Definition außer dem Default
  (`podman network ls` → nur `podman`, `/etc/containers/networks` leer).

Nach der Bereinigung:

- `podman network ls` → nur Default-Netz `podman`.
- Smoke-Test: `podman network create bindtest && podman run --rm --network bindtest
  alpine true && podman network rm bindtest` → startet sauber, `aardvark-dns` bindet
  ohne Fehler.

## 4. Wiederauftreten & Prävention

Nach **jedem** Devcontainer-/VM-Neustart ändert sich die boot ID, sodass mindestens die
stale-boot-Meldung erneut auftritt. Reset-Einzeiler für künftige Fälle:

```bash
rm -rf /run/libpod /run/containers/storage /run/containers/networks
```

Zusätzlicher Auslöser speziell für das `192.168.49.1`-Binding: das minikube-Netz. Wer
sowohl den Compose- als auch den minikube-Teil des Labs nutzt, sollte minikube sauber
herunterfahren (`minikube delete`), bevor Compose nach einem Reboot neu gestartet wird,
damit kein verwaister `192.168.49.1`-Eintrag zurückbleibt.
