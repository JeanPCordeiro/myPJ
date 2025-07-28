# Diagrammes d'Architecture

## 1. Vue d'Ensemble du Système

### 1.1 Architecture Fonctionnelle Globale

```mermaid
graph TB
    subgraph "Utilisateurs"
        Client[👤 Client]
        Collaborateur[👨‍💼 Collaborateur]
        Admin[👨‍💻 Administrateur]
    end
    
    subgraph "Couche Présentation"
        WebClient[🌐 Interface Client Web]
        WebCollab[🌐 Interface Collaborateur Web]
        MobileApp[📱 Application Mobile]
    end
    
    subgraph "Couche API Gateway"
        APIGateway[🚪 API Gateway]
        LoadBalancer[⚖️ Load Balancer]
        WAF[🛡️ Web Application Firewall]
    end
    
    subgraph "Couche Services Métier"
        DossierService[📁 Service Dossier]
        DocumentService[📄 Service Document]
        AuthService[🔐 Service Authentification]
        GEDService[🗄️ Service GED]
        NotificationService[📧 Service Notification]
    end
    
    subgraph "Couche Données"
        PostgreSQL[(🐘 PostgreSQL)]
        MongoDB[(🍃 MongoDB)]
        Redis[(🔴 Redis)]
        MinIO[(📦 MinIO)]
    end
    
    subgraph "Systèmes Externes"
        GED[🗃️ Système GED Existant]
        LDAP[👥 Annuaire LDAP]
        Email[📮 Serveur Email]
    end
    
    Client --> WebClient
    Collaborateur --> WebCollab
    Admin --> WebCollab
    Client --> MobileApp
    
    WebClient --> LoadBalancer
    WebCollab --> LoadBalancer
    MobileApp --> LoadBalancer
    
    LoadBalancer --> WAF
    WAF --> APIGateway
    
    APIGateway --> DossierService
    APIGateway --> DocumentService
    APIGateway --> AuthService
    APIGateway --> GEDService
    APIGateway --> NotificationService
    
    DossierService --> PostgreSQL
    DocumentService --> PostgreSQL
    DocumentService --> MinIO
    AuthService --> Redis
    GEDService --> MongoDB
    
    GEDService --> GED
    AuthService --> LDAP
    NotificationService --> Email
    
    style Client fill:#e1f5fe
    style Collaborateur fill:#e8f5e8
    style Admin fill:#fff3e0
    style APIGateway fill:#f3e5f5
    style PostgreSQL fill:#e3f2fd
    style MongoDB fill:#e8f5e8
    style Redis fill:#ffebee
    style MinIO fill:#fff8e1
```

### 1.2 Architecture Technique Détaillée

```mermaid
graph TB
    subgraph "Internet"
        Users[👥 Utilisateurs]
        CDN[☁️ CloudFront CDN]
    end
    
    subgraph "DMZ - Zone Démilitarisée"
        WAF[🛡️ Web Application Firewall]
        ALB[⚖️ Application Load Balancer]
        NLB[⚖️ Network Load Balancer]
    end
    
    subgraph "Kubernetes Cluster"
        subgraph "Ingress Layer"
            IngressController[🚪 NGINX Ingress Controller]
            CertManager[🔒 Cert Manager]
        end
        
        subgraph "Frontend Namespace"
            ClientPods[🌐 Client App Pods]
            CollabPods[🌐 Collaborateur App Pods]
        end
        
        subgraph "Backend Namespace"
            APIGatewayPods[🚪 API Gateway Pods]
            DossierPods[📁 Dossier Service Pods]
            DocumentPods[📄 Document Service Pods]
            AuthPods[🔐 Auth Service Pods]
            GEDPods[🗄️ GED Service Pods]
        end
        
        subgraph "Data Namespace"
            PostgreSQLCluster[(🐘 PostgreSQL Cluster)]
            MongoDBCluster[(🍃 MongoDB Cluster)]
            RedisCluster[(🔴 Redis Cluster)]
            MinIOCluster[(📦 MinIO Cluster)]
        end
        
        subgraph "Infrastructure Namespace"
            KafkaCluster[📨 Kafka Cluster]
            PrometheusStack[📊 Prometheus Stack]
            JaegerTracing[🔍 Jaeger Tracing]
            VeleroBackup[💾 Velero Backup]
        end
    end
    
    subgraph "External Systems"
        GEDSystem[🗃️ Système GED]
        LDAPServer[👥 Serveur LDAP]
        EmailServer[📮 Serveur Email]
        S3Backup[☁️ S3 Backup]
    end
    
    Users --> CDN
    CDN --> WAF
    WAF --> ALB
    ALB --> IngressController
    IngressController --> ClientPods
    IngressController --> CollabPods
    IngressController --> APIGatewayPods
    
    APIGatewayPods --> DossierPods
    APIGatewayPods --> DocumentPods
    APIGatewayPods --> AuthPods
    APIGatewayPods --> GEDPods
    
    DossierPods --> PostgreSQLCluster
    DocumentPods --> PostgreSQLCluster
    DocumentPods --> MinIOCluster
    AuthPods --> RedisCluster
    GEDPods --> MongoDBCluster
    
    GEDPods --> GEDSystem
    AuthPods --> LDAPServer
    
    VeleroBackup --> S3Backup
    
    style Users fill:#e1f5fe
    style CDN fill:#e8f5e8
    style WAF fill:#ffebee
    style KafkaCluster fill:#fff3e0
    style PrometheusStack fill:#f3e5f5
```

## 2. Architecture des Microservices

### 2.1 Diagramme de Services

