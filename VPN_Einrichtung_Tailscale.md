# VPN-Einrichtung

## Tailscale auf Proxmox / LXC

**Schritt-für-Schritt-Anleitung**

---

## Ziel der Anleitung

In dieser Anleitung wird ausschließlich die Einrichtung des VPNs beschrieben. Als VPN wird Tailscale in einem eigenen Proxmox-LXC-Container verwendet. Der VPN-Container erhält die CT-ID 100 und den Hostnamen „vpn“.

## Voraussetzungen

- Proxmox VE ist bereits installiert und erreichbar.
- Ein Debian-12-Container kann in Proxmox erstellt werden.
- Der Proxmox-Host ist mit dem lokalen Netzwerk verbunden.
- Für Tailscale wird ein Tailscale-Konto benötigt.

## Proxmox-Version prüfen

Auf dem Proxmox-Host kann die installierte Version mit folgendem Befehl geprüft werden:

```bash
pveversion
```

*Proxmox-Version*

**image**

*Netzwerk-Konfiguration*

---

## 1. LXC-Container für das VPN erstellen

In Proxmox links den Proxmox-Server auswählen und anschließend auf „Create CT“ klicken.

### General

- CT ID: 100
- Hostname: vpn
- Ein sicheres Root-Passwort festlegen

### Template

Als Betriebssystem-Template Debian 12 auswählen.

### Disk

- Storage: local-lvm
- Größe: z. B. 8 GB

### CPU und Arbeitsspeicher

- CPU: 1 Core
- RAM: 512 MB oder 1 GB
- Swap: 512 MB

### Network

- Bridge: vmbr0
- IPv4: DHCP
- IPv6: nicht benötigt

Danach auf „Finish“ klicken.

## 2. VPN-Container starten

Links den Container „100 (vpn)“ auswählen und auf „Start“ klicken. Danach „Console“ öffnen.

Wenn die Debian-Konsole erscheint, kannst du mit den nächsten Schritten fortfahren.

## 3. Netzwerkverbindung testen

Zuerst prüfen, ob der Container den Router erreicht:

```bash
ping -c 4 192.168.178.1
```

Danach die Internetverbindung testen:

```bash
ping -c 4 8.8.8.8
```

Wenn Antworten zurückkommen, hat der Container Internetzugriff.

## 4. Debian aktualisieren

```bash
apt update
```

```bash
apt upgrade -y
```

## 5. curl installieren

```bash
apt install curl -y
```

## 6. Tailscale installieren

Tailscale wird mit dem offiziellen Installationsskript installiert:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Warten, bis die Installation vollständig abgeschlossen ist.

## 7. Tailscale-Dienst starten

```bash
systemctl start tailscaled
```

Anschließend den Status prüfen:

```bash
systemctl status tailscaled
```

Gesucht wird die Meldung „Active: active (running)“. Mit „q“ kannst du die Statusanzeige verlassen.

## 8. Tailscale mit dem Konto verbinden

```bash
tailscale up
```

Tailscale zeigt anschließend einen Link an. Diesen Link auf einem normalen PC oder Smartphone im Browser öffnen und mit dem Tailscale-Konto anmelden.

## 9. VPN-IP anzeigen

```bash
tailscale ip
```

Tailscale zeigt eine VPN-IP an. Sie liegt normalerweise im 100.x.x.x-Bereich. Diese Adresse ist für die Verbindung über das Tailscale-Netz wichtig.

## 10. Verbundene Geräte prüfen

```bash
tailscale status
```

Hier werden die Geräte angezeigt, die im Tailscale-Netz verbunden sind.

## 11. Automatischen Start aktivieren

Damit Tailscale nach einem Neustart automatisch gestartet wird:

```bash
systemctl enable tailscaled
```

## 12. Proxmox-Container prüfen

Auf dem Proxmox-Host kann geprüft werden, ob der Container läuft:

```bash
pct status 100
```

Zusätzlich kann eine Übersicht aller Container angezeigt werden:

```bash
pct list
```

## 13. Neustart testen

Um zu prüfen, ob der VPN-Container nach einem Neustart wieder funktioniert:

```bash
pct reboot 100
```

Danach einige Sekunden warten und prüfen:

```bash
pct status 100
```

Wenn „status: running“ angezeigt wird, erneut in die Container-Konsole gehen und Tailscale prüfen:

```bash
tailscale status
```

---

## Fertig – VPN-Struktur

```text
Internet

   │

   ▼

Router

   │

   ▼

Proxmox

   │

   └── LXC 100 „vpn“

          └── Tailscale VPN

                ├── PC

                ├── Handy

                └── weitere Geräte
```

## Wichtige Hinweise

- Der VPN-Container hat die CT-ID 100.
- Der VPN-Container sollte nicht gelöscht werden, wenn später weitere Dienste wie Nextcloud eingerichtet werden.
- Weitere Dienste können in eigenen LXC-Containern betrieben werden.
- Das Root-Passwort und das Tailscale-Konto sollten sicher aufbewahrt werden.
