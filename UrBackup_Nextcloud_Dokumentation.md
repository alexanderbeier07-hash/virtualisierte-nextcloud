# Praxisprojekt – Backup & Wiederherstellung

## UrBackup für Nextcloud

**Installation · Einrichtung · Test · Aufbewahrung**

**Stand: 02.10.2026**

---

## 1. Ziel

Zusätzlich zu den vollständigen Proxmox-Backups werden die wichtigen Nextcloud-Benutzerdateien und Bilder dateibasiert mit UrBackup auf einer separaten USB-Partition gesichert.

Die vollständigen Proxmox-Backups bleiben die zentrale Option für eine komplette Container-Wiederherstellung.

---

## 2. Aufbau

| Komponente | Adresse / Pfad | Aufgabe |
|---|---|---|
| Proxmox | `192.168.178.132` | Virtualisierung |
| Nextcloud CT 101 | `10.10.10.10` | Dateien und Bilder |
| Uptime Kuma CT 102 | `10.10.10.11` | Monitoring |
| UrBackup CT 103 | `10.10.10.12` | Datei-Backup-Server |
| UrBackup-Speicher | `/backup` | USB-Partition |
| Backup-Quelle | `/var/www/nextcloud/data` | Nextcloud-Daten |

### Datenfluss

```text
Nextcloud CT 101
       │
       ▼
UrBackup Client
       │
       ▼
UrBackup CT 103
       │
       ▼
USB-Speicher
```

---

## 3. USB-Speicher

| Partition | Größe | Verwendung |
|---|---:|---|
| `/dev/sdb1` | 5 GiB | Proxmox-Backups |
| `/dev/sdb2` | ca. 24,3 GiB | UrBackup |

Die UrBackup-Partition wird auf dem Proxmox-Host unter

```text
/mnt/urbackup-data
```

gemountet und in CT 103 als

```text
/backup
```

eingebunden.

LXC-Mount:

```text
mp0: /mnt/urbackup-data,mp=/backup
```

> **Wichtig:** `/dev/sda` enthält den Proxmox-Host und wurde nicht verändert.

---

## 4. UrBackup-Container

| Eigenschaft | Wert |
|---|---|
| CT-ID | `103` |
| Hostname | `urbackup` |
| OS | Debian 12 |
| IP | `10.10.10.12/24` |
| Gateway | `10.10.10.1` |
| CPU | 1 vCPU |
| RAM | 1024 MB |
| Rootfs | 8 GB |
| Backup-Mount | `/backup` |

Der LXC ist unprivilegiert. Für den gemounteten Backup-Ordner mussten deshalb die UID/GID-Zuordnungen auf dem Proxmox-Host berücksichtigt werden.

---

## 5. Installation des UrBackup-Servers

Im UrBackup-Container wurde das Server-Paket heruntergeladen:

```bash
wget https://hndl.urbackup.org/Server/latest/debian/bookworm/urbackup-server_2.5.38_amd64.deb
```

Installation:

```bash
dpkg -i urbackup-server_2.5.38_amd64.deb
```

Fehlende Abhängigkeiten installieren:

```bash
apt -f install -y
```

Bei der Paketkonfiguration wurde

```text
/backup
```

als Backup-Speicher eingetragen.

Danach wurde die Paketkonfiguration mit folgendem Befehl erfolgreich abgeschlossen:

```bash
dpkg --configure -a
```

### Dienst prüfen

```bash
systemctl status urbackupsrv --no-pager
```

### Port prüfen

```bash
ss -lntp | grep 55414
```

Ergebnis:

Der Dienst `urbackupsrv` läuft und lauscht auf Port `55414`.

*Abb. 1 – UrBackup-Dienst läuft.*

---

## 6. Weboberfläche und Sicherheit

Der Zugriff erfolgt über eine Weiterleitung vom Proxmox-Host auf den internen UrBackup-Container.

### Portweiterleitung einrichten

```bash
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 55414 -j DNAT --to-destination 10.10.10.12:55414
```

Weiterleitung vom externen Netzwerk zum UrBackup-Container erlauben:

```bash
iptables -A FORWARD -i vmbr0 -o vmbr1 -p tcp -d 10.10.10.12 --dport 55414 -j ACCEPT
```

Antwortverkehr erlauben:

```bash
iptables -A FORWARD -i vmbr1 -o vmbr0 -p tcp -s 10.10.10.12 --sport 55414 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Weboberfläche

```text
http://192.168.178.132:55414
```

Anschließend wurde ein Admin-Benutzer mit Passwort eingerichtet.

*Abb. 2 – UrBackup-Weboberfläche.*

---

## 7. UrBackup-Client in Nextcloud

Im Nextcloud-Container wurde der UrBackup-Client heruntergeladen:

```bash
wget https://hndl.urbackup.org/Client/2.5.31/UrBackup%20Client%20Linux%202.5.31.sh
```

Datei ausführbar machen:

```bash
chmod +x "UrBackup Client Linux 2.5.31.sh"
```

Installation starten:

```bash
./"UrBackup Client Linux 2.5.31.sh"
```

Bei der Snapshot-Abfrage wurde **Option 5 – kein Snapshot-Mechanismus** gewählt.

Für dieses Projekt werden nur Dateisicherungen benötigt; Image-Backups sind nicht vorgesehen.

---

## 8. Backup-Pfad

Als Backup-Quelle wurde ausschließlich der Nextcloud-Datenordner eingetragen:

```text
/var/www/nextcloud/data
```

Backup-Verzeichnis hinzufügen:

```bash
urbackupclientctl add-backupdir --path /var/www/nextcloud/data
```

Konfiguration kontrollieren:

```bash
urbackupclientctl list-backupdirs
```

Die Ausgabe bestätigte:

```text
/var/www/nextcloud/data
```

als konfigurierten Backup-Pfad.

---

## 9. Verbindung prüfen

Den Status des UrBackup-Clients prüfen:

```bash
urbackupclientctl status
```

Der Client erkannte den UrBackup-Server unter:

```text
10.10.10.12
```

*Abb. 4 – Server 10.10.10.12 wird erkannt.*

---

## 10. Erster Backup-Test

Ein manuelles Backup starten:

```bash
urbackupclientctl start -f
```

Der manuelle Test meldete:

```text
Completed successfully
```

Im UrBackup-Webinterface wurde die Dateisicherung anschließend mit dem Status **OK** angezeigt.

*Abb. 5 – Manueller Backup-Test.*

*Abb. 6 – Dateisicherung OK.*

---

## 11. Kontrolle des USB-Backups

Auf dem UrBackup-Container wurde der Inhalt des Backup-Speichers kontrolliert:

```bash
du -sh /backup/*
```

### Ergebnisse

| Eintrag | Testgröße |
|---|---:|
| `/backup/Nextcloud` | 105 MB |
| `/backup/clients` | 4,0 KB |
| `/backup/urbackup_tmp_files` | 4,0 KB |

Damit wurde bestätigt, dass die Daten tatsächlich im UrBackup-Speicher angekommen sind.

*Abb. 7 – UrBackup-Daten unter `/backup`.*

---

## 12. Automatische Sicherung und Aufbewahrung

| Einstellung | Wert |
|---|---|
| Inkrementelle Dateisicherung | alle 3 Stunden |
| Vollständige Dateisicherung | alle 1 Tag |
| Max. inkrementelle Sicherungen | 3 |
| Min. inkrementelle Sicherungen | 1 |
| Max. vollständige Sicherungen | 1 |
| Min. vollständige Sicherungen | 1 |

UrBackup überschreibt ältere Sicherungen nicht direkt. Es verwaltet Backup-Versionen und räumt ältere Versionen entsprechend den Aufbewahrungsgrenzen automatisch auf.

Dadurch wird verhindert, dass der kleine USB-Speicher unbegrenzt wächst.

*Abb. 8 – Aufbewahrungseinstellungen.*

---

## 13. Gesamte Infrastruktur

### Nextcloud

Zentrale Dateiablage für Benutzerdateien und Bilder.

### UrBackup

Dateibasierte Zusatzsicherung der Nextcloud-Daten.

### Proxmox Backup

Vollständige Container-Sicherung.

### USB-Backup

Zusätzliche portable Sicherung.

### Uptime Kuma

Überwachung der Dienste und Backup-Komponenten.

### VPN / Tailscale

Fernzugriff.

### vmbr1 / NAT

Separates internes Netzwerk.

---

## 14. Wiederherstellung

UrBackup-Dateien werden über die UrBackup-Weboberfläche durchsucht und wiederhergestellt.

Die Daten auf dem USB-Stick liegen in der von UrBackup verwalteten Backup-Struktur und sind nicht als gewöhnliche Windows-Ordnerstruktur gedacht.

Für eine vollständige Wiederherstellung der Nextcloud-Umgebung bleibt das Proxmox-Backup die zentrale Recovery-Option.

UrBackup dient als zusätzliche dateibasierte Sicherung wichtiger Benutzerdateien.

---

## 15. Ergebnis

### Erfolgreich umgesetzt

- UrBackup Server **2.5.38** läuft in CT 103.
- Der Nextcloud-Client ist verbunden.
- `/var/www/nextcloud/data` wird gesichert.
- Die Sicherung wurde erfolgreich auf dem USB-Speicher nachgewiesen.
- Die Aufbewahrung wurde eingerichtet.
- Ein manueller Backup-Test wurde erfolgreich durchgeführt.

Damit ist neben den Proxmox-Backups eine zusätzliche, dateibasierte Backup-Ebene für Nextcloud vorhanden.
