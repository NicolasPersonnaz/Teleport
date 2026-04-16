#  Teleport : Portail de Documentation

Bienvenue dans la documentation centrale du projet **Teleport**. Ce dépôt regroupe l'automatisation des déploiements via Teleport v18 et GitLab CI ainsi que des conseils et des commandes de maintenance. 

Pour une navigation détaillée par service, veuillez consulter les sections du Wiki ci-dessous.

---

## 📖 Sommaire du Wiki

Sélectionnez une catégorie pour accéder aux guides détaillés :

### 🏗️ Infrastructure & CI/CD
* **[Pipeline CI/CD (Configuration)](../../wiki/ci-cd)** *Automatisation complète, gestion des jobs de sécurité (Gitleaks, Semgrep) et déploiement via OIDC.*

### 🔐 Enrôlement des Ressources
* **[Enrôlement des Pages Web](../../wiki/enrollement-page-web)** *Guide pour sécuriser et exposer vos applications web via le proxy Teleport Application Access.*
* **[Enrôlement des Bases de Données](../../wiki/enrollement-bdd)** *Procédure de connexion et sécurisation des flux vers vos instances SQL/NoSQL.*

### 💻 Utilisation Quotidienne
* **[Teleport en Ligne de Commande (CLI)](../../wiki/teleport-en-ligne-de-commande)** *Aide-mémoire des commandes `tsh`, `tbot` et `tctl` pour l'administration et l'accès.*

---

## 🛡️ Architecture "Zero Trust"

Ce projet implémente les piliers suivants :
1. **Identité Éphémère** : Aucun stockage de clés SSH sur GitLab.
2. **Moindre Privilège** : Utilisation du rôle `deployer` restreint à `rsync`.
3. **Audit Total** : Traçabilité de chaque commande dans l'Audit Log Teleport.



---
*Maintenu par Nicolas Personnaz • Dernière mise à jour : Avril 2024*
