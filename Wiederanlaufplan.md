# Wiederanlaufplan – Komplettausfall der Proxmox-Umgebung

## 1. Ziel

Dieser Wiederanlaufplan beschreibt das Vorgehen bei einem vollständigen Ausfall des Proxmox-Servers.

Als Beispiel wird angenommen, dass der Mini-PC bzw. die Festplatte mit Proxmox vollständig ausfällt und die darauf laufenden Dienste nicht mehr erreichbar sind.

Ziel ist es, die virtuelle Infrastruktur mithilfe der vorhandenen Backups wiederherzustellen.

---

## 2. Ausgangssituation

Die Serverumgebung besteht aus einem Proxmox-Server und mehreren LXC-Containern.

| Container | Aufgabe | IP-Adresse |
|---|---|---|
| CT 100 | VPN / Tailscale | 10.10.10.x bzw. vorhandene VPN-IP |
| CT 101 | Nextcloud | 10.10.10.10 |
| CT 102 | Uptime Kuma | 10.10.10.11 |
| CT 103 | UrBackup | 10.10.10.12 |

Der Proxmox-Host besitzt im normalen Netzwerk die IP-Adresse:

`192.168.178.132`

Für das interne Container-Netz wird `vmbr1` mit dem Netzwerk `10.10.10.0/24` verwendet.

---

## 3. Vorhandene Sicherungen

Für den Notfall stehen mehrere Sicherungsmöglichkeiten zur Verfügung:

### Proxmox-Backup

Jeden Abend um 21:00 Uhr wird ein vollständiges Backup der Container erstellt.

- Backup-Ziel: interne Proxmox-Sicherung
- Backup enthält die vollständigen Container
- Nextcloud-Daten sind enthalten
- Aufbewahrung: 3 Backups

Zusätzlich wird um 22:00 Uhr ein Backup auf dem USB-Speicher erstellt.

- Backup-Ziel: USB
- Aufbewahrung: 1 Backup
- Die Nextcloud-Daten unter `/var/www/nextcloud/data` sind bei diesem Backup ausgeschlossen.

### UrBackup

Die wichtigen Nextcloud-Dateien werden zusätzlich über UrBackup gesichert.

Der Datenfluss ist:

```text
Nextcloud
    │
    ▼
UrBackup Client
    │
    ▼
UrBackup Server
    │
    ▼
USB-Speicher
```

Dadurch existiert neben dem vollständigen Proxmox-Backup eine zusätzliche dateibasierte Sicherung der Nextcloud-Daten.

---

# 4. Vorgehen bei einem Komplettausfall

## Schritt 1 – Ausfall feststellen

Zuerst wird geprüft, ob der Proxmox-Server tatsächlich nicht mehr erreichbar ist.

Beispiele:

- Proxmox-Weboberfläche ist nicht erreichbar.
- Der Mini-PC startet nicht.
- Die Container sind nicht erreichbar.
- Der Server reagiert nicht auf Netzwerkzugriffe.

Wenn eindeutig ein Hardware- oder Systemausfall vorliegt, beginnt die Wiederherstellung.

---

## Schritt 2 – Hardware reparieren oder Ersatzhardware bereitstellen

Wenn möglich, wird der Mini-PC repariert.

Falls die Hardware nicht repariert werden kann, wird geeignete Ersatzhardware verwendet.

Anschließend wird Proxmox VE neu installiert.

Die vorhandenen Backups werden dabei nicht gelöscht.

---

## Schritt 3 – Proxmox-Netzwerk wiederherstellen

Nach der Installation wird das Netzwerk wieder eingerichtet.

Das normale Netzwerk verwendet:

```text
vmbr0
IP: 192.168.178.132/24
Gateway: 192.168.178.1
```

Das interne Container-Netz verwendet:

```text
vmbr1
Netz: 10.10.10.0/24
Proxmox: 10.10.10.1
```

Für die Internetverbindung der Container müssen außerdem IPv4-Forwarding und NAT wieder eingerichtet werden.

Beispiel:

