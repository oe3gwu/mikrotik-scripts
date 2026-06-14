# MikroTik Scripts

RouterOS-Skripte für MikroTik-Router. Dieses Repository enthält Automatisierungen, die sich in die DHCP- und DNS-Konfiguration einbinden lassen.

## Scripts

### `create-dns-record-from-dhcp.txt`

Erstellt automatisch statische DNS-Einträge, wenn ein DHCP-Lease gebunden wird, und entfernt sie wieder, wenn der Lease freigegeben wird.

#### Funktionsweise

1. **Lease gebunden** (`$leaseBound = 1`):
   - Hostname aus dem DHCP-Lease lesen (mit Fallback auf `host-name` bzw. `active-host-name` im Lease-Eintrag)
   - Domain aus dem **spezifischsten** passenden DHCP-Netzwerk ermitteln (längste Präfixlänge gewinnt)
   - FQDN zusammenbauen; die Domain wird nur angehängt, wenn sie noch nicht im Hostnamen enthalten ist
   - Vorhandenen Eintrag mit gleichem Namen entfernen und neuen statischen DNS-Eintrag anlegen (TTL: 600 s)

2. **Lease freigegeben**:
   - Statischen DNS-Eintrag mit passender IP-Adresse und TTL entfernen

Leere oder `unknown`-Hostnames werden ignoriert — es wird kein DNS-Eintrag erzeugt. Fehlt für die IP-Adresse eine Domain in der DHCP-Netzwerk-Konfiguration, bricht das Skript ebenfalls ab.

#### Voraussetzungen

- MikroTik RouterOS mit DHCP-Server und DNS
- Pro DHCP-Netzwerk muss das Feld **Domain** gesetzt sein (z. B. `home.arpa`)
- Clients sollten einen Hostnamen per DHCP mitteilen (Option 12)

#### Installation

1. Skript auf den Router laden:

   ```text
   /tool fetch url="https://raw.githubusercontent.com/oe3gwu/mikrotik-scripts/main/create-dns-record-from-dhcp.txt"
   ```

   Alternativ den Inhalt von `create-dns-record-from-dhcp.txt` manuell unter **System → Scripts** einfügen.

2. Skript anlegen (Name nach Belieben, z. B. `dhcp-dns`):

   ```text
   /system script add name=dhcp-dns source=[/file get create-dns-record-from-dhcp.txt contents]
   ```

3. Als DHCP-Lease-Skript registrieren:

   ```text
   /ip dhcp-server lease-script add name=dhcp-dns policy=read,write,test script=dhcp-dns
   ```

   Oder pro DHCP-Server:

   ```text
   /ip dhcp-server set [find] lease-script=dhcp-dns
   ```

#### DHCP-Netzwerk konfigurieren

Für jedes Subnetz die Domain setzen:

```text
/ip dhcp-server network set [find address=192.168.1.0/24] domain=home.arpa
```

Bei überlappenden oder verschachtelten Netzen gewinnt das Netz mit der **längsten Präfixlänge** — ein Client in `192.168.1.0/24` bekommt also die Domain dieses Netzes, nicht die eines übergeordneten `192.168.0.0/16`.

#### Beispiel

| DHCP-Lease | Netzwerk-Domain | Ergebnis (DNS static) |
|---|---|---|
| `raspberry` → `192.168.1.42` | `home.arpa` | `raspberry.home.arpa` → `192.168.1.42` |
| `nas.home.arpa` → `192.168.1.10` | `home.arpa` | `nas.home.arpa` → `192.168.1.10` (Domain nicht doppelt angehängt) |

#### Hinweise

- Der TTL-Wert `600s` im Skript kennzeichnet die vom Skript verwalteten Einträge und wird beim Aufräumen beim Lease-Ende zum Finden genutzt.
- Clients ohne Hostname (oder mit `unknown`) erhalten keinen DNS-Eintrag — das ist beabsichtigt.
- Getestet u. a. mit Clients, die `dhcpcd` nutzen (z. B. Raspberry Pi); der Hostname-Fallback berücksichtigt dabei auch `active-host-name`.

## Lizenz

Keine explizite Lizenz angegeben. Nutzung auf eigene Verantwortung.
