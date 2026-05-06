# Procédure d'installation complète (chronologique)

> **Auteur :** Edib Saoud  
> **Période :** 10/11/2025 - 12/04/2026  
> **Version :** 2.0  
> **Important :** tous les secrets sont volontairement masqués dans ce document.

---

## 0. Pré-requis

### 0.1 Matériel / VM utilisés

- Windows Server 2022 (AD DS + NPS) : `192.168.50.200`
- Switch Cisco 2960-S : `192.168.50.253`
- AP Cisco C9105 (EWC) : `192.168.50.150`
- Routeur Cisco 1941 : sous-interfaces VLAN + VRRP
- Debian (`debian-rp`) : TFTP + Syslog + DHCP Kea + Keepalived

### 0.2 Paramètres cibles

- Domaine AD : `iris.local`
- Groupes AD pour VLAN dynamiques :
  - `G_VLAN10_Iris`
  - `G_VLAN20_Profs`
  - `G_VLAN30_Admin`
- Méthode EAP : **PEAP + EAP-MSCHAPv2**
- VLAN d'attente (non authentifié) : **99**

---

## 1. Installation Windows Server 2022 (base)

### 1.1 Installation OS

1. Installer Windows Server 2022.
2. Appliquer les mises à jour.
3. Définir un mot de passe administrateur local.

### 1.2 Configuration réseau initiale (CLI PowerShell)

```powershell
# Exécuter en administrateur
Rename-Computer -NewName "SRV-ADNPS-IRIS"

New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.50.200 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.50.254

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.50.200

Restart-Computer
```

**Résultat attendu :**
- serveur joignable en `192.168.50.200`;
- nom machine correct;
- DNS local configuré.

**Capture attendue :**
- `ipconfig /all` du serveur.

**Chemin de dépôt :** `./preuves/02_procedure_installation/01_ipconfig_serveur.png`

---

## 2. Passage en contrôleur de domaine (AD DS en CLI)

### 2.1 Installation du rôle AD DS

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

### 2.2 Création de la forêt / domaine

```powershell
Install-ADDSForest `
  -DomainName "iris.local" `
  -DomainNetbiosName "IRIS" `
  -InstallDNS `
  -SafeModeAdministratorPassword (Read-Host "DSRM password" -AsSecureString) `
  -Force
```

Le serveur redémarre automatiquement.

**Résultat attendu :**
- domaine `iris.local` opérationnel;
- DNS AD actif;
- ouverture de session possible en `IRIS\Administrator`.

**Capture attendue :**
- console `Get-ADDomain` ou écran de connexion domaine.

**Chemin de dépôt :** `./preuves/02_procedure_installation/02_get-addomain_ou_logon_domaine.png`

---

## 3. Activation AD opérationnelle : OU, groupes, utilisateurs (CLI)

### 3.1 Création des OU

```powershell
Import-Module ActiveDirectory

New-ADOrganizationalUnit -Name "Iris_Lab" -Path "DC=iris,DC=local"
New-ADOrganizationalUnit -Name "SISR" -Path "DC=iris,DC=local"
New-ADOrganizationalUnit -Name "SLAM" -Path "DC=iris,DC=local"
New-ADOrganizationalUnit -Name "Administration" -Path "DC=iris,DC=local"
```

### 3.2 Création des groupes de sécurité pour VLAN dynamiques

```powershell
New-ADGroup -Name "G_VLAN10_Iris" -GroupScope Global -GroupCategory Security -Path "OU=Iris_Lab,DC=iris,DC=local"
New-ADGroup -Name "G_VLAN20_Profs" -GroupScope Global -GroupCategory Security -Path "OU=Iris_Lab,DC=iris,DC=local"
New-ADGroup -Name "G_VLAN30_Admin" -GroupScope Global -GroupCategory Security -Path "OU=Iris_Lab,DC=iris,DC=local"
```

### 3.3 Création d'utilisateurs de test

```powershell
$pwd = ConvertTo-SecureString "<PASSWORD_UTILISATEUR>" -AsPlainText -Force

New-ADUser -Name "edib" -SamAccountName "edib" -AccountPassword $pwd -Enabled $true -Path "OU=Iris_Lab,DC=iris,DC=local"
New-ADUser -Name "yanis.adidi" -SamAccountName "yanis.adidi" -AccountPassword $pwd -Enabled $true -Path "OU=SLAM,DC=iris,DC=local"