```mermaid
graph TB
    subgraph "API Gateway Layer"
        Gateway[🚪 API Gateway<br/>- Routing<br/>- Authentication<br/>- Rate Limiting<br/>- Load Balancing]
    end
    
    subgraph "Business Services Layer"
        DossierSvc[📁 Dossier Service<br/>- Gestion dossiers<br/>- Workflow étapes<br/>- Validation règles<br/>Port: 8081]
        
        DocumentSvc[📄 Document Service<br/>- Upload documents<br/>- Validation formats<br/>- Métadonnées<br/>Port: 8082]
        
        AuthSvc[🔐 Auth Service<br/>- JWT tokens<br/>- RBAC<br/>- Sessions<br/>Port: 8083]
        
        GEDSvc[🗄️ GED Service<br/>- Archivage<br/>- Recherche<br/>- Réutilisation<br/>Port: 8084]
        
        NotifSvc[📧 Notification Service<br/>- Email/SMS<br/>- Push notifications<br/>- Templates<br/>Port: 8085]
        
        StorageSvc[📦 Storage Service<br/>- Gestion fichiers<br/>- Chiffrement<br/>- Thumbnails<br/>Port: 8086]
    end
    
    subgraph "Infrastructure Services Layer"
        ConfigSvc[⚙️ Config Service<br/>- Configuration centralisée<br/>- Feature flags<br/>Port: 8087]
        
        AuditSvc[📋 Audit Service<br/>- Logs d'audit<br/>- Traçabilité<br/>- Compliance<br/>Port: 8088]
    end
    
    subgraph "Data Layer"
        PostgreSQL[(🐘 PostgreSQL<br/>- Données transactionnelles<br/>- Dossiers & Documents<br/>Port: 5432)]
        
        MongoDB[(🍃 MongoDB<br/>- Métadonnées étendues<br/>- Logs d'audit<br/>Port: 27017)]
        
        Redis[(🔴 Redis<br/>- Cache<br/>- Sessions<br/>Port: 6379)]
        
        MinIO[(📦 MinIO<br/>- Stockage fichiers<br/>- Objets S3<br/>Port: 9000)]
    end
    
    subgraph "Message Queue"
        Kafka[📨 Apache Kafka<br/>- Event streaming<br/>- Async processing<br/>Port: 9092]
    end
    
    Gateway --> DossierSvc
    Gateway --> DocumentSvc
    Gateway --> AuthSvc
    Gateway --> GEDSvc
    Gateway --> NotifSvc
    Gateway --> StorageSvc
    
    DossierSvc --> PostgreSQL
    DossierSvc --> Kafka
    
    DocumentSvc --> PostgreSQL
    DocumentSvc --> MinIO
    DocumentSvc --> Kafka
    
    AuthSvc --> Redis
    AuthSvc --> PostgreSQL
    
    GEDSvc --> MongoDB
    GEDSvc --> Kafka
    
    StorageSvc --> MinIO
    StorageSvc --> Redis
    
    NotifSvc --> Kafka
    
    AuditSvc --> MongoDB
    AuditSvc --> Kafka
    
    ConfigSvc --> Redis
    
    style Gateway fill:#f3e5f5
    style DossierSvc fill:#e8f5e8
    style DocumentSvc fill:#e1f5fe
    style AuthSvc fill:#ffebee
    style GEDSvc fill:#fff3e0
    style NotifSvc fill:#f1f8e9
    style StorageSvc fill:#fce4ec
```

### 2.2 Communication entre Services

```mermaid
sequenceDiagram
    participant Client as 👤 Client Web
    participant Gateway as 🚪 API Gateway
    participant Auth as 🔐 Auth Service
    participant Document as 📄 Document Service
    participant Storage as 📦 Storage Service
    participant Kafka as 📨 Kafka
    participant GED as 🗄️ GED Service
    participant Notification as 📧 Notification Service
    
    Client->>Gateway: POST /api/v1/documents (upload)
    Gateway->>Auth: Validate JWT token
    Auth-->>Gateway: Token valid + user context
    
    Gateway->>Document: POST /documents (with user context)
    Document->>Storage: Store file + metadata
    Storage-->>Document: Storage result (path, checksum)
    
    Document->>Document: Save document metadata to DB
    Document-->>Gateway: Document created (201)
    Gateway-->>Client: Upload successful
    
    Document->>Kafka: Publish DocumentUploadedEvent
    
    Note over Kafka: Async processing
    
    Kafka->>GED: DocumentUploadedEvent
    GED->>GED: Process for potential archiving
    
    Kafka->>Notification: DocumentUploadedEvent
    Notification->>Notification: Send confirmation email
    
    Note over Client,Notification: Document validation flow
    
    Client->>Gateway: POST /api/v1/documents/{id}/validate
    Gateway->>Document: Validate document
    Document->>Document: Update status to VALIDATED
    Document->>Kafka: Publish DocumentValidatedEvent
    
    Kafka->>GED: DocumentValidatedEvent
    GED->>GED: Archive to GED system
    GED->>Kafka: Publish DocumentArchivedEvent
    
    Kafka->>Notification: DocumentArchivedEvent
    Notification->>Client: Send archive confirmation
```

## 3. Architecture des Données

### 3.1 Modèle de Données Relationnel

```mermaid
erDiagram
    DOSSIERS {
        uuid id PK
        varchar numero_dossier UK
        varchar client_id
        varchar type_parcours
        varchar statut
        varchar etape_courante
        integer priorite
        timestamp date_creation
        timestamp date_modification
        timestamp date_echeance
        varchar collaborateur_id
        text commentaire_interne
        jsonb metadata
        integer version
        varchar created_by
        varchar updated_by
    }
    
    ETAPES_PARCOURS {
        uuid id PK
        uuid dossier_id FK
        varchar nom_etape
        varchar code_etape
        integer ordre_etape
        varchar statut
        timestamp date_debut
        timestamp date_fin
        integer duree_estimee_heures
        jsonb documents_obligatoires
        jsonb documents_optionnels
        jsonb regles_validation
        timestamp created_at
        timestamp updated_at
    }
    
    DOCUMENTS {
        uuid id PK
        uuid dossier_id FK
        uuid etape_id FK
        varchar nom_fichier
        varchar nom_original
        varchar type_document
        varchar categorie_document
        varchar mime_type
        bigint taille_fichier
        varchar checksum_sha256
        varchar chemin_stockage
        varchar statut
        boolean est_obligatoire
        timestamp date_depot
        timestamp date_validation
        timestamp date_expiration
        varchar deposant_id
        varchar validateur_id
        text motif_rejet
        text commentaire_validation
        integer numero_version
        uuid document_parent_id FK
        jsonb metadata
        varchar[] tags
        varchar ged_id
        timestamp date_archivage
        varchar chemin_ged
        varchar source_ged_id
        timestamp created_at
        timestamp updated_at
    }
    
    TYPES_DOCUMENTS {
        uuid id PK
        varchar code_type UK
        varchar libelle
        text description
        varchar[] formats_autorises
        integer taille_max_mb
        boolean est_actif
        jsonb regles_validation
        jsonb template_metadata
        timestamp created_at
    }
    
    UTILISATEURS {
        varchar id PK
        varchar email UK
        varchar nom
        varchar prenom
        varchar role
        varchar statut
        timestamp derniere_connexion
        jsonb preferences
        timestamp created_at
        timestamp updated_at
    }
    
    SESSIONS {
        uuid id PK
        varchar utilisateur_id FK
        varchar token_hash
        timestamp date_creation
        timestamp date_expiration
        inet ip_address
        text user_agent
        boolean est_actif
    }
    
    AUDIT_LOGS {
        uuid id PK
        varchar entity_type
        uuid entity_id
        varchar action
        varchar utilisateur_id
        timestamp timestamp
        inet ip_address
        text user_agent
        jsonb details
        jsonb ancien_etat
        jsonb nouvel_etat
    }
    
    DOSSIERS ||--o{ ETAPES_PARCOURS : "contient"
    DOSSIERS ||--o{ DOCUMENTS : "contient"
    ETAPES_PARCOURS ||--o{ DOCUMENTS : "associé à"
    DOCUMENTS ||--o| DOCUMENTS : "version de"
    DOCUMENTS }o--|| TYPES_DOCUMENTS : "est de type"
    UTILISATEURS ||--o{ SESSIONS : "a des"
    UTILISATEURS ||--o{ AUDIT_LOGS : "effectue"
```

