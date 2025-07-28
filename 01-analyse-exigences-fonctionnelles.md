# Analyse des Exigences Fonctionnelles - Dossier de Pièces Justificatives

## 1. Contexte et Objectifs

### Problématique
- Obsolescence du DNC (Dossier Numérique Client)
- Surcoûts liés à l'évolution du système existant
- Besoin de réurbanisation des produits de Dématérialisation

### Objectifs du Socle Omnicanal
- Centraliser les pièces justificatives via IHM Client et Collaborateur
- Affichage contextuel des pièces justificatives
- Récolte de documents depuis différentes sources
- Réutilisation des documents déjà transmis

## 2. Acteurs du Système

### Acteurs Principaux
- **Client** : Utilisateur final déposant et gérant ses pièces justificatives
- **Collaborateur/Conseiller** : Personnel bancaire gérant les dossiers clients
- **Système GED** : Système de Gestion Électronique de Documents (intégration)

### Besoins Métier
- **Conseiller/Client** : Solution centralisée, simplifiée et contextualisée
- **Conseiller commercial** : Parcours facilité sans rupture pour réception et archivage

## 3. Exigences Fonctionnelles Détaillées

### 3.1 Fonctionnalités Communes (Client & Collaborateur)

#### Visualisation
- **RF-001** : Afficher la liste des pièces justificatives d'un dossier
- **RF-002** : Visualiser un document (aperçu, téléchargement)
- **RF-003** : Visualiser les documents manquants par étape avec compteur
- **RF-004** : Visualiser les documents d'une étape avec statut (validé, en attente, rejeté)

#### Gestion des Documents
- **RF-005** : Ajouter un document au dossier
- **RF-006** : Supprimer un document du dossier
- **RF-007** : Remplacer un document existant

### 3.2 Fonctionnalités Spécifiques Collaborateur

#### Mise en GED
- **RF-008** : Mettre en GED un document individuel
- **RF-009** : Mettre en GED une liste de pièces justificatives d'un dossier
- **RF-010** : Mettre en GED tous les documents d'une étape

#### Gestion Avancée
- **RF-011** : Valider/Rejeter des documents
- **RF-012** : Ajouter des commentaires sur les documents
- **RF-013** : Historique des actions sur les documents

### 3.3 Services Transversaux

#### Contextualisation
- **RF-014** : Proposer une liste de pièces contextualisée selon le parcours
- **RF-015** : Adapter l'affichage selon le profil utilisateur (Client/Collaborateur)

#### Réutilisation
- **RF-016** : Reproposer une pièce déjà transmise depuis la GED
- **RF-017** : Reproposer une pièce depuis l'Espace doc en attente
- **RF-018** : Reproposer une pièce depuis un autre dossier de PJ

## 4. Exigences Non Fonctionnelles

### Performance
- **RNF-001** : Support de 500+ utilisateurs simultanés
- **RNF-002** : Temps de réponse < 2 secondes pour l'affichage des listes
- **RNF-003** : Support de documents jusqu'à 50MB
- **RNF-004** : Temps de téléchargement < 5 secondes pour documents 10MB

### Disponibilité
- **RNF-005** : Disponibilité 99.9% (haute disponibilité requise)
- **RNF-006** : Temps de récupération < 15 minutes en cas de panne

### Sécurité
- **RNF-007** : Authentification sécurisée des utilisateurs
- **RNF-008** : Chiffrement des documents en transit et au repos
- **RNF-009** : Traçabilité complète des actions sur les documents
- **RNF-010** : Contrôle d'accès basé sur les rôles (RBAC)

### Intégration
- **RNF-011** : Intégration avec système GED existant
- **RNF-012** : API REST pour intégration avec parcours bancaires
- **RNF-013** : Interface responsive (web, mobile)

## 5. Contraintes Techniques

### Contraintes d'Architecture
- Architecture microservices moderne
- API REST
- Base de données dédiée
- Interfaces web responsive

### Contraintes d'Intégration
- Intégration minimale (GED + authentification simple)
- Compatibilité avec systèmes existants

## 6. Cas d'Usage Prioritaires

### Cas d'Usage Client
1. **Déposer des pièces justificatives** dans un parcours bancaire
2. **Consulter l'état d'avancement** de son dossier
3. **Remplacer un document rejeté**

### Cas d'Usage Collaborateur
1. **Valider un dossier complet** de pièces justificatives
2. **Archiver en GED** les documents validés
3. **Suivre l'avancement** de multiples dossiers clients

## 7. Règles Métier

### Gestion des États
- **RM-001** : Un document peut avoir les états : En attente, Validé, Rejeté, Archivé
- **RM-002** : Un dossier est complet quand toutes les PJ obligatoires sont validées
- **RM-003** : Seuls les documents validés peuvent être mis en GED

### Droits d'Accès
- **RM-004** : Un client ne peut accéder qu'à ses propres dossiers
- **RM-005** : Un collaborateur peut accéder aux dossiers de ses clients assignés
- **RM-006** : Seuls les collaborateurs peuvent mettre en GED

### Traçabilité
- **RM-007** : Toute action sur un document doit être tracée (qui, quand, quoi)
- **RM-008** : L'historique des versions de documents doit être conservé