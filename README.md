
# Hybrid Cloud Lab – On-Prem Active Directory, Azure Arc & Azure Backup

Ein praxisnahes Hybrid-Cloud-Projekt, in dem ich einen lokalen Windows Server als
Domänencontroller aufgesetzt, per **Azure Arc** an Azure angebunden und mit
**Azure Backup / Recovery Services** gesichert und wiederhergestellt habe.

## 🎯 Projektziel

Simulation eines typischen Hybrid-Cloud-Szenarios, wie es in kleinen und mittleren
Unternehmen vorkommt: ein On-Premises-Server, der zentral über Azure verwaltet,
überwacht und gesichert wird – als Vorbereitung für spätere Themen wie
Microsoft Entra Connect Cloud Sync.

## 🏗️ Architektur

```
┌─────────────────────────────┐
│   VMware Workstation (Host) │
│                              │
│  ┌────────────────────────┐  │
│  │  SRV-HYBRID01          │  │
│  │  Windows Server 2025   │  │
│  │  192.168.100.10        │  │
│  │                        │  │
│  │  • AD DS (hybridlab.local) │
│  │  • DNS Server          │  │
│  │  • Azure Arc Agent     │  │
│  │  • Azure Backup (MARS) │  │
│  └───────────┬────────────┘  │
│              │ NAT (VMnet8)  │
└──────────────┼───────────────┘
               │
        ┌──────▼───────┐
        │  Azure Cloud │
        │  • Azure Arc │
        │  • Recovery  │
        │    Services  │
        │    Vault     │
        └──────────────┘
```

## 🛠️ Verwendete Technologien

- **Windows Server 2025** (Domänencontroller)
- **VMware Workstation** (Virtualisierung, NAT-Netzwerk-Konfiguration)
- **Active Directory Domain Services (AD DS)** + DNS
- **Azure Arc** (Server-Onboarding, hybride Verwaltung)
- **Azure Backup / Microsoft Azure Recovery Services (MARS) Agent**
- **PowerShell** (Diagnose, Service- und Feature-Management)

## 📋 Durchgeführte Schritte

1. **VM-Setup**: Windows Server 2025 in VMware Workstation installiert, Computername
   und statische IP-Konfiguration gesetzt (`192.168.100.10/24`)
2. **Domain Controller**: Server zum Domänencontroller für `hybridlab.local`
   heraufgestuft, inkl. AD DS- und DNS-Rolle
3. **Azure Arc Onboarding**: Server über Onboarding-Skript mit Azure Arc verbunden
   (Connected Machine Agent, Device-Code-Anmeldung)
4. **Azure Backup**: Regelmäßige Sicherung über den MARS-Agent eingerichtet,
   Wiederherstellungstest erfolgreich durchgeführt

## 🖼️ Screenshots

### Domain Controller erfolgreich konfiguriert
`whoami` bestätigt Domänenmitgliedschaft, `nslookup` löst die eigene Domäne korrekt auf:

![DC Funktionalität](screenshots/01-dc-whoami-nslookup.png)

### Netzwerkkonfiguration
Vollständige `ipconfig /all`-Ausgabe mit korrektem Hostnamen, DNS-Suffix und IP-Konfiguration:

![Netzwerkkonfiguration](screenshots/02-ipconfig-all.png)

### Azure Backup – Sicherung & Wiederherstellung erfolgreich
Letzte Sicherung und letzte Wiederherstellung beide erfolgreich abgeschlossen:

![Azure Backup Erfolg](screenshots/03-azure-backup-success.png)

## 🐞 Troubleshooting (ausgewählte Beispiele)

Details dazu in [docs/troubleshooting.md](docs/troubleshooting.md). Kurzüberblick:

| Problem | Ursache | Lösung |
|---|---|---|
| Server konnte eigenes Gateway nicht erreichen | VMware NAT-Subnetz (VMnet8) wich vom geplanten Adressbereich ab | Subnetz im Virtual Network Editor auf `192.168.100.0/24` angepasst |
| Domänencontroller-Heraufstufung schlug fehl | Lokales Administrator-Kennwort war leer | Kennwort über `net user` gesetzt, Voraussetzungsprüfung erneut durchlaufen |
| Nach Heraufstufung: NTDS/Netlogon starteten nicht | DNS-Rolle wurde bei der ersten Installation nicht mitinstalliert, Server blieb "Workgroup"-Mitglied | DNS-Rolle nachinstalliert, Heraufstufung mit "Neue Gesamtstruktur" sauber wiederholt |

## 📚 Gelernte Konzepte

- Unterschied zwischen Arbeitsgruppe und Domänenmitgliedschaft und dessen Auswirkung
  auf abhängige Dienste (NTDS, Netlogon)
- Zusammenspiel von VMware-Netzwerkeinstellungen (NAT/VMnet) und der internen
  Windows-Netzwerkkonfiguration
- Systematische Fehlerdiagnose über Ereignisprotokolle und PowerShell
  (`Get-WinEvent`, `Get-Service`, `Get-WindowsFeature`)
- Azure Arc als Brücke zwischen On-Premises-Servern und der Azure-Verwaltungsebene
- Azure Backup End-to-End: Registrierung, Sicherungsplan, Wiederherstellung über
  temporär eingebundenes Volume

## 👤 Über mich

Erstellt von Maryam Pourkhatoun im Rahmen meiner Weiterbildung zur Cloud Engineerin
am Digital Career Institute (DCI) Berlin, als praktische Vorbereitung auf die
AZ-900- und AZ-104-Zertifizierung.