### 3.2 Architecture de Stockage Multi-Tiers

```mermaid
graph TB
    subgraph "Tier 1 - Cache Haute Performance"
        CDN[☁️ CloudFront CDN<br/>- Static assets<br/>- Thumbnails<br/>- TTL: 24h]
        
        RedisCache[🔴 Redis Cache<br/>- Documents metadata<br/>- User sessions<br/>- TTL: 15min]
    end
    
    subgraph "Tier 2 - Stockage Principal"
        MinIOCluster[📦 MinIO Cluster<br/>- Documents actifs<br/>- Réplication 3x<br/>- Chiffrement AES-256]
        
        PostgreSQLCluster[(🐘 PostgreSQL Cluster<br/>- Métadonnées<br/>- Transactions<br/>- Backup quotidien)]
        
        MongoDBCluster[(🍃 MongoDB Cluster<br/>- Métadonnées étendues<br/>- Logs d'audit<br/>- Sharding)]
    end
    
    subgraph "Tier 3 - Archivage Long Terme"
        S3Glacier[❄️ AWS S3 Glacier<br/>- Documents archivés<br/>- Rétention 10 ans<br/>- Coût optimisé]
        
        GEDSystem[🗃️ Système GED<br/>- Documents validés<br/>- Recherche avancée<br/>- Workflow métier]
    end
    
    subgraph "Tier 4 - Backup et DR"
        S3Backup[☁️ S3 Backup<br/>- Sauvegardes<br/>- Multi-région<br/>- Versioning]
        
        VeleroBackup[💾 Velero Backup<br/>- Kubernetes resources<br/>- PV snapshots<br/>- Restore automation]
    end
    
    Users[👥 Utilisateurs] --> CDN
    CDN --> RedisCache
    RedisCache --> MinIOCluster
    
    Applications[🔧 Applications] --> PostgreSQLCluster
    Applications --> MongoDBCluster
    Applications --> MinIOCluster
    
    MinIOCluster --> S3Glacier
    MinIOCluster --> GEDSystem
    
    PostgreSQLCluster --> S3Backup
    MongoDBCluster --> S3Backup
    MinIOCluster --> VeleroBackup
    
    style CDN fill:#e8f5e8
    style RedisCache fill:#ffebee
    style MinIOCluster fill:#fff8e1
    style PostgreSQLCluster fill:#e3f2fd
    style MongoDBCluster fill:#e8f5e8
    style S3Glacier fill:#f3e5f5
    style GEDSystem fill:#fff3e0
```

## 4. Architecture de Déploiement

### 4.1 Infrastructure Kubernetes

```mermaid
graph TB
    subgraph "AWS Cloud"
        subgraph "VPC - 10.0.0.0/16"
            subgraph "Public Subnets"
                PublicAZ1[Public Subnet AZ-1<br/>10.0.1.0/24]
                PublicAZ2[Public Subnet AZ-2<br/>10.0.2.0/24]
                PublicAZ3[Public Subnet AZ-3<br/>10.0.3.0/24]
            end
            
            subgraph "Private Subnets"
                PrivateAZ1[Private Subnet AZ-1<br/>10.0.11.0/24]
                PrivateAZ2[Private Subnet AZ-2<br/>10.0.12.0/24]
                PrivateAZ3[Private Subnet AZ-3<br/>10.0.13.0/24]
            end
            
            subgraph "EKS Control Plane"
                EKSMaster[🎛️ EKS Master Nodes<br/>- Multi-AZ<br/>- Managed by AWS<br/>- API Server<br/>- etcd]
            end
            
            subgraph "Worker Nodes AZ-1"
                WorkerAZ1[🖥️ Worker Nodes<br/>- m5.xlarge<br/>- 2 On-Demand<br/>- 2 Spot Instances]
            end
            
            subgraph "Worker Nodes AZ-2"
                WorkerAZ2[🖥️ Worker Nodes<br/>- m5.xlarge<br/>- 2 On-Demand<br/>- 2 Spot Instances]
            end
            
            subgraph "Worker Nodes AZ-3"
                WorkerAZ3[🖥️ Worker Nodes<br/>- m5.xlarge<br/>- 2 On-Demand<br/>- 2 Spot Instances]
            end
            
            subgraph "Data Layer"
                RDS[🐘 RDS PostgreSQL<br/>- Multi-AZ<br/>- Read Replicas<br/>- Automated Backup]
                
                ElastiCache[🔴 ElastiCache Redis<br/>- Cluster Mode<br/>- Multi-AZ<br/>- Failover]
                
                EBS[💾 EBS Volumes<br/>- gp3 SSD<br/>- Encrypted<br/>- Snapshots]
            end
        end
        
        subgraph "External Services"
            ALB[⚖️ Application Load Balancer<br/>- SSL Termination<br/>- WAF Integration<br/>- Multi-AZ]
            
            S3[☁️ S3 Buckets<br/>- Documents Storage<br/>- Backup Storage<br/>- Static Assets]
            
            CloudFront[🌐 CloudFront CDN<br/>- Global Distribution<br/>- Edge Caching<br/>- SSL/TLS]
        end
    end
    
    Internet[🌍 Internet] --> CloudFront
    CloudFront --> ALB
    ALB --> PublicAZ1
    ALB --> PublicAZ2
    ALB --> PublicAZ3
    
    PublicAZ1 --> PrivateAZ1
    PublicAZ2 --> PrivateAZ2
    PublicAZ3 --> PrivateAZ3
    
    EKSMaster --> WorkerAZ1
    EKSMaster --> WorkerAZ2
    EKSMaster --> WorkerAZ3
    
    WorkerAZ1 --> PrivateAZ1
    WorkerAZ2 --> PrivateAZ2
    WorkerAZ3 --> PrivateAZ3
    
    WorkerAZ1 --> RDS
    WorkerAZ2 --> RDS
    WorkerAZ3 --> RDS
    
    WorkerAZ1 --> ElastiCache
    WorkerAZ2 --> ElastiCache
    WorkerAZ3 --> ElastiCache
    
    WorkerAZ1 --> EBS
    WorkerAZ2 --> EBS
    WorkerAZ3 --> EBS
    
    WorkerAZ1 --> S3
    WorkerAZ2 --> S3
    WorkerAZ3 --> S3
    
    style EKSMaster fill:#f3e5f5
    style WorkerAZ1 fill:#e8f5e8
    style WorkerAZ2 fill:#e8f5e8
    style WorkerAZ3 fill:#e8f5e8
    style RDS fill:#e3f2fd
    style ElastiCache fill:#ffebee
    style S3 fill:#fff8e1
```

