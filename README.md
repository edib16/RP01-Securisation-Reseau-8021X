# RP01 - Sécurisation réseau 802.1X / NPS / Active Directory

> **Auteur :** Edib Saoud  
> **Période :** 10/11/2025 - 12/04/2026  
> **Contexte :** BTS SIO SISR - IRIS Mediaschool  
> **Objectif :** Remplacer les accès réseau permissifs (PSK/ports ouverts) par une authentification nominative 802.1X adossée à AD.

---

## Vue d'ensemble du projet

Cette réalisation met en place une architecture AAA complète :
- **Supplicant** : postes Windows / smartphones.
- **Authenticators** : switch Cisco 2960-S + AP Cisco C9105 (EWC).
- **RADIUS** : Windows Server 2022 avec rôle **NPS**.
- **Référentiel d'identité** : **Active Directory** `iris.local`.
- **Segmentation dynamique** : affectation VLAN selon groupe AD.

### Équipements et adresses de management

| Équipement | Rôle | IP |
|:--|:--|:--|
| Windows Server 2022 | AD DS + NPS | `192.168.50.200` |
| Switch `SW1-IRIS` | 802.1X filaire | `192.168.50.253` |
| Routeur `R1-IRIS` | Inter-VLAN + ACL + VRRP | `192.168.50.254` (VIP) / `192.168.50.251` (physique) |
| AP `AP1-IRIS` | Wi-Fi 802.1X | `192.168.50.150` |
| Debian `debian-rp` | TFTP + Syslog + DHCP Kea + VRRP backup | `192.168.50.100` / `192.168.50.252` |

### Segmentation validée

| Groupe AD | VLAN | Sous-réseau | Plage DHCP (source conf) |
|:--|:--:|:--|:--|
| `G_VLAN10_Iris` | 10 | `192.168.10.0/24` | `192.168.10.10 - 192.168.10.200` |
| `G_VLAN20_Profs` | 20 | `192.168.20.0/24` | `192.168.20.10 - 192.168.20.200` |
| `G_VLAN30_Admin` | 30 | `192.168.30.0/24` | `192.168.30.10 - 192.168.30.200` |
| (invités) | 40 | `192.168.40.0/24` | Dépend du serveur DHCP central |
| management | 50 | `192.168.50.0/24` | `192.168.50.100 - 192.168.50.150` |
| pré-auth | 99 | VLAN de quarantaine L2 | Pas d'usage utilisateur final |

---

## Chronologie réelle de mise en œuvre

1. Installation de Windows Server 2022 et configuration IP.
2. Promotion en contrôleur de domaine (`iris.local`) via CLI.
3. Création des OU, groupes et utilisateurs AD via PowerShell.
4. Installation du rôle NPS.
5. Configuration NPS en GUI : clients RADIUS + policies VLAN 10/20/30.
6. Configuration Cisco (switch, AP, routeur) pour AAA/RADIUS/802.1X.
7. Mise en place des services complémentaires : NTP, TFTP backup, Syslog/SNMP, DHCP Kea, VRRP.
8. Tests fonctionnels (laptops Windows + téléphones) et validation des VLAN dynamiques.

---

## Parcours de lecture de la documentation

1. [01_DOSSIER_CHOIX_TECHNIQUE.md](01_DOSSIER_CHOIX_TECHNIQUE.md)  
   Décisions techniques, architecture cible, adressage, services complémentaires.
2. [02_PROCEDURE_INSTALLATION.md](02_PROCEDURE_INSTALLATION.md)  
   Guide d'installation détaillé, étape par étape (CLI + GUI).
3. [03_MODE_OPERATOIRE.md](03_MODE_OPERATOIRE.md)  
   Exploitation quotidienne, maintenance, dépannage.
4. [04_CAHIER_DE_RECETTE.md](04_CAHIER_DE_RECETTE.md)  
   Tests réalisés, résultats observés, conclusion de recette.

---

## Références techniques (run conf)

- `runconf-r1-iris.txt` : configuration du routeur `R1-IRIS`.
- `runconf-sw1-iris.txt` : configuration du switch `SW1-IRIS`.
- `runconf-ap1-iris.txt` : configuration de la borne/AP `AP1-IRIS`.
- `runconf-routeur-virtuel-debian.txt` : configuration Debian (`interfaces`, `keepalived`, `kea`).
- `Fiche RP01 Edib Saoud.pdf` : contexte et cadrage de la réalisation.

---

## Où déposer les preuves (captures)

| Source | Dossier | Ce qu'il faut capturer |
|:--|:--|:--|
| `02_PROCEDURE_INSTALLATION.md` | `./preuves/02_procedure_installation/` | 6 captures: IP serveur, AD domain, ADUC (OU/groupes), `show run` SW1, `show interfaces trunk`, profile AP `aaa-override` |
| `03_MODE_OPERATOIRE.md` | `./preuves/03_mode_operatoire/` | 1 capture: event NPS de succes (utilisateur + policy appliquee) |
| `04_CAHIER_DE_RECETTE.md` | Aucun dépôt requis | aucune capture demandée pour cette remise |
