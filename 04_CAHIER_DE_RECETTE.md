# Cahier de Recette et Tests - Accès Sécurisés 802.1X

> **Auteur :** Edib Saoud
> **Date :** 10/11/2025 - 12/04/2026
> **Version :** 1.0
> **Statut :** Validé

---

## 1. Objectifs de la Recette

L'objectif de cette phase de test est de certifier que l'infrastructure réseau bloque bien toute tentative de connexion illégitime. Un équipement ne disposant pas de comptes Active Directory valides doit être isolé, tandis qu'un compte valide doit être redirigé vers son VLAN métier.

## 2. Tableau de Recette Sécurité (802.1X)

| N° | Catégorie | Scénario de Test | Procédure | Résultat Attendu | Statut |
|:---|:---|:---|:---|:---|:---:|
| **T1** | **Isolation** | Branchement d'un poste non configuré | Brancher un PC personnel sur la prise RJ45 d'une salle de classe. | Le port physique reste "UP" mais la couche réseau est bloquée (`UNAUTHORIZED`). Le PC reste dans le VLAN de quarantaine 99 (`PRE_AUTH`). Aucune IP LAN n'est attribuée. | ✅ Validé |
| **T2** | **Connexion** | Authentification valide Étudiant | Activer l'authentification PEAP sur le PC et se connecter avec un compte AD de l'OU `Etudiants`. | Le switch passe le port en `AUTHORIZED`. Le serveur NPS renvoie l'ID de VLAN 10. Le DHCP attribue une IP de la plage IRIS. | ✅ Validé |
| **T3** | **Routage** | VLAN Dynamique Profs | Se déconnecter, puis rebrancher avec un compte AD du groupe "Professeurs". | Le port bascule automatiquement sur le VLAN 20. L'adresse IP change pour s'adapter au sous-réseau Profs. | ✅ Validé |
| **T4** | **Rejet AD** | Mot de passe erroné | Tenter une connexion filaire ou Wi-Fi avec un mot de passe AD incorrect. | Une fenêtre Windows indique "Échec d'authentification". Les logs NPS (EventVwr) affichent un événement de rejet "Mot de passe incorrect". Le port Cisco reste bloqué. | ✅ Validé |
| **T5** | **Management** | Isolation du réseau d'admin | Tenter d'accéder au serveur NPS (`192.168.50.200`) depuis un PC Etudiant (VLAN 10). | Le ping et le SSH échouent. Seuls les équipements du VLAN 50 (Management) peuvent administrer le serveur et le switch. | ✅ Validé |

---

## 3. Synthèse de la Recette

L'implémentation du protocole 802.1X via le serveur NPS a permis d'éradiquer la vulnérabilité d'usurpation d'accès réseau. 
- Les VLANs sont désormais affectés automatiquement en fonction du rôle de l'utilisateur, facilitant le travail de l'administration IT.
- L'Observateur d'événements Windows offre une traçabilité parfaite des connexions.

**Décision finale : L'architecture AAA est validée et déployée sur la totalité des ports LAN de l'école IRIS.**