```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-nat.conf
sysctl --system

sysctl net.ipv4.ip_forward

iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE
iptables -A FORWARD -i vmbr1 -o vmbr0 -j ACCEPT
iptables -A FORWARD -i vmbr0 -o vmbr1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Anschließend wird geprüft, ob der Proxmox-Host und das Netzwerk wieder funktionieren.

---

## Schritt 4 – USB-Backup anschließen

Der USB-Speicher mit den Backups wird an den wiederhergestellten Proxmox-Server angeschlossen.

Die Proxmox-Backups befinden sich auf der dafür vorgesehenen USB-Partition.

Der USB-Speicher wird wieder eingebunden und in Proxmox als Backup-Speicher eingerichtet.

Danach wird geprüft, ob die vorhandenen Backup-Dateien erkannt werden.

---

## Schritt 5 – Container wiederherstellen

Die benötigten Container werden aus dem neuesten verfügbaren vollständigen Proxmox-Backup wiederhergestellt.

Empfohlene Reihenfolge:

1. **CT 100 – VPN / Tailscale**
2. **CT 101 – Nextcloud**
3. **CT 102 – Uptime Kuma**
4. **CT 103 – UrBackup**

Nach jeder Wiederherstellung wird geprüft, ob der jeweilige Container gestartet werden kann.

---

## Schritt 6 – Dienste überprüfen

Nach der Wiederherstellung werden die einzelnen Dienste kontrolliert.

### Proxmox

```bash
pct list
```

Damit wird geprüft, ob die Container vorhanden sind.

Zusätzlich kann der Status eines Containers geprüft werden:

```bash
pct status 100
pct status 101
pct status 102
pct status 103
```

### Netzwerk

Von den Containern wird geprüft, ob das Gateway und das Internet erreichbar sind.

Beispiel:

```bash
ping -c 4 10.10.10.1
ping -c 4 8.8.8.8
ping -c 4 google.com
```

### VPN

Im VPN-Container:

```bash
tailscale status
```

Damit wird kontrolliert, ob Tailscale wieder läuft.

### Nextcloud

Es wird geprüft, ob die Nextcloud-Weboberfläche erreichbar ist und ob die gespeicherten Dateien vorhanden sind.

### Uptime Kuma

Die Weboberfläche von Uptime Kuma wird auf Erreichbarkeit und funktionierende Überwachung geprüft.

### UrBackup

Die UrBackup-Weboberfläche ist über den Proxmox-Host erreichbar:

```text
http://192.168.178.132:55414
```

Im Nextcloud-Container kann zusätzlich geprüft werden:

```bash
urbackupclientctl status
```

---

# 5. Wiederherstellung der Nextcloud-Daten

Sollte bei der Wiederherstellung ein Proxmox-Backup verwendet werden, das die Nextcloud-Daten nicht enthält, wird die zusätzliche UrBackup-Sicherung verwendet.

Die wichtigen Dateien befinden sich in der UrBackup-Sicherung.

Die Daten werden anschließend wieder in die wiederhergestellte Nextcloud-Umgebung übernommen.

Die genaue Wiederherstellung einzelner Dateien erfolgt über die UrBackup-Weboberfläche.

---

# 6. Abschlusskontrolle

Nach der Wiederherstellung werden folgende Punkte kontrolliert:

- [ ] Proxmox ist erreichbar.
- [ ] `vmbr0` funktioniert.
- [ ] `vmbr1` funktioniert.
- [ ] Internetzugriff der Container funktioniert.
- [ ] VPN / Tailscale funktioniert.
- [ ] Nextcloud ist erreichbar.
- [ ] Nextcloud-Dateien sind vorhanden.
- [ ] Uptime Kuma funktioniert.
- [ ] UrBackup ist erreichbar.
- [ ] Backup-Speicher wird erkannt.
- [ ] Alle benötigten Container starten automatisch bzw. können gestartet werden.

Wenn alle wichtigen Punkte funktionieren, ist die Umgebung wieder betriebsbereit.

---

## 7. Wichtig

Der Wiederanlaufplan beschreibt das Vorgehen für einen vollständigen Ausfall.

Die Wiederherstellung sollte regelmäßig getestet werden. Ein vorhandenes Backup allein bedeutet nicht automatisch, dass eine Wiederherstellung erfolgreich funktioniert.

Ein vollständiger Wiederherstellungstest der gesamten Umgebung ist in diesem Projekt noch nicht dokumentiert.
