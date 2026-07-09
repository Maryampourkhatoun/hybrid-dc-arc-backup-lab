# Troubleshooting-Log

Dieses Dokument beschreibt drei reale Probleme, die während des Labs aufgetreten
sind, sowie die Diagnose- und Lösungsschritte. Ziel ist, den tatsächlichen
Fehlerbehebungsprozess nachvollziehbar zu machen – nicht nur das Endergebnis.

---

## Problem 1: Server erreicht eigenes Gateway nicht

**Symptom:**
```
ping 192.168.100.2
Ping-Statistik für 192.168.100.2:
    Pakete: Gesendet = 4, Empfangen = 0, Verloren = 4 (100% Verlust)
```

**Diagnose:**
- `ipconfig /all` zeigte eine korrekte statische IP-Konfiguration
  (`192.168.100.10`, Gateway `192.168.100.2`)
- Die Windows-Konfiguration war also korrekt – das Problem musste außerhalb
  von Windows liegen
- Prüfung der VMware-Netzwerkadapter-Einstellungen: Adapter war korrekt auf
  **NAT** gesetzt
- Prüfung im **Virtual Network Editor**: Das NAT-Netzwerk (VMnet8) verwendete
  tatsächlich das Subnetz `172.16.242.0/24` – nicht `192.168.100.0/24`, wie in
  der VM konfiguriert

**Lösung:**
Im Virtual Network Editor das Subnetz von VMnet8 auf `192.168.100.0` /
`255.255.255.0` geändert. Nach VM-Neustart funktionierte die Verbindung zum
Gateway und zum Internet.

**Erkenntnis:** Eine korrekte IP-Konfiguration *innerhalb* der VM nützt nichts,
wenn das virtuelle Netzwerk *außerhalb* der VM (auf Hypervisor-Ebene) nicht
zum gleichen Adressbereich passt.

---

## Problem 2: Voraussetzungsprüfung für Domänencontroller schlägt fehl

**Symptom:**
```
Fehler bei der Überprüfung der Voraussetzungen für die Höherstufung des
Domänencontrollers. Das lokale Administratorkonto wird zum Domänen-
administratorkonto, wenn Sie eine neue Domäne erstellen. Die neue Domäne
kann nicht erstellt werden, da das Kennwort für das lokale Administrator-
konto nicht den Anforderungen entspricht.
```

**Diagnose:**
Das lokale Administrator-Kennwort war leer, da es bei der Windows-Installation
nie explizit gesetzt wurde.

**Lösung:**
```powershell
net user Administrator <NeuesKennwort>
```

Anschließend die Voraussetzungsprüfung erneut ausgeführt – diesmal erfolgreich.

**Erkenntnis:** Bevor ein Server zur ersten Domäne einer neuen Gesamtstruktur
wird, muss das lokale Administratorkonto ein den Richtlinien entsprechendes
Kennwort besitzen, da dieses Konto zum Domänen-Administratorkonto wird.

---

## Problem 3: NTDS und Netlogon starten nach Heraufstufung nicht

**Symptom:**
Nach einer scheinbar erfolgreichen Heraufstufung und Neustart:
```powershell
Get-Service NTDS, Netlogon
# Beide: Status = Stopped

net start netlogon
# Anmeldedienst konnte nicht gestartet werden. Der Dienst hat keinen
# Fehler gemeldet.
```

**Diagnose (schrittweise):**
1. `Get-Service NTDS` → gestoppt, ließ sich nicht manuell starten
2. Ereignisprotokoll geprüft (Event ID 3095, Quelle NETLOGON):
   > "Dieser Computer ist als Mitglied einer Arbeitsgruppe konfiguriert,
   > nicht als Mitglied einer Domäne."
3. Bestätigung per PowerShell:
   ```powershell
   Get-WmiObject Win32_ComputerSystem | Select Name, Domain, PartOfDomain, Workgroup
   # PartOfDomain: False, Workgroup: WORKGROUP
   ```
4. Zusätzlich zeigte `Get-WindowsFeature`, dass die **DNS-Server-Rolle**
   entgegen der üblichen Automatik bei der AD-DS-Heraufstufung nicht
   installiert worden war

**Ursache:** Die erste Heraufstufung war nicht vollständig durchgelaufen bzw.
wurde nicht korrekt übernommen – der Server blieb faktisch ein einfaches
Mitglied einer Arbeitsgruppe, obwohl der Assistent zuvor "erfolgreich"
durchgelaufen war.

**Lösung:**
1. DNS-Rolle manuell nachinstalliert:
   ```powershell
   Install-WindowsFeature -Name DNS -IncludeManagementTools
   ```
2. Heraufstufung mit der Option **"Neue Gesamtstruktur hinzufügen"** komplett
   neu durchgeführt (diesmal inklusive DNS-Server-Häkchen in den
   Domänencontrolleroptionen)
3. Nach dem Neustart Kontrolle:
   ```powershell
   whoami                          # hybridlab\administrator
   Get-Service NTDS, Netlogon, DNS # alle: Running
   nslookup hybridlab.local        # löst korrekt zu 192.168.100.10 auf
   ```

**Erkenntnis:** Ein "erfolgreich" abgeschlossener Konfigurationsassistent
garantiert nicht automatisch den gewünschten Endzustand. Die Kombination aus
Ereignisprotokoll-Auswertung (Event-IDs) und gezielten PowerShell-Diagnose-
befehlen (`Get-Service`, `Get-WindowsFeature`, `Get-WmiObject`) war nötig, um
die tatsächliche Ursache statt nur das Symptom zu finden.
