# Reflexion – Was ich im Praxisprojekt gelernt habe

## 1. Allgemeine Reflexion

Durch das Praxisprojekt habe ich einen besseren Einblick in die Planung und Administration einer kleinen Serverumgebung bekommen.

Am Anfang war für mich vor allem die Planung der einzelnen Dienste und deren Zusammenspiel schwierig. Während des Projekts habe ich gelernt, dass nicht nur die Installation eines Dienstes wichtig ist, sondern auch Netzwerk, Backup, Überwachung und Wiederherstellung berücksichtigt werden müssen.

---

## 2. Proxmox und Virtualisierung

Ich habe gelernt, wie man mit Proxmox VE eine Serverumgebung virtualisieren kann.

Dabei habe ich gelernt:

- LXC-Container in Proxmox zu erstellen.
- Ressourcen wie CPU, RAM und Speicher festzulegen.
- Netzwerke für Container einzurichten.
- Container zu starten, zu stoppen und zu überprüfen.
- Dienste voneinander zu trennen, indem sie in unterschiedlichen Containern betrieben werden.

Dadurch habe ich verstanden, warum Virtualisierung für eine solche Umgebung sinnvoll ist.

---

## 3. Netzwerke und NAT

Ein wichtiger Teil des Projekts war die Netzwerkkonfiguration.

Ich habe gelernt, dass die Container nicht einfach automatisch Internetzugriff haben, wenn sie in einem eigenen internen Netzwerk betrieben werden.

Dafür habe ich ein zusätzliches Netzwerk mit `vmbr1` eingerichtet:

```text
10.10.10.0/24
```

Der Proxmox-Host übernimmt dabei die Verbindung zwischen dem internen Netzwerk und dem normalen Netzwerk.

Dabei habe ich unter anderem gelernt, was folgende Begriffe und Funktionen bedeuten:

- Bridge
- Gateway
- IPv4 Forwarding
- NAT
- Masquerading
- Firewall-/Forwarding-Regeln

Das war für mich besonders hilfreich, weil ich dadurch besser verstanden habe, wie Container miteinander und mit dem Internet kommunizieren.

---

## 4. VPN

Mit Tailscale habe ich gelernt, wie ein VPN für den Zugriff auf die eigene Serverumgebung eingesetzt werden kann.

Dabei habe ich gelernt:

- Tailscale zu installieren.
- Den Dienst zu starten und zu aktivieren.
- Den VPN-Status zu überprüfen.
- Die VPN-Verbindung für den Zugriff auf die Umgebung zu verwenden.

---

## 5. Backup und Wiederherstellung

Ein weiterer wichtiger Punkt war das Thema Backup.

Ich habe gelernt, dass ein einzelnes Backup nicht immer ausreicht.

Deshalb gibt es in meinem Projekt mehrere Sicherungsebenen:

- vollständige Proxmox-Backups
- zusätzliche USB-Sicherung
- separate dateibasierte Sicherung wichtiger Nextcloud-Daten mit UrBackup

Besonders wichtig war für mich die Erkenntnis, dass Backup und Wiederherstellung zwei unterschiedliche Themen sind.

Ein Backup ist nur dann wirklich hilfreich, wenn die Daten im Notfall auch wiederhergestellt werden können.

---

## 6. Monitoring

Mit Uptime Kuma habe ich gelernt, wie sich Dienste überwachen lassen.

Damit kann überprüft werden, ob wichtige Dienste erreichbar sind.

Dadurch habe ich verstanden, dass eine Serverumgebung nicht nur eingerichtet, sondern auch dauerhaft überwacht werden sollte.

---

## 7. Fehlersuche

Während des Projekts sind verschiedene Probleme aufgetreten.

Dabei habe ich gelernt, Fehler Schritt für Schritt einzugrenzen, anstatt direkt alles neu zu installieren.

Beispiele waren:

- Probleme beim Netzwerkzugriff der Container.
- Probleme mit Internetzugriff über das interne Netzwerk.
- Probleme bei der Installation von Nextcloud.
- Überprüfung von Diensten und deren Status.
- Kontrolle von IP-Adressen und Gateways.
- Überprüfung von Logs und Fehlermeldungen.

Dadurch habe ich gelernt, Fehlermeldungen genauer zu lesen und die Ursache eines Problems systematisch zu suchen.

---

## 8. Was ich beim nächsten Mal anders machen würde

Bei einem weiteren Projekt würde ich von Anfang an eine genauere Dokumentation der einzelnen Installationsschritte führen.

Außerdem würde ich wichtige Änderungen am Netzwerk und an den Backups direkt dokumentieren.

Einen vollständigen Wiederherstellungstest würde ich ebenfalls früher einplanen. Dadurch könnte überprüft werden, ob die erstellten Backups im Ernstfall tatsächlich ausreichen.

---

## 9. Fazit

Das Projekt hat mir gezeigt, dass eine Serverumgebung aus mehreren miteinander verbundenen Bereichen besteht.

Ich habe dabei praktische Erfahrungen mit:

- Proxmox
- LXC
- Netzwerken
- NAT
- VPN
- Nextcloud
- Backup
- UrBackup
- Monitoring
- Fehleranalyse

gesammelt.

Besonders gelernt habe ich, dass Planung, Dokumentation, Backup und Wiederherstellung genauso wichtig sind wie die eigentliche Installation eines Dienstes.
