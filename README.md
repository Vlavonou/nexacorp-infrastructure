# NexaCorp Infrastructure IT

> Déploiement d'une infrastructure IT complète simulant un environnement professionnel réel, avec application des bonnes pratiques de sécurité à chaque couche (réseau, système, application, identité).

---

## Présentation du projet

Ce projet simule le système d'information d'une PME fictive appelée **NexaCorp**. L'infrastructure tourne entièrement sur une seule machine hôte via **VMware Workstation** et comprend cinq machines virtuelles interconnectées via trois segments réseau distincts, gérés par un firewall pfSense.

---

## Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────┐
│         pfSense (Firewall)          │
│  WAN: 192.168.110.146               │
│  LAN: 192.168.10.1/24              │
│  DMZ: 192.168.20.1/24              │
└──────────┬──────────────┬───────────┘
           │              │
    ┌──────▼──────┐  ┌───▼────────┐
    │  LAN         │  │    DMZ     │
    │ VMnet2       │  │  VMnet3    │
    │ 192.168.10.x │  │192.168.20.x│
    └──────────────┘  └────────────┘
    │  WinSrv-AD       │  Linux-Web
    │  Linux-Log       │  192.168.20.10
    │  Win10-Client    │
```

---

## Machines Virtuelles

| VM | Rôle | OS | IP | Réseau |
|---|---|---|---|---|
| VM1 — pfSense | Firewall / Routeur | pfSense 2.8 CE | WAN/LAN/DMZ | VMnet8/2/3 |
| VM2 — WinSrv-AD | Active Directory / DNS / DHCP | Windows Server 2022 | 192.168.10.10 | VMnet2 |
| VM3 — Linux-Web | Serveur Web Apache | Ubuntu Server 24.04 | 192.168.20.10 | VMnet3 |
| VM4 — Linux-Log | Centralisation des logs | Ubuntu Server 24.04 | 192.168.10.20 | VMnet2 |
| VM5 — Win10-Client | Poste utilisateur | Windows 10 Pro | DHCP .10.100 | VMnet2 |

---

## Ressources Allouées

| VM | RAM | vCPU | Disque |
|---|---|---|---|
| pfSense | 512 Mo | 1 | 8 Go |
| WinSrv-AD | 2.5 Go | 2 | 40 Go |
| Linux-Web | 1 Go | 1 | 20 Go |
| Linux-Log | 1 Go | 1 | 20 Go |
| Win10-Client | 2 Go | 2 | 40 Go |
| **Total** | **7 Go / 12 Go** | **7 vCPUs** | **128 Go** |

---

## Technologies Utilisées

| Catégorie | Technologie |
|---|---|
| Hyperviseur | VMware Workstation |
| Firewall | pfSense 2.8 Community Edition |
| Annuaire | Active Directory (Windows Server 2022) |
| Serveur web | Apache2 (Ubuntu Server 24.04) |
| Centralisation logs | Rsyslog + NXLog |
| Protection bruteforce | Fail2ban |
| Audit système | Nmap, Lynis |
| SSH | OpenSSH (port 2222, clé ed25519) |
| Firewall local | UFW (Ubuntu) |
| Intégrité fichiers | AIDE |

---

## Fonctionnalités Implémentées

### Réseau et Firewall
- Segmentation en trois zones : WAN, LAN, DMZ
- Règles firewall pfSense avec principe du moindre privilège
- NAT Outbound hybride pour LAN et DMZ
- Port forwarding HTTP/HTTPS vers le serveur web
- Isolation DMZ → LAN (sauf logs port 514)

### Active Directory
- Domaine `nexacorp.local` avec contrôleur de domaine NEXASRV01
- Unités d'organisation : Utilisateurs, Groupes, Ordinateurs
- Comptes : alice.dupont (IT-Admins), bob.martin (Commerciaux), claire.ndong (Direction), svc-web
- DHCP scope 192.168.10.100-200 avec options complètes

### GPO (Group Policy Objects)
- **GPO1** — Politique de mots de passe (longueur 10, complexité, expiration 90j, verrouillage 5 tentatives)
- **GPO2** — Restrictions accès (panneau de config, connexion locale/RDP par groupe)
- **GPO3** — Journalisation avancée (Account Logon, Logon/Logoff, Object Access, Privilege Use)

### Sécurité SSH
- Port non standard : 2222
- Authentification par clé ed25519 uniquement
- PermitRootLogin désactivé
- AllowUsers restreint aux comptes dédiés (adminweb, adminlog)
- Bannière légale configurée

### Centralisation des Logs
- **Rsyslog** en mode serveur sur Linux-Log (UDP/TCP port 514)
- **Rsyslog** client sur Linux-Web (logs système + logs Apache)
- **NXLog** sur WinSrv-AD (événements Windows Security vers Syslog)
- Organisation par hôte : `/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log`

### Fail2ban
- Prison SSH : ban après 3 échecs en 10 min, durée 1h
- Prison Apache : ban après 5 échecs en 10 min, durée 1h

### Hardening Linux
- Mises à jour automatiques (unattended-upgrades)
- Services inutiles désactivés (ModemManager, fwupd, multipathd, upower)
- Paramètres kernel sécurisés (sysctl : ip_forward=0, syncookies=1, ASLR=2)
- Comptes système verrouillés
- Logs sudo configurés
- Intégrité des fichiers avec AIDE

### Audit de Sécurité
- 5 failles introduites volontairement et corrigées
- Outils utilisés : Nmap, Lynis, PowerShell AD, find/stat
- Rapport complet avec détection et validation des corrections

---

## Flux Réseau

```
Internet ──► WAN pfSense ──► DMZ:80/443 (HTTP/HTTPS vers Linux-Web)

