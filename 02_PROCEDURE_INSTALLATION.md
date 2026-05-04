# Procédure d'Installation - Serveur NPS et Switchs Cisco (802.1X)

> **Auteur :** Edib Saoud
> **Date :** 10/11/2025 - 12/04/2026
> **Version :** 1.0
> **Statut :** Validé

---

## 1. Préparation du Serveur NPS (Windows Server 2022)

1. Depuis le Gestionnaire de Serveur, ajouter le rôle **Services de stratégie et d'accès réseau (NPS)**.
2. Ouvrir la console **Serveur de stratégie réseau (NPS)**.
3. Faire un clic-droit sur NPS(Local) au sommet de l'arborescence et sélectionner **Inscrire le serveur dans Active Directory**.

### 1.1 Ajout des Clients RADIUS (Authentificateurs)
Il s'agit d'enregistrer le Switch et la borne AP Cisco pour qu'ils soient autorisés à dialoguer avec le serveur.

1. Dans `Clients et serveurs RADIUS` > `Clients RADIUS`, faire un clic-droit **Nouveau**.
2. **Nom convivial :** `Switch-Cisco-Etage1`
3. **Adresse IP :** `192.168.50.253`
4. **Secret partagé :** Saisir un secret fort (ex: `P@ssw0rdRadiusIRIS!`). *Cette clé sera indispensable côté Cisco*.

![NPS - Clients RADIUS enregistrés](./images/clients_radius.png)

### 1.2 Configuration des Stratégies (VLAN Dynamique)
1. Créer une nouvelle **Stratégie réseau** (ex: `Acces-8021x-Etudiants`).
2. **Conditions :** Ajouter "Groupes Windows" et sélectionner le groupe AD des étudiants IRIS.
3. **Autorisations :** Autoriser l'accès.
4. **Méthode EAP :** Ajouter `Microsoft : EAP protégé (PEAP)` et valider la méthode MS-CHAP v2.
5. **Attributs RADIUS (Affectation de VLAN) :** Ajouter les paramètres Standards suivants :
   - `Tunnel-Type` : *Virtual LANs (VLAN)*
   - `Tunnel-Medium-Type` : *802 (Includes all 802 media)*
   - `Tunnel-Pvt-Group-ID` : *10* (L'ID du VLAN IRIS défini dans l'étude technique).

*Répéter cette stratégie pour les Professeurs avec l'ID `20` et l'Administration avec l'ID `30`.*

---

## 2. Configuration du Switch Cisco (Client RADIUS)

Se connecter en mode console ou SSH sur le switch (`192.168.50.253`).

### 2.1 Activation AAA et RADIUS
```cisco
enable
configure terminal

! Activer le modèle AAA
aaa new-model

! Configurer l'authentification 802.1X
aaa authentication dot1x default group radius

! Déclarer le serveur NPS
radius server NPS-WIN
 address ipv4 192.168.50.200 auth-port 1812 acct-port 1813
 key P@ssw0rdRadiusIRIS!

! Activer le 802.1X globalement sur le switch
dot1x system-auth-control
```

### 2.2 Configuration d'un Port (Interface Gigabit)
Il faut configurer les ports des salles de cours pour forcer l'authentification.
```cisco
interface GigabitEthernet0/1
 description PORT-SALLE-COURS
 switchport mode access
 ! Assigner le port au VLAN PRE_AUTH par défaut en attendant l'authentification
 switchport access vlan 99 
 
 ! Paramètres 802.1X
 authentication port-control auto
 dot1x pae authenticator
```
Sauvegarder la configuration (`write memory`).

---

## 3. Configuration des Clients Windows 10/11

Afin que le PC puisse envoyer les informations de connexion AD lors du branchement du câble réseau :
1. Sur le PC client, ouvrir `services.msc`.
2. Démarrer et mettre en Automatique le service **Configuration automatique de réseau câblé** (`Wired AutoConfig` / `dot3svc`).
3. Dans les propriétés de la carte réseau, l'onglet **Authentification** devient visible. S'assurer que le protocole PEAP est activé.
