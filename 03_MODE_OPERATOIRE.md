# Mode Opératoire - Dépannage RADIUS / 802.1X

> **Auteur :** Edib Saoud
> **Date :** 10/11/2025 - 12/04/2026
> **Version :** 1.0
> **Cible :** Support Informatique Niveau 2 / Administrateurs

---

## 1. Dépannage côté NPS (Windows Server)

**Cas d'usage :** Un utilisateur (étudiant ou prof) signale qu'il n'a "pas d'Internet" en branchant son PC en salle de cours ou sur le Wi-Fi de l'école.

L'Observateur d'événements Windows de `192.168.50.200` est le meilleur outil de diagnostic.
1. Ouvrir l'**Observateur d'événements** (`eventvwr.msc`).
2. Naviguer vers : `Rôles de serveur` > `Serveur de stratégie réseau et d'accès (NPS)`.
3. Chercher les événements de refus (Erreur / Avertissement) :
   - **Raison "Utilisateur inconnu ou mot de passe incorrect" :** Le compte AD est expiré, verrouillé ou le mot de passe saisi est faux.
   - **Raison "Incompatibilité de méthode d'authentification" :** Le client essaie de se connecter sans utiliser PEAP-MS-CHAP v2.
   - **Raison "Condition de stratégie réseau non satisfaite" :** L'utilisateur n'appartient à aucun groupe AD défini dans les stratégies NPS (ex: Il n'est ni prof, ni étudiant).

---

## 2. Dépannage côté Switch Cisco

**Cas d'usage :** Vérifier l'état du blocage d'un port spécifique en ligne de commande.

### 2.1 Voir si un port est bloqué ou autorisé
Se connecter au Switch en SSH :
```cisco
show dot1x interface GigabitEthernet0/1 details
```
*Vérifier la ligne `Port Status` : elle doit être `AUTHORIZED` si un utilisateur valide est connecté, ou `UNAUTHORIZED` (bloqué) si la demande est échouée ou absente.*

### 2.2 Voir la communication avec le serveur RADIUS
```cisco
show radius statistics
```
*Permet de vérifier si le Switch arrive bien à envoyer les paquets UDP sur les ports 1812/1813 vers l'adresse `192.168.50.200`. Si des paquets sont rejetés, cela signifie souvent que le Secret Partagé est incorrect entre Cisco et NPS.*

---

## 3. Dépannage côté Client (Poste de travail)

Si le serveur NPS ne reçoit aucune demande de connexion :
1. Vérifier si le service réseau câblé est activé : `services.msc` > **Configuration automatique de réseau câblé** (Il doit être en cours d'exécution).
2. Si le client est hors-domaine (ex: BYOD), l'utilisateur doit impérativement configurer l'onglet Authentification de sa carte réseau Ethernet et décocher "Utiliser automatiquement mon nom de session Windows" pour qu'une fenêtre lui demande de taper ses identifiants `Iris.local`.
