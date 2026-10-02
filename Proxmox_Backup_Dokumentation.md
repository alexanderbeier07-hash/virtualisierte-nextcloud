# Proxmox Backup-Dokumentation

**Vollständiges Backup + USB-Backup ohne Nextcloud-Daten**

**Stand: 02.10.2026**

> **Hinweis:** Der aktuelle Aufbau wurde erfolgreich getestet. Das interne Backup bleibt vollständig. Das USB-Backup enthält die Container ohne den Nextcloud-Datenordner; die wichtigen Dateien sollen anschließend separat mit UrBackup gesichert werden.

---

## 1. Ziel

- **21:00 Uhr:** vollständiges Backup aller Container auf dem internen Proxmox-Speicher.
- **22:00 Uhr:** kleineres Backup auf dem USB-Stick, ohne `/var/www/nextcloud/data`.
- **UrBackup:** separate Sicherung wichtiger Nextcloud-Dateien wie Bilder und Dokumente.
- **USB-Backup:** Keep Last 1, damit der kleine Speicher nicht vollläuft.

---

## 2. Aktuelle Umgebung

- **Proxmox:** `192.168.178.132`
- **CT 100:** VPN/Tailscale – `192.168.178.129`
- **CT 101:** Nextcloud – `10.10.10.10`
- **CT 102:** Uptime Kuma – `10.10.10.11`

---

## 3. USB-Stick

- **`/dev/sdb`:** ca. 29,3 GiB
- **`/dev/sdb1`:** 5 GiB, Label `PROXMOX-BACKUP`
- **`/dev/sdb2`:** 24,3 GiB, Label `URBACKUP`
- Der Proxmox-Systemdatenträger **`/dev/sda`** wurde nicht verändert.

*Bild: Partitionierung des USB-Sticks*

### Formatierung

```bash
mkfs.ext4 -L PROXMOX-BACKUP /dev/sdb1

mkfs.ext4 -L URBACKUP /dev/sdb2
```

---

## 4. 5-GB-Partition als Proxmox-Speicher

Verzeichnis für das Backup erstellen:

```bash
mkdir -p /mnt/proxmox-backup
```

Partition einbinden:

```bash
mount /dev/sdb1 /mnt/proxmox-backup
```

Die Partition als Proxmox-Storage hinzufügen:

```bash
pvesm add dir usb-backup --path /mnt/proxmox-backup --content backup
```

Status prüfen:

```bash
pvesm status
```

### Ergebnis

- **Proxmox Storage-Name:** `usb-backup`
- **Mountpoint:** `/mnt/proxmox-backup`

*Bild: `usb-backup` wird von Proxmox als aktiv angezeigt.*

### Automatisches Mounten

Die Datei `/etc/fstab` öffnen:

```bash
nano /etc/fstab
```

Folgende Zeile eintragen:

```text
UUID=dc0168a7-eb2a-4f83-b93a-38682ee2f74d /mnt/proxmox-backup ext4 defaults,nofail 0 2
```

Danach testen:

```bash
mount -a
```

Und kontrollieren:

```bash
df -h /mnt/proxmox-backup
```

Die Partition wird nach einem Neustart automatisch eingebunden.

*Bild: Erfolgreicher Mount-Test*

---

## 5. Nextcloud-Daten

Das Nextcloud-Datenverzeichnis des CT 101 wurde mit folgendem Befehl ermittelt:

```bash
pct exec 101 -- su -s /bin/bash www-data -c "php /var/www/nextcloud/occ config\:system\:get datadirectory"
```

Ergebnis:

```text
/var/www/nextcloud/data
```

Die Größe des Datenordners wurde anschließend geprüft:

```bash
pct exec 101 -- du -sh /var/www/nextcloud/data
```

Ergebnis:

```text
168M /var/www/nextcloud/data
```

> **Hinweis:** Der Datenordner ist aktuell 168 MB groß und wird beim USB-Proxmox-Backup ausgeschlossen.

---

## 6. Backup-Jobs

### 21:00 – vollständiges internes Backup

- **Storage:** `local`
- **CT 100, 101, 102**
- **Snapshot + ZSTD**
- **Keep Last 3**
- **Nextcloud-Daten enthalten**
- **Job-ID:** `backup-eef19f69-92f2`

---

### 22:00 – USB-Backup

- **Storage:** `usb-backup`
- **CT 100, 101, 102**
- **Snapshot + ZSTD**
- **Keep Last 1**
- **Nextcloud-Datenordner ausgeschlossen**
- **Job-ID:** `backup-d299e139-b651`

Ausgeschlossener Pfad:

```text
exclude-path /var/www/nextcloud/data
```

*Bild: Beide Backup-Jobs in Proxmox*

*Bild: `jobs.cfg` mit `exclude-path` für den USB-Job*

---

## 7. Erfolgreicher USB-Backup-Test

Der USB-Backup-Job wurde erfolgreich getestet.

### Ergebnisse

| Container | Backup-Größe |
|---|---:|
| CT 100 VPN | 271,12 MiB |
| CT 101 Nextcloud ohne `data` | 652,32 MiB |
| CT 102 Uptime Kuma | 392,55 MiB |
| **Gesamt** | **ca. 1,29 GiB** |

Alle drei Container wurden erfolgreich als `tar.zst` gesichert.

*Bild: Erfolgreicher Test des USB-Backups*

---

## 8. Aktueller Aufbau

### Interne Festplatte

- **21:00 Uhr:** komplettes Backup inklusive Nextcloud-Dateien
- **Keep Last 3**

### USB – 5 GiB

- **22:00 Uhr:** Container / System / Konfiguration
- Nextcloud `/var/www/nextcloud/data` ausgeschlossen
- **Keep Last 1**
- Aktueller Test: **ca. 1,29 GiB**

### USB – 24,3 GiB

- Noch nicht eingerichtet.
- Geplant für **UrBackup**.
- Ziel: wichtige Nextcloud-Dateien separat sichern.

---

## 9. Nächste Schritte

- 24,3-GB-Partition `/dev/sdb2` einbinden.
- UrBackup Server installieren.
- Nextcloud-Dateien für UrBackup bereitstellen.
- Dateibackup testen.
- Wiederherstellung einzelner Dateien testen.

> **Hinweis:** Die vollständigen internen Proxmox-Backups bleiben die zusätzliche Rückfallebene. Das separate Datei-Backup mit UrBackup ist für wichtige Dateien gedacht und ersetzt nicht automatisch ein konsistentes vollständiges Nextcloud-Backup.
