# Architecture Technique Globale - Dossier de Pièces Justificatives

## 1. Vue d'Ensemble de l'Architecture

### Principes Architecturaux
- **Architecture Microservices** : Services découplés et indépendants
- **API First** : Exposition via API REST standardisées
- **Scalabilité Horizontale** : Support de la montée en charge
- **Haute Disponibilité** : Redondance et résilience
- **Sécurité by Design** : Sécurité intégrée à tous les niveaux

### Stack Technologique Recommandée
- **Backend** : Java Spring Boot / Node.js Express
- **Base de Données** : PostgreSQL (principale) + MongoDB (documents)
- **Cache** : Redis
- **Message Queue** : Apache Kafka / RabbitMQ
- **Storage** : MinIO (S3-compatible) / Azure Blob Storage
- **Frontend** : React.js / Vue.js
- **API Gateway** : Kong / AWS API Gateway
- **Orchestration** : Kubernetes
- **Monitoring** : Prometheus + Grafana

## 2. Architecture des Microservices

### 2.1 Services Métier

#### Service Dossier (dossier-service)
**Responsabilités :**
- Gestion du cycle de vie des dossiers de PJ
- Orchestration des étapes de validation
- Gestion des états et transitions

**Technologies :**
- Java Spring Boot
- PostgreSQL
- Redis (cache)

#### Service Document (document-service)
**Responsabilités :**
- Gestion des métadonnées des documents
- Validation des formats et tailles
- Gestion des versions

**Technologies :**
- Java Spring Boot
- PostgreSQL
- MinIO (stockage fichiers)

#### Service Stockage (storage-service)
**Responsabilités :**
- Stockage physique des fichiers
- Chiffrement des documents
- Gestion des accès aux fichiers

**Technologies :**
- Node.js Express
- MinIO / Azure Blob Storage
- Chiffrement AES-256

#### Service GED (ged-integration-service)
**Responsabilités :**
- Intégration avec le système GED existant
- Archivage des documents validés
- Synchronisation des métadonnées

**Technologies :**
- Java Spring Boot
- Connecteurs GED spécifiques
- Message Queue pour traitement asynchrone

#### Service Notification (notification-service)
**Responsabilités :**
- Notifications temps réel
- Alertes et rappels
- Communication multi-canal

**Technologies :**
- Node.js Express
- WebSocket
- Email/SMS providers

### 2.2 Services Transversaux

#### Service Authentification (auth-service)
**Responsabilités :**
- Authentification des utilisateurs
- Gestion des tokens JWT
- Contrôle d'accès (RBAC)

**Technologies :**
- Java Spring Security
- PostgreSQL
- Redis (sessions)

#### Service Audit (audit-service)
**Responsabilités :**
- Traçabilité des actions
- Logs d'audit
- Conformité réglementaire

**Technologies :**
- Node.js Express
- MongoDB
- Elasticsearch (recherche logs)

#### Service Configuration (config-service)
**Responsabilités :**
- Configuration centralisée
- Gestion des environnements
- Feature flags

**Technologies :**
- Spring Cloud Config
- Git repository
- Redis

## 3. Architecture des Données

### 3.1 Base de Données Principale (PostgreSQL)

#### Tables Principales
```sql
-- Dossiers de pièces justificatives
dossiers (
    id UUID PRIMARY KEY,
    client_id VARCHAR(50) NOT NULL,
    type_parcours VARCHAR(100) NOT NULL,
    statut VARCHAR(20) NOT NULL,
    etape_courante VARCHAR(50),
    date_creation TIMESTAMP,
    date_modification TIMESTAMP,
    collaborateur_id VARCHAR(50)
);

-- Documents
documents (
    id UUID PRIMARY KEY,
    dossier_id UUID REFERENCES dossiers(id),
    nom_fichier VARCHAR(255) NOT NULL,
    type_document VARCHAR(100) NOT NULL,
    taille_fichier BIGINT,
    chemin_stockage VARCHAR(500),
    statut VARCHAR(20) NOT NULL,
    date_depot TIMESTAMP,
    date_validation TIMESTAMP,
    validateur_id VARCHAR(50)
);

-- Étapes de parcours
etapes_parcours (
    id UUID PRIMARY KEY,
    dossier_id UUID REFERENCES dossiers(id),
    nom_etape VARCHAR(100) NOT NULL,
    ordre_etape INTEGER,
    statut VARCHAR(20) NOT NULL,
    documents_obligatoires JSONB,
    documents_optionnels JSONB
);
```

### 3.2 Base de Données Documents (MongoDB)

#### Collections
```javascript
// Métadonnées étendues des documents
documents_metadata: {
    _id: ObjectId,
    document_id: UUID,
    metadata: {
        author: String,
        creation_date: Date,
        modification_date: Date,
        tags: [String],
        custom_fields: Object
    },
    versions: [{
        version: Number,
        file_path: String,
        checksum: String,
        date: Date
    }]
}

// Historique des actions
audit_trail: {
    _id: ObjectId,
    entity_type: String, // 'dossier', 'document'
    entity_id: UUID,
    action: String,
    user_id: String,
    timestamp: Date,
    details: Object
}
```

