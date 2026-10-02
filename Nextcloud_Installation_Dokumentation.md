# Nextcloud – Installation & Einrichtung

**Praxisprojekt – Nextcloud auf Proxmox**

**Stand: 02.10.2026**

> **Hinweis:** Diese Dokumentation rekonstruiert den damaligen Installationsablauf aus der vorhandenen Projektdokumentation und dem bisherigen Einrichtungsverlauf. Die Nextcloud-Installation wurde während der ursprünglichen Einrichtung **nicht vollständig erfolgreich abgeschlossen**. Der aufgetretene Fehler wird deshalb am Ende dokumentiert und nicht als erfolgreiches Ergebnis dargestellt.

---

## 1. Ziel

Ziel war die Einrichtung einer eigenen Nextcloud-Instanz in einem separaten Proxmox-LXC-Container.

Die geplante Struktur:

```text
Internet / LAN
      │
      ▼
   Proxmox
      │
      └── CT 101
          └── Nextcloud
```

Die Nextcloud sollte als eigener Dienst in CT 101 laufen und später über das interne NAT-Netz erreichbar sein.

---

## 2. Nextcloud-Container erstellen

Für Nextcloud wurde ein eigener LXC-Container erstellt.

### Container

| Einstellung | Wert |
|---|---|
| CT-ID | `101` |
| Betriebssystem | Debian 12 |
| CPU | 1 Core |
| RAM | 2048 MB |
| Swap | 512 MB |
| Root-Disk | 30 GB |
| Netzwerk-Bridge | `vmbr1` |
| IPv4 | `10.10.10.10/24` |
| Gateway | `10.10.10.1` |

Der Container wurde als eigener LXC betrieben.

---

## 3. Netzwerk testen

Nach dem Start des Containers wurde zuerst die Verbindung zum NAT-Gateway getestet:

```bash
ping -c 4 10.10.10.1
```

Anschließend wurde die Internetverbindung geprüft:

```bash
ping -c 4 8.8.8.8
```

Zum Test der DNS-Auflösung:

```bash
ping -c 4 google.com
```

Diese Tests sind wichtig, bevor die eigentliche Nextcloud-Installation beginnt.

---

## 4. Debian aktualisieren

Im Nextcloud-Container:

```bash
apt update
```

Danach:

```bash
apt upgrade -y
```

---

## 5. Apache, MariaDB und PHP installieren

Für Nextcloud wurden Apache, MariaDB, PHP und die benötigten PHP-Erweiterungen installiert.

Der verwendete Installationsbefehl war:

```bash
apt install apache2 mariadb-server libapache2-mod-php php php-gd php-mysql php-curl php-mbstring php-intl php-imagick php-xml php-zip php-bcmath php-gmp php-apcu php-redis -y
```

Damit wurden unter anderem installiert:

- Apache2
- MariaDB
- PHP
- PHP-GD
- PHP-MySQL
- PHP-cURL
- PHP-mbstring
- PHP-Intl
- PHP-Imagick
- PHP-XML
- PHP-ZIP
- PHP-BCMath
- PHP-GMP
- PHP-APCu
- PHP-Redis

---

## 6. PHP-Version prüfen

Die installierte PHP-Version wurde überprüft:

```bash
php -v
```

Verwendet wurde Debian 12 mit PHP 8.2.

Die PHP-Version war für die Auswahl der Nextcloud-Version relevant.

---

## 7. MariaDB vorbereiten

MariaDB wurde zunächst gestartet bzw. aktiviert:

```bash
systemctl enable --now mariadb
```

Danach wurde die MariaDB-Konsole geöffnet:

```bash
mysql
```

---

## 8. Nextcloud-Datenbank erstellen

In MariaDB wurde eine eigene Datenbank für Nextcloud erstellt.

```sql
CREATE DATABASE nextcloud;
```

Danach wurde ein eigener Datenbankbenutzer angelegt:

```sql
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'DEIN_PASSWORT';
```

Dem Benutzer wurden die benötigten Rechte gegeben:

```sql
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
```

Danach:

```sql
FLUSH PRIVILEGES;
```

MariaDB verlassen:

```sql
EXIT;
```

> **Hinweis:** Das tatsächliche Passwort wird hier absichtlich nicht dokumentiert.

---

## 9. Nextcloud-Version auswählen

Zunächst wurde Nextcloud 35.0.0 verwendet.

Beim Aufruf im Browser wurde jedoch festgestellt, dass diese Version mindestens PHP 8.3 benötigt.

Da Debian 12 in der verwendeten Umgebung PHP 8.2 bereitstellte, wurde anschließend Nextcloud **34.0.4** verwendet.

Damit sollte die Nextcloud-Version zur vorhandenen PHP-Version passen.

---

## 10. Nextcloud herunterladen

Die Nextcloud-Dateien wurden anschließend unter `/var/www/nextcloud` bereitgestellt.

Zielverzeichnis:

```text
/var/www/nextcloud
```

Die Dateien wurden dort entpackt bzw. installiert.

---

## 11. Rechte für Nextcloud setzen

Der Webserver verwendet den Benutzer `www-data`.

Daher wurde der Nextcloud-Ordner diesem Benutzer zugeordnet:

```bash
chown -R www-data:www-data /var/www/nextcloud
```

---

## 12. Apache VirtualHost erstellen

Für Nextcloud wurde eine eigene Apache-Konfiguration erstellt:

```bash
nano /etc/apache2/sites-available/nextcloud.conf
```

Verwendete Konfiguration:

```apache
<VirtualHost *:80>
    ServerName nextcloud
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>

    TransferLog /var/log/apache2/nextcloud_access.log
    ErrorLog /var/log/apache2/nextcloud_error.log
</VirtualHost>
```

Datei speichern:

- `STRG + O`
- `Enter`
- `STRG + X`

---

## 13. Apache-Module aktivieren

Für Nextcloud wurden zusätzliche Apache-Module aktiviert:

```bash
a2enmod rewrite headers env dir mime setenvif
```

Anschließend wurde die Nextcloud-Site aktiviert:

```bash
a2ensite nextcloud.conf
```

Die Standardseite von Apache wurde deaktiviert:

```bash
a2dissite 000-default.conf
```

Danach Apache neu laden:

```bash
systemctl reload apache2
```

---

## 14. Apache prüfen

Der Apache-Dienst kann kontrolliert werden:

```bash
systemctl status apache2
```

Zusätzlich kann die Konfiguration getestet werden:

```bash
apache2ctl configtest
```

Bei einer korrekten Konfiguration sollte erscheinen:

```text
Syntax OK
```

---

## 15. Nextcloud im Browser öffnen

Die Nextcloud wurde anschließend über die IP-Adresse des Containers aufgerufen.

Die verwendete Adresse des CT 101 war:

```text
http://10.10.10.10
```

Damit sollte die Nextcloud-Installationsseite erscheinen.

---

## 16. Nextcloud-Installation im Browser

Im Webinstaller wurden die grundlegenden Daten eingetragen.

### Datenbank

```text
Datenbankbenutzer:
nextcloud

Datenbankname:
nextcloud

Datenbankhost:
localhost
```

Als Datenbankpasswort wurde das bei der MariaDB-Einrichtung festgelegte Passwort verwendet.

Das Nextcloud-Datenverzeichnis sollte unter:

```text
/var/www/nextcloud/data
```

liegen.

---

## 17. Installationsfehler

Während der Installation trat ein interner Fehler auf.

In den Nextcloud-Logs wurde folgende Fehlermeldung gefunden:

```text
Migration step 'OCA\Files_Sharing\Migration\Version33000Date20260306120000' is unknown
```

Die betroffene Migration gehört zur App:

```text
files_sharing
```

Die entsprechende Datei war im Nextcloud-Verzeichnis vorhanden:

```text
apps/files_sharing/lib/Migration/Version33000Date20260306120000.php
```

Auch die Klasse war vorhanden:

```text
class Version33000Date20260306120000 extends SimpleMigrationStep
```

mit dem Namespace:

