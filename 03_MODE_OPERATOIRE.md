# Mode opératoire d'exploitation (N2/N3)

> **Auteur :** Edib Saoud  
> **Version :** 2.0  
> **Cible :** exploitation système/réseau  
> **Périmètre :** AD DS, NPS, SW1, AP1, R1, services Debian.

---

## 1. Procédure standard d'onboarding utilisateur

### 1.1 Créer l'utilisateur (AD en CLI)

```powershell
Import-Module ActiveDirectory
$pwd = ConvertTo-SecureString "<PASSWORD_TEMPORAIRE>" -AsPlainText -Force

New-ADUser -Name "prenom.nom" `
  -SamAccountName "prenom.nom" `
  -AccountPassword $pwd `
  -Enabled $true `
  -Path "OU=Iris_Lab,DC=iris,DC=local"
```

### 1.2 Assigner le VLAN par appartenance groupe

```powershell
# Étudiant
Add-ADGroupMember -Identity "G_VLAN10_Iris" -Members "prenom.nom"

# Prof
# Add-ADGroupMember -Identity "G_VLAN20_Profs" -Members "prenom.nom"

# Admin
# Add-ADGroupMember -Identity "G_VLAN30_Admin" -Members "prenom.nom"
```

### 1.3 Contrôle post-création

```powershell
Get-ADUser prenom.nom -Properties MemberOf | Select-Object SamAccountName,MemberOf
```

**Attendu :** le groupe AD correspond au VLAN voulu.

---

## 2. Procédure de contrôle d'accès 802.1X

### 2.1 Côté NPS (Windows)

1. Ouvrir `eventvwr.msc`.
2. Aller dans `Rôles de serveur > NPS`.
3. Vérifier :
   - **acceptation** (ex: ID 6272) ;
   - **rejet** (mot de passe faux, groupe non conforme, méthode EAP non conforme).

**Capture attendue :**
- événement succès + nom utilisateur + policy appliquée.

**Chemin de dépôt :** `./preuves/03_mode_operatoire/01_nps_event_succes_policy.png`

### 2.2 Côté switch SW1

```cisco
show authentication sessions interface gi1/0/13 details
show dot1x interface gi1/0/13 details
show radius statistics
```

Points à contrôler :
- état `AUTHORIZED` quand l'utilisateur est valide ;
- serveurs RADIUS joignables ;
- pas d'explosion d'échecs RADIUS.

### 2.3 Côté AP

```cisco
show wlan summary
show aaa servers
show ntp status
```

Points à contrôler :
- WLAN `IRIS-WIFI` actif ;
- serveur NPS `.200` joignable ;
- horloge synchronisée.

---

## 3. Exploitation des services complémentaires

### 3.1 Sauvegardes de configuration Cisco vers TFTP

Sur chaque équipement Cisco :

```cisco
write memory
```

Effet attendu :
- génération d'un fichier de backup dans les dossiers TFTP :
  - `/srv/tftp/ROUTER/`
  - `/srv/tftp/SWITCH/`
  - `/srv/tftp/AP/`

### 3.2 Vérification NTP

```cisco
show ntp associations
show clock detail
```

Attendu :
- source NTP interne visible (`192.168.50.200` ou `.100`);
- horodatage cohérent dans les logs.

### 3.3 Vérification Syslog/SNMP

```cisco
show logging
show snmp community
```

Attendu :
- envoi de logs vers `192.168.50.100` ;
- communautés SNMP présentes (en production: stockées en coffre secret).

---

## 4. Matrice de dépannage rapide

| Symptôme | Contrôle prioritaire | Cause probable | Action corrective |
|:--|:--|:--|:--|
| Utilisateur n'a pas d'accès | Event Viewer NPS + `show auth sessions` | Mot de passe faux / groupe AD absent | Corriger creds ou appartenance groupe |
| Utilisateur toujours en VLAN 99 | Attributs NPS `Tunnel-Pvt-Group-ID` | Policy mal ordonnée / condition incomplète | Corriger l'ordre des policies 10/20/30 |
| Aucun log NPS | `show radius statistics` | Secret RADIUS incohérent ou route KO | Réaligner clé `<SECRET_RADIUS_COMMUN>` et routage |
| Horodatages incohérents | `show ntp status` | NTP down | Réactiver source NTP interne |
| Config perdue après reboot | Vérifier archive path TFTP | Backup non exécuté ou TFTP indisponible | Corriger `archive path` puis `wr mem` |

---

## 5. Changement contrôlé (ajout/modif policy VLAN)

1. Créer d'abord le groupe AD cible.
2. Créer la policy NPS en bas de liste.
3. Tester avec 1 compte pilote.
4. Contrôler Event Viewer + VLAN obtenu.
5. Monter la policy au bon ordre.
6. Documenter la modification (date, opérateur, impact).

---

## 6. Check-list hebdomadaire d'exploitation

1. Contrôler 5 événements NPS succès + 5 rejets récents.
2. Vérifier l'horloge de SW1/AP1/R1 (NTP).
3. Vérifier présence des backups TFTP de la semaine.
4. Vérifier disponibilité de `192.168.50.200` et `192.168.50.100`.
5. Vérifier qu'aucun secret n'est exposé dans les exports diffusés.
