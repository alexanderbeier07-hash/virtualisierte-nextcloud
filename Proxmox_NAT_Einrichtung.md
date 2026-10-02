# Proxmox NAT-Netzwerk

Schritt-für-Schritt-Einrichtung eines internen NAT-Netzwerks auf Proxmox, damit mehrere CTs/VMs über die Internetverbindung des Proxmox-Hosts ins Internet gelangen können.

---

## 1. Ausgangssituation

Der Proxmox-Server besitzt eine Verbindung mit Internetzugriff.

### Bestehendes Netzwerk

| Gerät / Netzwerk | Adresse |
|---|---|
| Proxmox | `192.168.178.132/24` |
| Gateway / Router | `192.168.178.1` |
| Externe Bridge | `vmbr0` |

Ziel ist ein zusätzliches internes Netzwerk:

```text
10.10.10.0/24
```

Der Proxmox-Server übernimmt dabei Routing und NAT.

---

## 2. Internes NAT-Netz erstellen

Auf dem Proxmox-Server die Netzwerkkonfiguration öffnen:

```bash
nano /etc/network/interfaces
```

Unterhalb der bestehenden Konfiguration von `vmbr0` folgende Bridge ergänzen:

```text
auto vmbr1

iface vmbr1 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

Datei speichern:

- `STRG + O`
- `Enter`
- `STRG + X`

---

## 3. Neue Netzwerk-Konfiguration laden

Die Konfiguration wird ohne Neustart des gesamten Proxmox-Servers angewendet:

```bash
ifreload -a
```

Anschließend kontrollieren:

```bash
ip addr show vmbr1
```

Erwartet wird:

```text
10.10.10.1/24
```

---

## 4. IPv4-Forwarding aktivieren

Damit Proxmox den Netzwerkverkehr der CTs/VMs weiterleiten darf:

```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-nat.conf
```

Danach die Einstellungen laden:

```bash
sysctl --system
```

Kontrolle:

```bash
sysctl net.ipv4.ip_forward
```

Erwartetes Ergebnis:

```text
net.ipv4.ip_forward = 1
```

---

## 5. NAT einrichten

Die internen Adressen `10.10.10.0/24` werden beim Verlassen über `vmbr0` per Masquerading umgesetzt:

```bash
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE
```

Ausgehenden Verkehr von `vmbr1` nach `vmbr0` erlauben:

```bash
iptables -A FORWARD -i vmbr1 -o vmbr0 -j ACCEPT
```

Antwortpakete aus dem Internet zurück ins interne Netz erlauben:

```bash
iptables -A FORWARD -i vmbr0 -o vmbr1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

---

## 6. CT an das NAT-Netz anschließen

Beim Erstellen oder Konfigurieren des CTs als Netzwerk-Bridge `vmbr1` auswählen.

Beispiel:

| Einstellung | Wert |
|---|---|
| Bridge | `vmbr1` |
| IPv4 | `10.10.10.10/24` |
| Gateway | `10.10.10.1` |

---

## 7. NAT-Gateway testen

Im CT zuerst das interne Gateway testen:

```bash
ping -c 4 10.10.10.1
```

Wenn Antworten kommen, funktioniert die Verbindung zwischen CT und Proxmox.

---

## 8. Internetverbindung testen

Danach die Internetverbindung über eine IP-Adresse testen:

```bash
ping -c 4 8.8.8.8
```

Wenn Antworten kommen, funktioniert das NAT.

---

## 9. DNS testen

Zum Schluss die Namensauflösung testen:

```bash
ping -c 4 google.com
```

Wenn auch hier Antworten kommen, funktionieren:

- internes Netzwerk
- NAT
- Internetzugriff
- DNS

---

## 10. Netzwerkaufbau

```text
CT
10.10.10.10
     │
     ▼
vmbr1
10.10.10.1
     │
     ▼
Proxmox NAT
     │
     ▼
vmbr0
192.168.178.132
     │
     ▼
Router
192.168.178.1
     │
     ▼
Internet
```

---

## 11. Weitere CTs hinzufügen

Weitere Container können ebenfalls an `vmbr1` angeschlossen werden.

Beispiel:

```text
CT 101 → 10.10.10.10
CT 102 → 10.10.10.11
CT 103 → 10.10.10.12
```

Alle Container verwenden:

```text
Gateway: 10.10.10.1
Bridge:  vmbr1
```

Dadurch können alle Container dieselbe Internetverbindung des Proxmox-Servers über NAT verwenden.

---

## 12. Wichtige Befehle – Übersicht

### Netzwerk-Konfiguration

```bash
nano /etc/network/interfaces
```

### Netzwerk neu laden

```bash
ifreload -a
```

### Bridge überprüfen

```bash
ip addr show vmbr1
```

### IPv4-Forwarding aktivieren

```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-nat.conf
sysctl --system
```

### Forwarding überprüfen

```bash
sysctl net.ipv4.ip_forward
```

### NAT aktivieren

```bash
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o vmbr0 -j MASQUERADE
```

### Forwarding-Regeln

```bash
iptables -A FORWARD -i vmbr1 -o vmbr0 -j ACCEPT

iptables -A FORWARD -i vmbr0 -o vmbr1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

### Verbindung testen

```bash
ping -c 4 10.10.10.1
```

```bash
ping -c 4 8.8.8.8
```

```bash
ping -c 4 google.com
```

---

## Fertig

Das interne NAT-Netz ist damit aufgebaut:

```text
10.10.10.0/24
       │
       ▼
    vmbr1
       │
       ▼
   Proxmox
       │
      NAT
       │
       ▼
    vmbr0
       │
       ▼
   Internet
```