LAN ──────► WAN          (NAT, accès internet)
LAN ──────► DMZ:80/443   (accès site web)
LAN ──────► DMZ:2222     (SSH vers Linux-Web)
LAN ──────► pfSense:443  (administration web pfSense)

DMZ ──────► LAN:514      (logs Rsyslog vers Linux-Log)  ✅ autorisé
DMZ ──────► LAN          (tout le reste)                ❌ bloqué
DMZ ──────► Internet     (apt update, mises à jour)     ✅ autorisé
```

---

## Structure du Dépôt

```
nexacorp-infrastructure/
│
├── README.md
├── docs/
│   ├── NexaCorp_Documentation_Technique.pdf
│   └── NexaCorp_Documentation_Technique.docx
├── configs/
│   ├── pfsense/
│   │   └── regles-firewall.md
│   ├── linux/
│   │   ├── rsyslog-server.conf
│   │   ├── rsyslog-client.conf
│   │   ├── sshd_config
│   │   ├── jail.local
│   │   └── 99-hardening.conf
│   └── windows/
│       └── nxlog.conf
└── screenshots/
    └── (captures d'écran)
```

---

## Audit de Sécurité — Résumé

| # | Faille | Machine | Niveau | Corrigée |
|---|---|---|---|---|
| 1 | SSH root autorisé + auth mot de passe | Linux-Web | CRITIQUE | ✅ |
| 2 | Service Telnet installé et actif | Linux-Web | CRITIQUE | ✅ |
| 3 | Fichier sensible chmod 644 | Linux-Web | ÉLEVÉ | ✅ |
| 4 | Compte AD sans expiration mot de passe | WinSrv-AD | MOYEN | ✅ |
| 5 | Règle firewall DMZ trop permissive | pfSense | ÉLEVÉ | ✅ |

---

## Compétences Démontrées

- Administration réseau et firewall (pfSense, NAT, règles de filtrage)
- Active Directory et GPO (Windows Server 2022)
- Administration Linux (Ubuntu Server, SSH, UFW, sysctl)
- Centralisation et analyse de logs (Rsyslog, NXLog)
- Sécurité système (Fail2ban, AIDE, hardening)
- Audit de sécurité (Nmap, Lynis, pentest basique)
- Virtualisation (VMware Workstation, segmentation réseau)

---

## Auteur

Projet réalisé dans le cadre d'un laboratoire de simulation d'infrastructure IT professionnelle.

---

*Infrastructure NexaCorp — Mai 2026*
