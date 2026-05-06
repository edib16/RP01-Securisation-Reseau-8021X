# Cahier de recette - RP01 802.1X/NPS

> **Auteur :** Edib Saoud  
> **Version :** 2.0  
> **Périmètre testé :** authentification filaire + Wi-Fi, attribution VLAN dynamique, refus d'accès non conforme.

---

## 1. Référentiel de test

### 1.1 Contexte technique

- NPS/AD : `192.168.50.200`
- SW1 : `192.168.50.253`
- AP1 : `192.168.50.150`
- VLAN dynamiques cibles : 10, 20, 30
- VLAN de quarantaine : 99

### 1.2 Groupes de référence

- `G_VLAN10_Iris` -> VLAN 10
- `G_VLAN20_Profs` -> VLAN 20
- `G_VLAN30_Admin` -> VLAN 30

### 1.3 Plages DHCP de référence (source configuration)

- VLAN10 : `192.168.10.10 - 192.168.10.200`
- VLAN20 : `192.168.20.10 - 192.168.20.200`
- VLAN30 : `192.168.30.10 - 192.168.30.200`

---

## 2. Tests exécutés (validés)

| ID | Scénario | Procédure exécutée | Résultat attendu | Résultat observé | Statut |
|:--|:--|:--|:--|:--|:--:|
| T01 | Connexion filaire compte étudiant | Laptop Windows branché en RJ45 + identifiants AD étudiant | Auth OK + VLAN 10 + IP en `192.168.10.x` | Conforme (VLAN 10 obtenu, IP dans le pool VLAN10) | ✅ |
| T02 | Connexion Wi-Fi compte étudiant | Smartphone sur SSID `IRIS-WIFI` + identifiants AD étudiant | Auth OK + VLAN 10 + IP en `192.168.10.x` | Conforme | ✅ |
| T03 | Connexion filaire compte prof | Laptop Windows + compte du groupe `G_VLAN20_Profs` | Auth OK + VLAN 20 + IP en `192.168.20.x` | Conforme | ✅ |
| T04 | Connexion Wi-Fi compte prof | Smartphone + compte prof | Auth OK + VLAN 20 + IP en `192.168.20.x` | Conforme | ✅ |
| T05 | Connexion filaire compte admin | Laptop Windows + compte du groupe `G_VLAN30_Admin` | Auth OK + VLAN 30 + IP en `192.168.30.x` | Conforme | ✅ |
| T06 | Connexion Wi-Fi compte admin | Smartphone + compte admin | Auth OK + VLAN 30 + IP en `192.168.30.x` | Conforme | ✅ |
| T07 | Mot de passe erroné | Tentative de connexion avec identifiants invalides | Refus NPS + pas d'accès réseau | Conforme (connexion refusée) | ✅ |

---

## 3. Tests techniques complémentaires (recommandés)

> Ces tests ne remplacent pas les tests validés ci-dessus, ils complètent la robustesse du dossier.

| ID | Test complémentaire | But |
|:--|:--|:--|
| C01 | Coupure temporaire NPS principal (.200) avec fallback RADIUS | Vérifier continuité d'authentification |
| C02 | Vérification horodatage NTP sur SW1/AP1/R1 | Fiabilité audit et corrélation logs |
| C03 | Vérification présence backups TFTP après `wr mem` | Garantie de reprise |
| C04 | Vérification ACL étudiants vers VLAN management | Validation cloisonnement réseau |

---

## 4. Conclusion de recette

La recette fonctionnelle est **validée** sur les cas d'usage principaux : authentification réussie selon profil AD, affectation dynamique VLAN correcte, et refus d'accès en cas d'identifiants invalides.  
Le comportement observé sur téléphones et laptops Windows est cohérent avec la politique de sécurité cible.