Add-ADGroupMember -Identity "G_VLAN10_Iris" -Members "edib"
Add-ADGroupMember -Identity "G_VLAN20_Profs" -Members "yanis.adidi"
```

**Résultat attendu :**
- OU visibles dans ADUC;
- groupes de sécurité disponibles;
- utilisateurs rattachés aux bons groupes.

**Capture attendue :**
- capture ADUC de l'arborescence OU et de l'appartenance groupe.

**Chemin de dépôt :** `./preuves/02_procedure_installation/03_aduc_ou_groupes_utilisateurs.png`

---

## 4. Mise en place du serveur NPS (GUI)

> Partie réalisée en interface graphique (conforme à la réalisation réelle).

### 4.1 Installation du rôle NPS

```powershell
Install-WindowsFeature NPAS -IncludeManagementTools
```

### 4.2 Inscription NPS dans Active Directory

1. Ouvrir **Network Policy Server (nps.msc)**.
2. Clic droit sur `NPS (Local)` > **Inscrire le serveur dans Active Directory**.

### 4.3 Déclaration des clients RADIUS

Créer 3 clients RADIUS :

| Nom client | IP | Rôle |
|:--|:--|:--|
| `SW1-IRIS` | `192.168.50.253` | Authenticator filaire |
| `AP1-IRIS` | `192.168.50.150` | Authenticator Wi-Fi |
| `R1-IRIS` | `192.168.50.254` | Routeur / équipements réseau |

Pour chaque client :
- activer le client RADIUS;
- définir un secret partagé : `<SECRET_RADIUS_COMMUN>`;
- fournisseur : RADIUS standard.

### 4.4 Connection Request Policy (traitement local)

Créer une policy `Auth_Locale_WiFi` :
- Type de traitement : authentification locale;
- source : clients RADIUS déclarés.

### 4.5 Policies réseau d'autorisation (VLAN dynamiques)

Créer 3 policies réseau, dans cet ordre :
1. `Acces_VLAN10_IRIS`
2. `Acces_VLAN20_IRIS`
3. `Acces_VLAN30_IRIS`

Paramètres communs :
- **Conditions** :
  - Windows Group correspondant (`G_VLAN10_Iris`, `G_VLAN20_Profs`, `G_VLAN30_Admin`)
  - NAS Port Type : Ethernet + Wireless (802.11)
- **Accès** : `Grant access`
- **Contrainte EAP** : PEAP + EAP-MSCHAPv2
- **Settings RADIUS** :
  - Tunnel-Type = VLAN
  - Tunnel-Medium-Type = 802
  - Tunnel-Pvt-Group-ID = `10` ou `20` ou `30`

---

## 5. Configuration du switch Cisco SW1-IRIS

### 5.1 AAA + RADIUS + 802.1X global

```cisco
conf t
aaa new-model
aaa group server radius RAD_GROUP
 server name Windows_NPS
 server name MON_RADIUS

aaa authentication dot1x default group radius
aaa authorization network default group radius
dot1x system-auth-control

radius server Windows_NPS
 address ipv4 192.168.50.200 auth-port 1812 acct-port 1813
 key <SECRET_RADIUS_COMMUN>

radius server MON_RADIUS
 address ipv4 192.168.50.100 auth-port 1812 acct-port 1813
 key <SECRET_RADIUS_SECOURS>

ip radius source-interface Vlan50
end
wr mem
```

### 5.2 Ports d'accès 802.1X (quarantaine VLAN 99)

```cisco
conf t
interface range GigabitEthernet1/0/13-1/0/52
 description ZONE_UTILISATEURS_DYNAMIQUE
 switchport mode access
 switchport access vlan 99
 authentication port-control auto
 authentication violation protect
 dot1x pae authenticator
 dot1x timeout tx-period 10
 spanning-tree portfast