### 4.2 Namespaces et Isolation

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "dpj-frontend Namespace"
            ClientApp[🌐 Client App<br/>- React SPA<br/>- NGINX<br/>- Replicas: 3]
            
            CollabApp[🌐 Collaborateur App<br/>- React SPA<br/>- NGINX<br/>- Replicas: 3]
        end
        
        subgraph "dpj-backend Namespace"
            APIGateway[🚪 API Gateway<br/>- Spring Cloud Gateway<br/>- Replicas: 3<br/>- HPA: 3-10]
            
            DossierService[📁 Dossier Service<br/>- Spring Boot<br/>- Replicas: 3<br/>- HPA: 3-10]
            
            DocumentService[📄 Document Service<br/>- Spring Boot<br/>- Replicas: 3<br/>- HPA: 3-10]
            
            AuthService[🔐 Auth Service<br/>- Spring Boot<br/>- Replicas: 2<br/>- HPA: 2-5]
            
            GEDService[🗄️ GED Service<br/>- Spring Boot<br/>- Replicas: 2<br/>- HPA: 2-8]
        end
        
        subgraph "dpj-data Namespace"
            PostgreSQL[🐘 PostgreSQL<br/>- StatefulSet<br/>- Replicas: 3<br/>- Primary + 2 Replicas]
            
            MongoDB[🍃 MongoDB<br/>- StatefulSet<br/>- Replicas: 3<br/>- ReplicaSet]
            
            Redis[🔴 Redis<br/>- StatefulSet<br/>- Replicas: 3<br/>- Cluster Mode]
            
            MinIO[📦 MinIO<br/>- StatefulSet<br/>- Replicas: 4<br/>- Distributed Mode]
        end
        
        subgraph "dpj-infrastructure Namespace"
            Kafka[📨 Kafka<br/>- StatefulSet<br/>- Replicas: 3<br/>- Zookeeper: 3]
            
            SchemaRegistry[📋 Schema Registry<br/>- Deployment<br/>- Replicas: 2]
        end
        
        subgraph "dpj-monitoring Namespace"
            Prometheus[📊 Prometheus<br/>- StatefulSet<br/>- Replicas: 2<br/>- HA Mode]
            
            Grafana[📈 Grafana<br/>- Deployment<br/>- Replicas: 2]
            
            Jaeger[🔍 Jaeger<br/>- Deployment<br/>- All-in-one]
            
            AlertManager[🚨 AlertManager<br/>- StatefulSet<br/>- Replicas: 3]
        end
        
        subgraph "Network Policies"
            FrontendPolicy[🔒 Frontend → Backend Only]
            BackendPolicy[🔒 Backend → Data Only]
            DataPolicy[🔒 Data → Internal Only]
            MonitoringPolicy[🔒 Monitoring → All Read]
        end
    end
    
    ClientApp --> APIGateway
    CollabApp --> APIGateway
    
    APIGateway --> DossierService
    APIGateway --> DocumentService
    APIGateway --> AuthService
    APIGateway --> GEDService
    
    DossierService --> PostgreSQL
    DocumentService --> PostgreSQL
    DocumentService --> MinIO
    AuthService --> Redis
    GEDService --> MongoDB
    
    DossierService --> Kafka
    DocumentService --> Kafka
    GEDService --> Kafka
    
    Prometheus --> APIGateway
    Prometheus --> DossierService
    Prometheus --> DocumentService
    Prometheus --> AuthService
    Prometheus --> GEDService
    
    style ClientApp fill:#e1f5fe
    style CollabApp fill:#e8f5e8
    style APIGateway fill:#f3e5f5
    style PostgreSQL fill:#e3f2fd
    style MongoDB fill:#e8f5e8
    style Redis fill:#ffebee
    style MinIO fill:#fff8e1
    style Kafka fill:#fff3e0
