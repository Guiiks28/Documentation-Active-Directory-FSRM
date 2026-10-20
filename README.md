# Documentation Technique : Déploiement Active Directory, GPO, FSRM & Sécurité

Ce dépôt contient la procédure pas à pas de configuration d'un contrôleur de domaine Active Directory, l'organisation de l'annuaire, le partage de fichiers sécurisé avec quotas et filtrage, ainsi que le déploiement de GPO (mappage de lecteurs et fonds d'écran).

---

## 📋 Informations sur l'Environnement

* **Nom du Domaine** : `oclock.lan`
* **Serveur Active Directory** : `WS2026` (Windows Server)
* **Adresse IP Serveur** : `10.0.0.10/16` (Passerelle : `10.0.0.1`, DNS : `1.1.1.1`)
* **Volume dédié aux partages** : Disque `R:` (`Promotions Entreprise`)
* **Promotions (UO)** : Volva, Heimdall, Zinc, Licorne, Basilic

---

## 🛠️ Étape 1 : Préparation & Configuration du Serveur

### 1. Adressage IP & Nom du Serveur
* Attribution d'une adresse IP fixe (`10.0.0.10/16`).
* Renommage du serveur en `WS2026`.
* Configuration des serveurs DNS (`1.1.1.1`).

### 2. Configuration du Pare-feu (ICMP) & RDP
* **Autorisation du Ping (ICMPv4)** entrant via PowerShell :
  ```powershell
  New-NetFirewallRule -DisplayName 'ICMPv4 entrant (labo)' -Direction Inbound -Protocol ICMPv4 -IcmpType 8 -Action Allow -Profile Any
  
---

## 📁 Étape 2 : Structure Active Directory & Partages de Fichiers

### 1. Arborescence Active Directory (UO, Groupes & Utilisateurs)
* **Création de la structure d'UO** :
  * Unité d'Organisation parente : `Promotions`
  * Sous-UO par promotion : `Volva`, `Heimdall`, `Zinc`, `Licorne`, `Basilic`
* **Groupes de Sécurité Locaux au Domaine (GL)** :
  * Création des groupes de droits d'accès pour chaque promotion (ex: `GL_etudiant_basilic_RW`, `GL_formateur_promo_RW`).
* **Gestion des comptes** :
  * Création et affectation des comptes étudiants dans leurs UO et groupes de sécurité respectifs.

### 2. Intégration du Poste Client au Domaine
* **Jonction Windows 11** :
  * Modification des paramètres système pour joindre le domaine `oclock.lan`.
* **Vérification de session** :
  * Connexion avec un compte du domaine (ex: `oclock\t.s`) et validation en ligne de commande :
    ```powershell
    whoami
    ```

### 3. Allocation de Stockage & Création des Partages
1. **Initialisation du Disque** :
   * Allocation et formatage NTFS d'un second volume dédié `R:` nommé `Promotions Entreprise`.
2. **Arborescence des dossiers par promotion** sous `R:\Partages\` :
   * `Commun_PromoVolva`
   * `Commun_PromoHeimdall`
   * `Commun_PromoZinc`
   * `Commun_PromoLicorne`
   * `Commun_PromoBasilic`

### 4. Sécurisation NTFS & Partage réseau
* **Suppression de l'héritage NTFS** sur l'ensemble des sous-dossiers de promotions.
* **Attribution des autorisations NTFS** :
  * `GL_etudiant_<promo>_RW` : Modification, Lecture & exécution, Affichage du contenu, Lecture, Écriture.
  * `GL_formateur_promo_RW` : Modification, Lecture & exécution, Affichage du contenu, Lecture, Écriture.
  * `Administrateurs` : Contrôle total.
* **Activation du Partage Avancé** :
  * Partage réseau sous la forme `\\WS2026\Commun_Promo<Nom>$` (ou partage ouvert selon la visibilité souhaitée).
  
---

## ⚙️ Étape 3 : Stratégies de Groupe (GPO) & Restrictions

### 1. Mappage Automatique des Lecteurs Réseau
* **Création de la GPO** : 
  * Création d'un objet de stratégie de groupe nommé `GP_Mappage_Promo`.
* **Configuration des préférences** :
  * Chemin dans l'éditeur : *Configuration utilisateur > Préférences > Paramètres Windows > Mappages de lecteurs*.
  * Association automatique de chaque lettre de lecteur (A:, B:, E:, F:, G:) au partage réseau de la promotion correspondante (ex: `\\WS2026\Commun_PromoBasilic`, etc.).

### 2. Déploiement des Fonds d'Écran Personnalisés
* **Préparation** :
  * Stockage des images de fond d'écran par promotion sur un partage réseau accessible.
* **Création des GPO dédiées** :
  * Création de 5 GPO distinctes : `GP_FondEcran_Volva`, `GP_FondEcran_Heimdall`, `GP_FondEcran_Zinc`, `GP_FondEcran_Licorne` et `GP_FondEcran_Basilic`.
* **Liaison** :
  * Liaison de chaque GPO directement sur l'Unité d'Organisation (UO) de la promotion correspondante.
  * Activation du paramètre *Fond d'écran du Bureau* pointant vers l'image attitrée.

### 3. Restrictions Horaires de Connexion
* **Paramétrage AD** :
  * Modification directe dans les propriétés des comptes utilisateurs (ou via un lot d'utilisateurs) dans Active Directory -> *Heures de connexion*.
  * Blocage des plages horaires non autorisées (par exemple, restriction de la tranche 17h00-08h00 pour les promotions Licorne et Basilic).
   
---

## 🛡️ Étape 4 : Gestion de l'Espace Stockage (FSRM)

Installation et configuration du rôle **Gestionnaire de ressources du serveur de fichiers (FSRM)** pour encadrer l'utilisation du stockage.

### 1. Quotas de Stockage
* **Application d'un modèle** :
  * Création/application d'un quota **conditionnel de 30 Go** sur le dossier parent `R:\Partages`.
  * Activation de l'option *Appliquer automatiquement le modèle et créer des quotas sur les sous-dossiers existants et nouveaux* (30 Go max par promotion).
* **Notifications** :
  * Configuration des alertes d'avertissement dans le journal d'événements à **85%** et **100%** de l'utilisation du quota.

### 2. Filtrage de Fichiers 
* **Création d'un modèle de filtre** :
  * Ciblage des fichiers non autorisés (ex: fichiers multimédias/divx, archives, exécutables).
* **Type de filtrage** :
  * Application d'un **filtrage actif** sur le dossier `R:\Partages` pour empêcher purement et simplement les utilisateurs d'enregistrer des fichiers non autorisés.

---

## ✅ Validation & Tests

* **Test Étudiant** : 
  * Connexion sur un poste client avec un compte de la promotion (ex: `Licorne`).
  * Le fond d'écran de la promotion s'applique automatiquement.
  * Seul le lecteur réseau correspondant à sa promotion est monté et accessible.
  * L'accès aux dossiers des autres promotions est bloqué par les permissions NTFS.
* **Test Formateur** : 
  * Connexion avec un compte enseignant pour vérifier l'accès global aux différents partages des promotions.