end
wr mem
```

### 5.3 Uplink et management

- Uplink routeur : `Gi1/0/1` trunk VLAN `10,20,30,40,50,99`.
- AP : `Gi1/0/3` trunk natif VLAN 50.
- SVI management : `Vlan50 = 192.168.50.253/24`.
- Passerelle switch : `ip default-gateway 192.168.50.254`.

**Capture attendue :**
- `show run | section aaa|radius|dot1x`
- `show interfaces trunk`

**Chemin de dépôt :**
- `./preuves/02_procedure_installation/06_sw1_showrun_aaa_radius_dot1x.png`
- `./preuves/02_procedure_installation/07_sw1_show_interfaces_trunk.png`

---

## 6. Configuration AP Cisco (IRIS-WIFI)

### 6.1 AAA/RADIUS sur AP

```cisco
conf t
aaa new-model
aaa group server radius GRP-RADIUS
 server name WinNPS
 server name DebRADIUS

aaa authentication dot1x DOT1X-IRIS group GRP-RADIUS
aaa authorization network DOT1X-IRIS group GRP-RADIUS
dot1x system-auth-control
end
wr mem
```

### 6.2 WLAN + override VLAN

```cisco
conf t
wireless profile policy default-policy-profile
 aaa-override

wlan IRIS-WIFI 1 IRIS-WIFI
 security dot1x authentication-list DOT1X-IRIS
 no shutdown
end
wr mem
```

### 6.3 Mapping VLAN sur AP

Dans le profile flex/policy :
- `vlan-name Iris` -> VLAN 10
- `vlan-name Profs` -> VLAN 20
- `vlan-name Administration` -> VLAN 30

**Capture attendue :**
- écran policy profile montrant `aaa-override`.

**Chemin de dépôt :** `./preuves/02_procedure_installation/08_ap1_policy_profile_aaa_override.png`

---

## 7. Configuration routeur R1-IRIS (inter-VLAN + sécurité)

### 7.1 Sous-interfaces VLAN

`Gi0/0` est découpée en sous-interfaces :
- `.10` `192.168.10.251/24`
- `.20` `192.168.20.251/24`
- `.30` `192.168.30.251/24`
- `.40` `192.168.40.251/24`
- `.50` `192.168.50.251/24`
- `.99` native `192.168.99.251/24`

### 7.2 VRRP

VIP par VLAN : `.254` (priorité routeur = 50 ; Debian keepalived en backup déclaré).

### 7.3 ACL d'isolement

ACL `ACL_ETUDIANTS` appliquée en entrée sur VLAN 10 :
- autorise DHCP + accès services autorisés;
- bloque accès vers VLAN 20/30/40/50.

### 7.4 RADIUS/SNMP/Syslog/NTP

- RADIUS vers `.200` et `.100`;
- Syslog vers `.100`;
- NTP vers `.200` (prefer) puis `.100`.

---

## 8. Services complémentaires (section dédiée)

### 8.1 Debian - DHCP Kea

- Pools DHCP configurés :
  - VLAN10 : `192.168.10.10-192.168.10.200`
  - VLAN20 : `192.168.20.10-192.168.20.200`
  - VLAN30 : `192.168.30.10-192.168.30.200`
  - VLAN50 : `192.168.50.100-192.168.50.150`
- Réservation AP : `192.168.50.160`.

### 8.2 Debian - Keepalived / VRRP

- Instances `VI_10`, `VI_20`, `VI_30`, `VI_50` avec VIP `.254`.

### 8.3 NTP

- Switch : `ntp server 192.168.50.100 prefer`
- AP : `ntp server 192.168.50.200 prefer` + fallback `.100`
- Routeur : `ntp server 192.168.50.200 prefer` + `.100`

### 8.4 Sauvegardes TFTP automatiques

- R1 : `tftp://192.168.50.100/ROUTER/...`
- SW1 : `tftp://192.168.50.100/SWITCH/...`
- AP1 : `tftp://192.168.50.100/AP/...`

---

## 9. Validation finale après déploiement

### 9.1 Vérifications minimales

```cisco
# SW1
show dot1x all
show radius statistics
show authentication sessions

# R1
show vrrp brief
show ip interface brief | include 0/0.

# AP
show wlan summary
show ntp status
```

### 9.2 Résultat attendu

- un compte AD étudiant rejoint VLAN 10;
- un compte prof rejoint VLAN 20;
- un compte admin rejoint VLAN 30;
- un mauvais mot de passe reste rejeté;
- logs NPS disponibles pour audit.