## 4. Architecture de Communication

### 4.1 Communication Synchrone (API REST)

#### API Gateway
- Point d'entrée unique pour toutes les requêtes
- Authentification et autorisation
- Rate limiting et throttling
- Load balancing
- Monitoring et métriques

#### Patterns de Communication
- **Request/Response** : Opérations CRUD standard
- **Circuit Breaker** : Résilience aux pannes
- **Retry Pattern** : Gestion des erreurs temporaires

### 4.2 Communication Asynchrone (Message Queue)

#### Topics Kafka
```
- dossier.created
- dossier.updated
- dossier.completed
- document.uploaded
- document.validated
- document.archived
- notification.send
```

#### Avantages
- Découplage des services
- Traitement asynchrone des tâches lourdes
- Résilience et reprise sur erreur
- Scalabilité indépendante

## 5. Architecture de Sécurité

### 5.1 Authentification et Autorisation

#### JWT (JSON Web Tokens)
```json
{
  "sub": "user123",
  "role": "collaborateur",
  "permissions": ["read:dossier", "write:document", "archive:ged"],
  "exp": 1640995200,
  "iat": 1640908800
}
```

#### Contrôle d'Accès (RBAC)
- **Client** : Accès à ses propres dossiers uniquement
- **Collaborateur** : Accès aux dossiers assignés
- **Administrateur** : Accès complet avec audit

### 5.2 Sécurité des Données

#### Chiffrement
- **En transit** : TLS 1.3 pour toutes les communications
- **Au repos** : AES-256 pour les documents stockés
- **Base de données** : Chiffrement transparent (TDE)

#### Validation et Sanitisation
- Validation des formats de fichiers
- Scan antivirus des documents uploadés
- Sanitisation des inputs utilisateur

## 6. Architecture de Performance

### 6.1 Stratégies de Cache

#### Cache Multi-Niveaux
```
Browser Cache (1h)
  ↓
CDN Cache (24h)
  ↓
API Gateway Cache (15min)
  ↓
Service Cache Redis (5min)
  ↓
Database
```

#### Données Cachées
- Listes de documents par dossier
- Métadonnées des documents
- Configuration des parcours
- Informations utilisateur

### 6.2 Optimisations Base de Données

#### Index Stratégiques
```sql
-- Index pour recherche rapide
CREATE INDEX idx_dossiers_client_statut ON dossiers(client_id, statut);
CREATE INDEX idx_documents_dossier_type ON documents(dossier_id, type_document);
CREATE INDEX idx_audit_entity_date ON audit_trail(entity_id, timestamp);
```

#### Partitioning
- Partitioning par date pour les tables d'audit
- Sharding par client_id pour les gros volumes

## 7. Architecture de Déploiement

### 7.1 Containerisation (Docker)

#### Structure des Containers
```dockerfile
# Exemple pour document-service
FROM openjdk:17-jre-slim
COPY target/document-service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 7.2 Orchestration (Kubernetes)

#### Déploiement par Environnement
- **Development** : 1 replica par service
- **Staging** : 2 replicas par service
- **Production** : 3+ replicas avec auto-scaling

#### Configuration Kubernetes
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: document-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: document-service
  template:
    spec:
      containers:
      - name: document-service
        image: dpj/document-service:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

## 8. Monitoring et Observabilité

### 8.1 Métriques Techniques
- **Performance** : Temps de réponse, throughput
- **Disponibilité** : Uptime, health checks
- **Ressources** : CPU, mémoire, stockage
- **Erreurs** : Taux d'erreur, exceptions

### 8.2 Métriques Métier
- **Utilisation** : Nombre de dossiers créés/jour
- **Performance** : Temps moyen de validation
- **Qualité** : Taux de documents rejetés
- **Satisfaction** : Temps de traitement par étape

## 9. Stratégie de Scalabilité

### 9.1 Scalabilité Horizontale

#### Auto-scaling Kubernetes
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: document-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: document-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 9.2 Optimisations pour 500+ Utilisateurs

#### Connection Pooling
- Pool de connexions DB optimisé
- Connection pooling Redis
- HTTP/2 pour les APIs

#### Traitement Asynchrone
- Upload de fichiers en arrière-plan
- Validation asynchrone des documents
- Archivage GED différé

## 10. Plan de Continuité

### 10.1 Haute Disponibilité

#### Multi-AZ Deployment
- Réplication des services sur plusieurs zones
- Load balancing géographique
- Failover automatique

#### Backup et Recovery
- Backup automatique des bases de données (RPO: 1h)
- Réplication des fichiers (RTO: 15min)
- Tests de recovery mensuels

### 10.2 Disaster Recovery

#### Stratégie 3-2-1
- 3 copies des données
- 2 supports différents
- 1 copie hors site

#### Plan de Reprise
1. **Détection** : Monitoring automatique (< 2min)
2. **Escalade** : Notification équipe (< 5min)
3. **Recovery** : Basculement automatique (< 15min)
4. **Validation** : Tests fonctionnels (< 30min)