```

## 5. Architecture de Sécurité

### 5.1 Flux d'Authentification et Autorisation

```mermaid
sequenceDiagram
    participant User as 👤 Utilisateur
    participant Frontend as 🌐 Frontend App
    participant Gateway as 🚪 API Gateway
    participant Auth as 🔐 Auth Service
    participant LDAP as 👥 LDAP
    participant Redis as 🔴 Redis
    participant Backend as 🔧 Backend Service
    
    User->>Frontend: Login (email, password)
    Frontend->>Gateway: POST /auth/login
    Gateway->>Auth: Forward login request
    
    Auth->>LDAP: Validate credentials
    LDAP-->>Auth: User validated
    
    Auth->>Auth: Generate JWT tokens
    Auth->>Redis: Store session
    Auth-->>Gateway: JWT + Refresh Token
    Gateway-->>Frontend: Tokens + User info
    Frontend-->>User: Login successful
    
    Note over User,Backend: Subsequent API calls
    
    User->>Frontend: API request
    Frontend->>Gateway: Request + JWT token
    Gateway->>Gateway: Validate JWT signature
    Gateway->>Auth: Verify token & get permissions
    Auth->>Redis: Check session validity
    Redis-->>Auth: Session valid
    Auth-->>Gateway: User context + permissions
    
    Gateway->>Backend: Forward request + user context
    Backend->>Backend: Check permissions for resource
    Backend-->>Gateway: Response
    Gateway-->>Frontend: Response
    Frontend-->>User: Display result
    
    Note over User,Backend: Token refresh flow
    
    Frontend->>Gateway: Refresh token request
    Gateway->>Auth: Validate refresh token
    Auth->>Redis: Check refresh token
    Redis-->>Auth: Token valid
    Auth->>Auth: Generate new access token
    Auth-->>Gateway: New access token
    Gateway-->>Frontend: New token
```

### 5.2 Architecture de Chiffrement

```mermaid
graph TB
    subgraph "Data Encryption at Rest"
        subgraph "Database Encryption"
            PostgreSQLTDE[🐘 PostgreSQL<br/>- TDE (Transparent Data Encryption)<br/>- Column-level encryption<br/>- Key rotation]
            
            MongoDBEnc[🍃 MongoDB<br/>- Encryption at rest<br/>- Field-level encryption<br/>- KMIP integration]
        end
        
        subgraph "File Storage Encryption"
            MinIOEnc[📦 MinIO<br/>- Server-side encryption<br/>- AES-256-GCM<br/>- Per-object keys]
            
            EBSEnc[💾 EBS Volumes<br/>- AWS KMS encryption<br/>- AES-256<br/>- Automatic key rotation]
        end
    end
    
    subgraph "Data Encryption in Transit"
        TLSTermination[🔒 TLS Termination<br/>- TLS 1.3<br/>- Perfect Forward Secrecy<br/>- HSTS headers]
        
        ServiceMesh[🕸️ Service Mesh (Istio)<br/>- mTLS between services<br/>- Certificate rotation<br/>- Traffic encryption]
    end
    
    subgraph "Key Management"
        AWSKMS[🔑 AWS KMS<br/>- Master keys<br/>- Key rotation<br/>- Audit logging]
        
        VaultKMS[🔐 HashiCorp Vault<br/>- Dynamic secrets<br/>- PKI certificates<br/>- Secret rotation]
    end
    
    subgraph "Application Layer Encryption"
        DocumentEnc[📄 Document Encryption<br/>- Unique key per document<br/>- Envelope encryption<br/>- Secure key derivation]
        
        PIIEnc[🔒 PII Encryption<br/>- Format-preserving encryption<br/>- Tokenization<br/>- Data masking]
    end
    
    Internet[🌍 Internet] --> TLSTermination
    TLSTermination --> ServiceMesh
    ServiceMesh --> PostgreSQLTDE
    ServiceMesh --> MongoDBEnc
    ServiceMesh --> MinIOEnc
    
    AWSKMS --> EBSEnc
    AWSKMS --> MinIOEnc
    VaultKMS --> DocumentEnc
    VaultKMS --> PIIEnc
    
    DocumentEnc --> MinIOEnc
    PIIEnc --> PostgreSQLTDE
    
style TLSTermination fill:#ffebee
    style ServiceMesh fill:#f3e5f5
    style AWSKMS fill:#fff3e0
    style VaultKMS fill:#e8f5e8
    style DocumentEnc fill:#e1f5fe
```

## 6. Architecture de Monitoring et Observabilité

### 6.1 Stack de Monitoring

```mermaid
graph TB
    subgraph "Applications"
        Frontend[🌐 Frontend Apps]
        Backend[🔧 Backend Services]
        Databases[💾 Databases]
        Infrastructure[🏗️ Infrastructure]
    end
    
    subgraph "Metrics Collection"
        PrometheusServer[📊 Prometheus Server<br/>- Metrics scraping<br/>- Time series DB<br/>- Alert evaluation]
        
        NodeExporter[📈 Node Exporter<br/>- System metrics<br/>- Hardware monitoring<br/>- OS statistics]
        
        KubeStateMetrics[☸️ Kube State Metrics<br/>- Kubernetes objects<br/>- Resource states<br/>- Cluster health]
        
        AppMetrics[📱 Application Metrics<br/>- Custom metrics<br/>- Business KPIs<br/>- Performance counters]
    end
    
    subgraph "Logs Collection"
        FluentBit[📝 Fluent Bit<br/>- Log collection<br/>- Parsing & filtering<br/>- Multi-destination]
        
        Elasticsearch[🔍 Elasticsearch<br/>- Log storage<br/>- Full-text search<br/>- Log aggregation]
        
        Kibana[📊 Kibana<br/>- Log visualization<br/>- Dashboard creation<br/>- Query interface]
    end
    
    subgraph "Distributed Tracing"
        JaegerAgent[🔍 Jaeger Agent<br/>- Trace collection<br/>- Local buffering<br/>- Batch sending]
        
        JaegerCollector[📥 Jaeger Collector<br/>- Trace processing<br/>- Validation<br/>- Storage writing]
        
        JaegerQuery[🔎 Jaeger Query<br/>- Trace retrieval<br/>- UI service<br/>- API gateway]
        
        TraceStorage[(🗄️ Trace Storage<br/>- Elasticsearch<br/>- Cassandra<br/>- Retention policies)]
    end
    
    subgraph "Visualization & Alerting"
        Grafana[📈 Grafana<br/>- Dashboards<br/>- Visualization<br/>- Multi-datasource]
        
        AlertManager[🚨 AlertManager<br/>- Alert routing<br/>- Notification<br/>- Silencing]
        
        PagerDuty[📞 PagerDuty<br/>- Incident management<br/>- Escalation<br/>- On-call rotation]
    end
    
    Frontend --> AppMetrics
    Backend --> AppMetrics
    Backend --> JaegerAgent
    Databases --> NodeExporter
    Infrastructure --> NodeExporter
    Infrastructure --> KubeStateMetrics
    
    AppMetrics --> PrometheusServer
    NodeExporter --> PrometheusServer
    KubeStateMetrics --> PrometheusServer
    
    Frontend --> FluentBit
    Backend --> FluentBit
    FluentBit --> Elasticsearch
    Elasticsearch --> Kibana
    
    JaegerAgent --> JaegerCollector
    JaegerCollector --> TraceStorage
    TraceStorage --> JaegerQuery
    
    PrometheusServer --> Grafana
    Elasticsearch --> Grafana
    JaegerQuery --> Grafana
    
    PrometheusServer --> AlertManager
    AlertManager --> PagerDuty
    
    style PrometheusServer fill:#ff9800
    style Grafana fill:#e91e63
    style Elasticsearch fill:#00bcd4
    style JaegerCollector fill:#9c27b0
    style AlertManager fill:#f44336