```text
OCA\Files_Sharing\Migration
```

---

## 18. Weitere Fehlersuche

Es wurde überprüft, ob die Migration in den Composer-Autoload-Dateien eingetragen war.

Dabei wurde kein passender Eintrag in:

```text
autoload_static.php
```

bzw.

```text
autoload_psr4.php
```

gefunden.

Die App-Version von `files_sharing` wurde ebenfalls geprüft:

```text
1.26.0
```

---

## 19. Datenbank zurücksetzen

Um eine möglicherweise fehlerhafte bzw. unvollständige Installation auszuschließen, wurde die Nextcloud-Datenbank zurückgesetzt.

In MariaDB:

```sql
DROP DATABASE nextcloud;
```

Danach wurde sie erneut erstellt:

```sql
CREATE DATABASE nextcloud;
```

Die Berechtigungen für den Benutzer wurden erneut gesetzt:

```sql
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
```

Danach:

```sql
FLUSH PRIVILEGES;
```

---

## 20. Ergebnis der ursprünglichen Installation

Auch nach dem Zurücksetzen der Datenbank trat der Migrationsfehler erneut auf.

Die Installation wurde deshalb an dieser Stelle nicht als erfolgreich abgeschlossen dokumentiert.

Der Fehler lautete weiterhin:

```text
Migration step 'OCA\Files_Sharing\Migration\Version33000Date20260306120000' is unknown
```

Anschließend wurde entschieden, den bisherigen CT 101 zu entfernen und einen neuen CT 101 für einen sauberen neuen Installationsversuch zu erstellen.

> **Wichtig:** Der bestehende VPN-Container **CT 100** wurde dabei nicht verändert.

---

## 21. Verwendete Verzeichnisstruktur

Die geplante Nextcloud-Struktur war:

```text
/var/www/nextcloud
├── apps/
├── config/
├── data/
├── lib/
├── occ
└── ...
```

Das Datenverzeichnis war:

```text
/var/www/nextcloud/data
```

Dieses Verzeichnis wurde später auch für das separate UrBackup-Konzept verwendet.

---

## 22. Zusammenfassung der verwendeten Komponenten

| Komponente | Verwendung |
|---|---|
| Proxmox | Virtualisierung |
| LXC 101 | Nextcloud-Container |
| Debian 12 | Betriebssystem |
| Apache2 | Webserver |
| MariaDB | Datenbank |
| PHP 8.2 | PHP-Laufzeitumgebung |
| Nextcloud 34.0.4 | Cloud-Anwendung |
| `vmbr1` | Internes NAT-Netz |
| `10.10.10.10` | IP des Nextcloud-CT |

---

## 23. Geplante Netzwerkstruktur

```text
Router
192.168.178.1
       │
       ▼
Proxmox
192.168.178.132
       │
      NAT
       │
       ▼
vmbr1
10.10.10.1
       │
       ├── CT 101
       │   Nextcloud
       │   10.10.10.10
       │
       ├── CT 102
       │   Uptime Kuma
       │   10.10.10.11
       │
       └── CT 103
           UrBackup
           10.10.10.12
```

---

## 24. Hinweis für die weitere Dokumentation

Die spätere UrBackup-Dokumentation baut auf dem Nextcloud-Datenverzeichnis

```text
/var/www/nextcloud/data
```

auf.

Die vollständigen Container-Backups werden weiterhin über Proxmox durchgeführt.

UrBackup übernimmt zusätzlich die dateibasierte Sicherung wichtiger Nextcloud-Dateien.

---

## Ergebnis

Die grundlegende Nextcloud-Umgebung wurde vorbereitet:

- eigener Proxmox-LXC-Container
- Debian 12
- Apache2
- MariaDB
- PHP 8.2
- benötigte PHP-Erweiterungen
- eigene MariaDB-Datenbank
- Apache-VirtualHost
- internes NAT-Netz
- Nextcloud 34.0.4

Die ursprüngliche Installation konnte wegen des dokumentierten `files_sharing`-Migrationsfehlers jedoch nicht erfolgreich abgeschlossen werden.
