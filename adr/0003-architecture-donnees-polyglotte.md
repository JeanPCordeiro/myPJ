# ADR-0003: Architecture de Données Polyglotte

## Statut

Accepté

## Contexte

Le système DPJ doit gérer différents types de données avec des besoins variés :
- Données transactionnelles (utilisateurs, dossiers, relations)
- Documents et métadonnées (fichiers, contenu non-structuré)
- Cache haute performance (sessions, données temporaires)
- Recherche et indexation (recherche full-text)

Une approche mono-base de données ne peut pas répondre efficacement à tous ces besoins.

## Options Considérées

### Option 1: PostgreSQL uniquement
- **Avantages** : Simplicité, ACID, JSON support
- **Inconvénients** : Performance limitée pour documents volumineux

### Option 2: MongoDB uniquement
- **Avantages** : Flexibilité schéma, performance documents
- **Inconvénients** : Pas de transactions ACID complètes

### Option 3: Architecture Polyglotte
- **Avantages** : Optimisation par type de données
- **Inconvénients** : Complexité opérationnelle

## Décision

Nous adoptons une **architecture de données polyglotte** avec :

### PostgreSQL (Données Transactionnelles)
- **Usage** : Utilisateurs, dossiers, relations, audit
- **Justification** : ACID, intégrité référentielle, requêtes complexes
- **Schémas** : `users`, `dossiers`, `documents_metadata`, `audit_logs`

### MongoDB (Documents et Métadonnées)
- **Usage** : Contenu documents, métadonnées flexibles, historique
- **Justification** : Schéma flexible, performance documents volumineux
- **Collections** : `document_content`, `document_versions`, `search_index`

### Redis (Cache et Sessions)
- **Usage** : Cache applicatif, sessions utilisateur, données temporaires
- **Justification** : Performance mémoire, TTL automatique
- **Structures** : Cache L1, sessions JWT, rate limiting

### MinIO (Stockage Objets)
- **Usage** : Fichiers binaires, documents scannés
- **Justification** : Compatibilité S3, scalabilité, réplication
- **Organisation** : Buckets par type de document

## Conséquences

### Positives
- **Performance** : Chaque base optimisée pour son usage
- **Scalabilité** : Scaling indépendant par type de données
- **Flexibilité** : Adaptation aux besoins spécifiques
- **Résilience** : Isolation des pannes par type de données

### Négatives
- **Complexité** : Gestion de multiples systèmes de données
- **Cohérence** : Gestion de la cohérence inter-bases
- **Opérations** : Monitoring et maintenance multiples
- **Développement** : Expertise requise sur plusieurs technologies

### Risques
- Incohérence des données entre systèmes
- Complexité des transactions distribuées
- Surcharge opérationnelle

### Mitigations
- **Saga Pattern** pour les transactions distribuées
- **Event Sourcing** pour la cohérence éventuelle
- **Monitoring unifié** avec Prometheus
- **Backup coordonné** entre tous les systèmes
- **Documentation** des patterns d'accès aux données