```

### 6.2 Flux de Monitoring en Temps Réel

```mermaid
sequenceDiagram
    participant App as 🔧 Application
    participant Prometheus as 📊 Prometheus
    participant Grafana as 📈 Grafana
    participant AlertManager as 🚨 AlertManager
    participant PagerDuty as 📞 PagerDuty
    participant OnCall as 👨‍💻 On-Call Engineer
    
    loop Every 15 seconds
        Prometheus->>App: Scrape /metrics endpoint
        App-->>Prometheus: Metrics data (CPU, memory, requests, etc.)
    end
    
    Prometheus->>Prometheus: Evaluate alert rules
    
    alt High CPU Usage Detected
        Prometheus->>AlertManager: Fire alert: HighCPUUsage
        AlertManager->>AlertManager: Apply routing rules
        AlertManager->>PagerDuty: Send alert notification
        PagerDuty->>OnCall: SMS/Call notification
        OnCall->>Grafana: Check dashboard for details
        Grafana->>Prometheus: Query historical data
        Prometheus-->>Grafana: Time series data
        OnCall->>OnCall: Investigate and resolve
        OnCall->>AlertManager: Acknowledge alert
    end
    
    loop Real-time Dashboard Updates
        Grafana->>Prometheus: Query current metrics
        Prometheus-->>Grafana: Latest metric values
        Grafana->>Grafana: Update dashboard panels
    end
```

## 7. Architecture CI/CD

### 7.1 Pipeline de Déploiement

```mermaid
graph TB
    subgraph "Source Control"
        GitLab[📚 GitLab Repository<br/>- Source code<br/>- Infrastructure as Code<br/>- Documentation]
    end
    
    subgraph "CI/CD Pipeline"
        subgraph "Build Stage"
            BuildBackend[🔨 Build Backend<br/>- Maven compile<br/>- Unit tests<br/>- Code coverage]
            
            BuildFrontend[🔨 Build Frontend<br/>- npm build<br/>- Unit tests<br/>- Bundle optimization]
        end
        
        subgraph "Quality Stage"
            SonarQube[🔍 SonarQube<br/>- Code quality<br/>- Security scan<br/>- Technical debt]
            
            SecurityScan[🛡️ Security Scan<br/>- OWASP ZAP<br/>- Dependency check<br/>- Container scan]
        end
        
        subgraph "Package Stage"
            DockerBuild[🐳 Docker Build<br/>- Multi-stage build<br/>- Image optimization<br/>- Vulnerability scan]
            
            HelmPackage[⚓ Helm Package<br/>- Chart validation<br/>- Template rendering<br/>- Version tagging]
        end
        
        subgraph "Deploy Stage"
            DeployStaging[🚀 Deploy Staging<br/>- Automated deployment<br/>- Smoke tests<br/>- Integration tests]
            
            DeployProduction[🚀 Deploy Production<br/>- Manual approval<br/>- Blue-green deployment<br/>- Health checks]
        end
    end
    
    subgraph "Container Registry"
        DockerRegistry[📦 Docker Registry<br/>- Image storage<br/>- Vulnerability scanning<br/>- Retention policies]
    end
    
    subgraph "Kubernetes Clusters"
        StagingCluster[☸️ Staging Cluster<br/>- Feature testing<br/>- Performance testing<br/>- User acceptance]
        
        ProductionCluster[☸️ Production Cluster<br/>- Live environment<br/>- High availability<br/>- Monitoring]
    end
    
    subgraph "Monitoring & Feedback"
        Monitoring[📊 Monitoring<br/>- Application metrics<br/>- Infrastructure metrics<br/>- Business metrics]
        
        Alerting[🚨 Alerting<br/>- Automated alerts<br/>- Incident response<br/>- Escalation]
    end
    
    GitLab --> BuildBackend
    GitLab --> BuildFrontend
    
    BuildBackend --> SonarQube
    BuildFrontend --> SonarQube
    
    SonarQube --> SecurityScan
    SecurityScan --> DockerBuild
    DockerBuild --> HelmPackage
    
    DockerBuild --> DockerRegistry
    HelmPackage --> DeployStaging
    
    DockerRegistry --> StagingCluster
    DeployStaging --> StagingCluster
    
    StagingCluster --> DeployProduction
    DeployProduction --> ProductionCluster
    DockerRegistry --> ProductionCluster
    
    ProductionCluster --> Monitoring
    Monitoring --> Alerting
    Alerting --> GitLab
    
    style GitLab fill:#fc6d26
    style DockerBuild fill:#2496ed
    style StagingCluster fill:#326ce5
    style ProductionCluster fill:#ff6b6b
    style Monitoring fill:#4ecdc4
```

### 7.2 Stratégies de Déploiement

```mermaid
graph TB
    subgraph "Blue-Green Deployment"
        subgraph "Current State"
            BlueEnv[🔵 Blue Environment<br/>- Current production<br/>- Serving 100% traffic<br/>- Version 1.0]
            
            GreenEnv[🟢 Green Environment<br/>- New version<br/>- No traffic<br/>- Version 1.1]
        end
        
        LoadBalancer[⚖️ Load Balancer<br/>- Traffic routing<br/>- Health checks<br/>- Instant switch]
        
        subgraph "After Switch"
            BlueEnvOld[🔵 Blue Environment<br/>- Previous version<br/>- No traffic<br/>- Standby for rollback]
            
            GreenEnvActive[🟢 Green Environment<br/>- New production<br/>- Serving 100% traffic<br/>- Version 1.1]
        end
    end
    
    subgraph "Canary Deployment"
        subgraph "Traffic Split"
            StableVersion[📦 Stable Version<br/>- Version 1.0<br/>- 90% traffic<br/>- Proven stable]
            
            CanaryVersion[🐤 Canary Version<br/>- Version 1.1<br/>- 10% traffic<br/>- Under evaluation]
        end
        
        TrafficSplitter[🔀 Traffic Splitter<br/>- Istio/Envoy<br/>- Gradual rollout<br/>- Automatic rollback]
        
        subgraph "Monitoring"
            CanaryMetrics[📊 Canary Metrics<br/>- Error rates<br/>- Response times<br/>- Business KPIs]
            
            AutoRollback[🔄 Auto Rollback<br/>- Threshold-based<br/>- Immediate response<br/>- Safety net]
        end
    end
    
    Users[👥 Users] --> LoadBalancer
    LoadBalancer --> BlueEnv
    LoadBalancer -.-> GreenEnv
    
    Users --> TrafficSplitter
    TrafficSplitter --> StableVersion
    TrafficSplitter --> CanaryVersion
    
    CanaryVersion --> CanaryMetrics
    CanaryMetrics --> AutoRollback
    AutoRollback --> TrafficSplitter
    
    style BlueEnv fill:#2196f3
    style GreenEnv fill:#4caf50
    style StableVersion fill:#ff9800
    style CanaryVersion fill:#ffeb3b
    style AutoRollback fill:#f44336
```

## 8. Architecture de Disaster Recovery

### 8.1 Plan de Continuité d'Activité

```mermaid
graph TB
    subgraph "Primary Region (eu-west-1)"
        subgraph "Production Environment"
            PrimaryCluster[☸️ Primary K8s Cluster<br/>- Active workloads<br/>- Real-time processing<br/>- User traffic]
            
            PrimaryDB[(🐘 Primary Database<br/>- Read/Write operations<br/>- Real-time data<br/>- Transaction processing)]
            
            PrimaryStorage[📦 Primary Storage<br/>- Active documents<br/>- User uploads<br/>- Application data]
        end
    end
    
    subgraph "Secondary Region (eu-central-1)"
        subgraph "DR Environment"
            DRCluster[☸️ DR K8s Cluster<br/>- Standby workloads<br/>- Minimal resources<br/>- Ready for activation]
            
            DRDatabase[(🐘 DR Database<br/>- Read replica<br/>- Async replication<br/>- Backup restoration)]
            
            DRStorage[📦 DR Storage<br/>- Replicated documents<br/>- Cross-region sync<br/>- Point-in-time recovery]
        end
    end
    
    subgraph "Backup & Recovery"
        VeleroBackup[💾 Velero Backup<br/>- K8s resources<br/>- PV snapshots<br/>- Cross-region backup]
        
        DatabaseBackup[🗄️ Database Backup<br/>- Automated backups<br/>- Point-in-time recovery<br/>- Cross-region replication]
        
        S3CrossRegion[☁️ S3 Cross-Region<br/>- Document replication<br/>- Versioning<br/>- Lifecycle policies]
    end
    
    subgraph "Monitoring & Alerting"
        HealthChecks[❤️ Health Checks<br/>- Service monitoring<br/>- Database connectivity<br/>- Storage availability]
        
        FailoverTrigger[🚨 Failover Trigger<br/>- Automated detection<br/>- Manual override<br/>- Escalation procedures]
    end
    
    subgraph "DNS & Traffic Management"
        Route53[🌐 Route 53<br/>- Health-based routing<br/>- Automatic failover<br/>- Global load balancing]
        
        CloudFront[☁️ CloudFront<br/>- Origin failover<br/>- Edge caching<br/>- SSL termination]
    end
    
    PrimaryCluster --> PrimaryDB
    PrimaryCluster --> PrimaryStorage
    
    PrimaryDB --> DRDatabase
    PrimaryStorage --> DRStorage
    
    PrimaryCluster --> VeleroBackup
    PrimaryDB --> DatabaseBackup
    PrimaryStorage --> S3CrossRegion
    
    VeleroBackup --> DRCluster
    DatabaseBackup --> DRDatabase
    S3CrossRegion --> DRStorage
    
    PrimaryCluster --> HealthChecks
    DRCluster --> HealthChecks
    HealthChecks --> FailoverTrigger
    
    FailoverTrigger --> Route53
    Route53 --> CloudFront
    CloudFront --> PrimaryCluster
    CloudFront -.-> DRCluster
    
    style PrimaryCluster fill:#4caf50
    style DRCluster fill:#ff9800
    style FailoverTrigger fill:#f44336
    style Route53 fill:#2196f3
```

### 8.2 Procédure de Failover

```mermaid
sequenceDiagram
    participant Monitor as 📊 Monitoring System
    participant Primary as 🟢 Primary Region
    participant DR as 🟠 DR Region
    participant DNS as 🌐 DNS (Route 53)
    participant Ops as 👨‍💻 Operations Team
    participant Users as 👥 Users
    
    Note over Monitor,Users: Normal Operations
    Users->>DNS: Request app.dpj.banque.fr
    DNS->>Primary: Route to primary region
    Primary-->>Users: Serve application
    
    Note over Monitor,Users: Disaster Scenario
    Monitor->>Primary: Health check
    Primary-->>Monitor: No response (timeout)
    Monitor->>Monitor: Retry health checks (3 attempts)
    Monitor->>Ops: Alert: Primary region down
    
    Note over Monitor,Users: Automated Failover
    Monitor->>DNS: Update health check status
    DNS->>DNS: Mark primary as unhealthy
    DNS->>DR: Route traffic to DR region
    
    Note over Monitor,Users: DR Activation
    Ops->>DR: Scale up DR cluster
    DR->>DR: Activate standby services
    DR->>DR: Promote read replica to primary
    DR->>DR: Update application configuration
    
    Note over Monitor,Users: Service Restoration
    Users->>DNS: Request app.dpj.banque.fr
    DNS->>DR: Route to DR region
    DR-->>Users: Serve application (degraded mode)
    
    Note over Monitor,Users: Recovery Process
    Ops->>Primary: Investigate and fix issues
    Primary->>Primary: Restore services
    Ops->>Monitor: Validate primary health
    Monitor->>DNS: Update health check status
    DNS->>Primary: Route traffic back to primary
    Ops->>DR: Scale down DR cluster
```

## 9. Architecture de Performance

### 9.1 Optimisations de Performance

```mermaid
graph TB
    subgraph "Client-Side Optimizations"
        CDN[☁️ CDN (CloudFront)<br/>- Static asset caching<br/>- Edge locations<br/>- Gzip compression]
        
        BrowserCache[🌐 Browser Cache<br/>- Cache headers<br/>- Service workers<br/>- Local storage]
        
        CodeSplitting[📦 Code Splitting<br/>- Lazy loading<br/>- Dynamic imports<br/>- Bundle optimization]
    end
    
    subgraph "Network Optimizations"
        HTTP2[🔗 HTTP/2<br/>- Multiplexing<br/>- Server push<br/>- Header compression]
        
        LoadBalancer[⚖️ Load Balancer<br/>- Connection pooling<br/>- Keep-alive<br/>- SSL termination]
        
        ServiceMesh[🕸️ Service Mesh<br/>- Circuit breakers<br/>- Retry policies<br/>- Load balancing]
    end
    
    subgraph "Application Optimizations"
        APIGateway[🚪 API Gateway<br/>- Request caching<br/>- Rate limiting<br/>- Response compression]
        
        Microservices[🔧 Microservices<br/>- Async processing<br/>- Connection pooling<br/>- JVM tuning]
        
        Caching[🔴 Redis Cache<br/>- Application cache<br/>- Session cache<br/>- Query cache]
    end
    
    subgraph "Database Optimizations"
        ReadReplicas[(📖 Read Replicas<br/>- Read scaling<br/>- Query distribution<br/>- Geo-distribution)]
        
        ConnectionPool[🏊 Connection Pool<br/>- HikariCP<br/>- Connection reuse<br/>- Pool sizing]
        
        QueryOptimization[🔍 Query Optimization<br/>- Index tuning<br/>- Query plans<br/>- Partitioning]
    end
    
    subgraph "Storage Optimizations"
        MinIOOptimization[📦 MinIO Optimization<br/>- Distributed storage<br/>- Erasure coding<br/>- Parallel uploads]
        
        S3Transfer[☁️ S3 Transfer<br/>- Multipart uploads<br/>- Transfer acceleration<br/>- Intelligent tiering]
    end
    
    Users[👥 Users] --> CDN
    CDN --> BrowserCache
    BrowserCache --> CodeSplitting
    
    CodeSplitting --> HTTP2
    HTTP2 --> LoadBalancer
    LoadBalancer --> ServiceMesh
    
    ServiceMesh --> APIGateway
    APIGateway --> Microservices
    Microservices --> Caching
    
    Microservices --> ReadReplicas
    ReadReplicas --> ConnectionPool
    ConnectionPool --> QueryOptimization
    
    Microservices --> MinIOOptimization
    MinIOOptimization --> S3Transfer
    
    style CDN fill:#4caf50
    style Caching fill:#ff5722
    style ReadReplicas fill:#2196f3
    style MinIOOptimization fill:#ff9800
```

### 9.2 Métriques de Performance

```mermaid
graph TB
    subgraph "Frontend Metrics"
        WebVitals[📊 Web Vitals<br/>- LCP: < 2.5s<br/>- FID: < 100ms<br/>- CLS: < 0.1]
        
        LoadTime[⏱️ Load Time<br/>- First Paint: < 1s<br/>- Time to Interactive: < 3s<br/>- Bundle Size: < 1MB]
    end
    
    subgraph "API Metrics"
        ResponseTime[⚡ Response Time<br/>- P95: < 500ms<br/>- P99: < 1s<br/>- Average: < 200ms]
        
        Throughput[📈 Throughput<br/>- RPS: 1000+<br/>- Concurrent Users: 500+<br/>- Peak Load: 2000 RPS]
        
        ErrorRate[❌ Error Rate<br/>- 4xx Errors: < 1%<br/>- 5xx Errors: < 0.1%<br/>- Timeout Rate: < 0.01%]
    end
    
    subgraph "Database Metrics"
        QueryPerformance[🔍 Query Performance<br/>- Average Query Time: < 50ms<br/>- Slow Queries: < 1%<br/>- Connection Pool: 80% max]
        
        DatabaseLoad[💾 Database Load<br/>- CPU Usage: < 70%<br/>- Memory Usage: < 80%<br/>- Disk I/O: < 80%]
    end
    
    subgraph "Infrastructure Metrics"
        ResourceUsage[🖥️ Resource Usage<br/>- CPU: < 70%<br/>- Memory: < 80%<br/>- Network: < 80%]
        
        Availability[✅ Availability<br/>- Uptime: 99.9%<br/>- MTTR: < 15min<br/>- MTBF: > 720h]
    end
    
    subgraph "Business Metrics"
        UserExperience[👤 User Experience<br/>- Session Duration: > 5min<br/>- Bounce Rate: < 20%<br/>- Task Completion: > 95%]
        
        DocumentProcessing[📄 Document Processing<br/>- Upload Success: > 99%<br/>- Processing Time: < 30s<br/>- Validation Time: < 2min]
    end
    
    WebVitals --> ResponseTime
    LoadTime --> Throughput
    ResponseTime --> QueryPerformance
    Throughput --> DatabaseLoad
    QueryPerformance --> ResourceUsage
    DatabaseLoad --> Availability
    ResourceUsage --> UserExperience
    Availability --> DocumentProcessing
    
    style WebVitals fill:#4caf50
    style ResponseTime fill:#2196f3
    style QueryPerformance fill:#ff9800
    style ResourceUsage fill:#9c27b0
    style UserExperience fill:#e91e63
```

Cette architecture complète de diagrammes fournit une vision claire et détaillée du système DPJ, facilitant la compréhension et la communication entre les équipes techniques et métier.