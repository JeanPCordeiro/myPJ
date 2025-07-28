# Architecture des Données et Modèles de Données

## 1. Vue d'Ensemble de l'Architecture des Données

### Stratégie Multi-Base
- **PostgreSQL** : Données transactionnelles et relationnelles
- **MongoDB** : Métadonnées étendues et documents non-structurés
- **Redis** : Cache et sessions
- **MinIO/S3** : Stockage des fichiers binaires

### Principes de Conception
- **Normalisation** : Éviter la redondance des données critiques
- **Dénormalisation Contrôlée** : Optimiser les requêtes fréquentes
- **Séparation des Préoccupations** : Données métier vs. données techniques
- **Évolutivité** : Support des changements de schéma

## 2. Modèle de Données Relationnel (PostgreSQL)

### 2.1 Domaine Dossier

#### Table `dossiers`
```sql
CREATE TABLE dossiers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    numero_dossier VARCHAR(50) UNIQUE NOT NULL,
    client_id VARCHAR(50) NOT NULL,
    type_parcours VARCHAR(100) NOT NULL,
    statut VARCHAR(20) NOT NULL CHECK (statut IN ('BROUILLON', 'EN_COURS', 'COMPLET', 'VALIDE', 'ARCHIVE')),
    etape_courante VARCHAR(50),
    priorite INTEGER DEFAULT 0,
    date_creation TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    date_modification TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    date_echeance TIMESTAMP WITH TIME ZONE,
    collaborateur_id VARCHAR(50),
    commentaire_interne TEXT,
    metadata JSONB,
    version INTEGER DEFAULT 1,
    created_by VARCHAR(50) NOT NULL,
    updated_by VARCHAR(50) NOT NULL
);

-- Index pour performance
CREATE INDEX idx_dossiers_client_statut ON dossiers(client_id, statut);
CREATE INDEX idx_dossiers_collaborateur ON dossiers(collaborateur_id) WHERE collaborateur_id IS NOT NULL;
CREATE INDEX idx_dossiers_type_parcours ON dossiers(type_parcours);
CREATE INDEX idx_dossiers_date_creation ON dossiers(date_creation);
```

#### Table `etapes_parcours`
```sql
CREATE TABLE etapes_parcours (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dossier_id UUID NOT NULL REFERENCES dossiers(id) ON DELETE CASCADE,
    nom_etape VARCHAR(100) NOT NULL,
    code_etape VARCHAR(50) NOT NULL,
    ordre_etape INTEGER NOT NULL,
    statut VARCHAR(20) NOT NULL CHECK (statut IN ('EN_ATTENTE', 'EN_COURS', 'COMPLETE', 'BLOQUEE')),
    date_debut TIMESTAMP WITH TIME ZONE,
    date_fin TIMESTAMP WITH TIME ZONE,
    duree_estimee_heures INTEGER,
    documents_obligatoires JSONB NOT NULL DEFAULT '[]',
    documents_optionnels JSONB NOT NULL DEFAULT '[]',
    regles_validation JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_etapes_dossier_ordre ON etapes_parcours(dossier_id, ordre_etape);
CREATE INDEX idx_etapes_statut ON etapes_parcours(statut);
```

### 2.2 Domaine Document

#### Table `documents`
```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dossier_id UUID NOT NULL REFERENCES dossiers(id) ON DELETE CASCADE,
    etape_id UUID REFERENCES etapes_parcours(id),
    nom_fichier VARCHAR(255) NOT NULL,
    nom_original VARCHAR(255) NOT NULL,
    type_document VARCHAR(100) NOT NULL,
    categorie_document VARCHAR(50),
    mime_type VARCHAR(100) NOT NULL,
    taille_fichier BIGINT NOT NULL,
    checksum_sha256 VARCHAR(64) NOT NULL,
    chemin_stockage VARCHAR(500) NOT NULL,
    statut VARCHAR(20) NOT NULL CHECK (statut IN ('EN_ATTENTE', 'VALIDE', 'REJETE', 'ARCHIVE')),
    est_obligatoire BOOLEAN DEFAULT FALSE,
    date_depot TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    date_validation TIMESTAMP WITH TIME ZONE,
    date_expiration TIMESTAMP WITH TIME ZONE,
    deposant_id VARCHAR(50) NOT NULL,
    validateur_id VARCHAR(50),
    motif_rejet TEXT,
    commentaire_validation TEXT,
    numero_version INTEGER DEFAULT 1,
    document_parent_id UUID REFERENCES documents(id),
    metadata JSONB DEFAULT '{}',
    tags VARCHAR(50)[] DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index pour performance
CREATE INDEX idx_documents_dossier_statut ON documents(dossier_id, statut);
CREATE INDEX idx_documents_type ON documents(type_document);
CREATE INDEX idx_documents_deposant ON documents(deposant_id);
CREATE INDEX idx_documents_checksum ON documents(checksum_sha256);
CREATE INDEX idx_documents_date_depot ON documents(date_depot);
```

#### Table `types_documents`
```sql
CREATE TABLE types_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_type VARCHAR(50) UNIQUE NOT NULL,
    libelle VARCHAR(200) NOT NULL,
    description TEXT,
    formats_autorises VARCHAR(20)[] NOT NULL DEFAULT '{pdf,jpg,jpeg,png}',
    taille_max_mb INTEGER DEFAULT 10,
    est_actif BOOLEAN DEFAULT TRUE,
    regles_validation JSONB,
    template_metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO types_documents (code_type, libelle, formats_autorises, taille_max_mb) VALUES
('PIECE_IDENTITE', 'Pièce d''identité', '{pdf,jpg,jpeg,png}', 5),
('JUSTIFICATIF_DOMICILE', 'Justificatif de domicile', '{pdf,jpg,jpeg,png}', 5),
('BULLETIN_SALAIRE', 'Bulletin de salaire', '{pdf}', 10),
('RIB', 'Relevé d''identité bancaire', '{pdf,jpg,jpeg,png}', 5),
('CONTRAT_TRAVAIL', 'Contrat de travail', '{pdf}', 20);
```

### 2.3 Domaine Utilisateur et Sécurité

#### Table `utilisateurs`
```sql
CREATE TABLE utilisateurs (
    id VARCHAR(50) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    nom VARCHAR(100) NOT NULL,
    prenom VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('CLIENT', 'COLLABORATEUR', 'ADMIN')),
    statut VARCHAR(20) DEFAULT 'ACTIF' CHECK (statut IN ('ACTIF', 'INACTIF', 'SUSPENDU')),
    derniere_connexion TIMESTAMP WITH TIME ZONE,
    preferences JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_utilisateurs_email ON utilisateurs(email);
CREATE INDEX idx_utilisateurs_role ON utilisateurs(role);
```

#### Table `sessions`
```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    utilisateur_id VARCHAR(50) NOT NULL REFERENCES utilisateurs(id),
    token_hash VARCHAR(64) NOT NULL,
    date_creation TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    date_expiration TIMESTAMP WITH TIME ZONE NOT NULL,
    ip_address INET,
    user_agent TEXT,
    est_actif BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_sessions_token ON sessions(token_hash);
CREATE INDEX idx_sessions_utilisateur ON sessions(utilisateur_id);
```

### 2.4 Domaine Audit et Traçabilité

#### Table `audit_logs`
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    utilisateur_id VARCHAR(50) NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    details JSONB,
    ancien_etat JSONB,
    nouvel_etat JSONB
);

-- Partitioning par mois pour performance
CREATE INDEX idx_audit_entity_date ON audit_logs(entity_id, timestamp);
CREATE INDEX idx_audit_utilisateur_date ON audit_logs(utilisateur_id, timestamp);
CREATE INDEX idx_audit_action ON audit_logs(action);
```

## 3. Modèle de Données NoSQL (MongoDB)

### 3.1 Collection `documents_metadata`
```javascript
{
  _id: ObjectId,
  document_id: UUID, // Référence vers PostgreSQL
  metadata_extended: {
    // Métadonnées techniques
    exif_data: Object,
    ocr_text: String,
    thumbnail_path: String,
    preview_path: String,
    
    // Métadonnées métier
    mots_cles: [String],
    classification: String,
    confidentialite: String,
    retention_period: Number,
    
    // Métadonnées de traitement
    processing_status: String,
    virus_scan_result: {
      status: String, // 'clean', 'infected', 'pending'
      scan_date: Date,
      engine_version: String
    },
    
    // Historique des versions
    versions: [{
      version_number: Number,
      file_path: String,
      checksum: String,
      size: Number,
      created_date: Date,
      created_by: String,
      changes_description: String
    }]
  },
  
  // Index de recherche full-text
  search_index: {
    content: String, // Contenu extrait pour recherche
    tags: [String],
    keywords: [String]
  },
  
  created_at: Date,
  updated_at: Date
}

// Index MongoDB
db.documents_metadata.createIndex({ "document_id": 1 }, { unique: true })
db.documents_metadata.createIndex({ "search_index.content": "text", "search_index.tags": "text" })
db.documents_metadata.createIndex({ "metadata_extended.classification": 1 })
```

### 3.2 Collection `dossiers_analytics`
```javascript
{
  _id: ObjectId,
  dossier_id: UUID,
  analytics: {
    // Métriques de performance
    temps_traitement_total: Number, // en heures
    temps_par_etape: [{
      etape: String,
      duree: Number,
      date_debut: Date,
      date_fin: Date
    }],
    
    // Métriques de qualité
    nombre_rejets: Number,
    taux_completion: Number,
    score_qualite: Number,
    
    // Métriques d'interaction
    nombre_consultations: Number,
    derniere_consultation: Date,
    utilisateurs_impliques: [String],
    
    // Alertes et notifications
    alertes_generees: [{
      type: String,
      message: String,
      date: Date,
      resolu: Boolean
    }]
  },
  
  created_at: Date,
  updated_at: Date
}

db.dossiers_analytics.createIndex({ "dossier_id": 1 }, { unique: true })
db.dossiers_analytics.createIndex({ "analytics.temps_traitement_total": 1 })
```

## 4. Modèle de Cache (Redis)

### 4.1 Structure des Clés Redis

#### Sessions Utilisateur
```
session:{token_hash} → {
  user_id: string,
  role: string,
  permissions: array,
  expires_at: timestamp
}
TTL: 24 heures
```

#### Cache des Dossiers
```
dossier:{dossier_id} → {
  id: uuid,
  statut: string,
  etape_courante: string,
  documents_count: number,
  last_updated: timestamp
}
TTL: 15 minutes
```

#### Cache des Documents
```
documents:list:{dossier_id} → [
  {
    id: uuid,
    nom_fichier: string,
    statut: string,
    type_document: string
  }
]
TTL: 10 minutes
```

#### Configuration Applicative
```
config:{environment}:{service} → {
  database_url: string,
  api_endpoints: object,
  feature_flags: object
}
TTL: 1 heure
```

## 5. Modèle de Données pour le Stockage de Fichiers

### 5.1 Structure MinIO/S3

#### Organisation des Buckets
```
dpj-documents-prod/
├── documents/
│   ├── {year}/
│   │   ├── {month}/
│   │   │   ├── {dossier_id}/
│   │   │   │   ├── {document_id}.{extension}
│   │   │   │   └── thumbnails/
│   │   │   │       └── {document_id}_thumb.jpg
├── archives/
│   └── {year}/
│       └── archived_documents/
└── temp/
    └── uploads/
        └── {session_id}/
```

#### Métadonnées S3
```json
{
  "Content-Type": "application/pdf",
  "x-amz-meta-document-id": "uuid",
  "x-amz-meta-dossier-id": "uuid",
  "x-amz-meta-uploaded-by": "user_id",
  "x-amz-meta-checksum": "sha256_hash",
  "x-amz-meta-virus-scan": "clean",
  "x-amz-server-side-encryption": "AES256"
}
```

## 6. Modèles de Données Métier (Domain Objects)

### 6.1 Entité Dossier
```java
@Entity
@Table(name = "dossiers")
public class Dossier {
    @Id
    private UUID id;
    
    @Column(name = "numero_dossier", unique = true)
    private String numeroDossier;
    
    @Column(name = "client_id")
    private String clientId;
    
    @Enumerated(EnumType.STRING)
    private TypeParcours typeParcours;
    
    @Enumerated(EnumType.STRING)
    private StatutDossier statut;
    
    @OneToMany(mappedBy = "dossier", cascade = CascadeType.ALL)
    private List<EtapeParcours> etapes;
    
    @OneToMany(mappedBy = "dossier", cascade = CascadeType.ALL)
    private List<Document> documents;
    
    // Getters, setters, méthodes métier
    
    public boolean isComplet() {
        return etapes.stream()
            .allMatch(EtapeParcours::isComplete);
    }
    
    public List<Document> getDocumentsManquants() {
        return etapes.stream()
            .flatMap(etape -> etape.getDocumentsObligatoiresManquants().stream())
            .collect(Collectors.toList());
    }
}
```

### 6.2 Entité Document
```java
@Entity
@Table(name = "documents")
public class Document {
    @Id
    private UUID id;
    
    @ManyToOne
    @JoinColumn(name = "dossier_id")
    private Dossier dossier;
    
    @Column(name = "nom_fichier")
    private String nomFichier;
    
    @Enumerated(EnumType.STRING)
    private TypeDocument typeDocument;
    
    @Enumerated(EnumType.STRING)
    private StatutDocument statut;
    
    @Column(name = "taille_fichier")
    private Long tailleFichier;
    
    @Column(name = "chemin_stockage")
    private String cheminStockage;
    
    // Méthodes métier
    
    public boolean isValide() {
        return StatutDocument.VALIDE.equals(this.statut);
    }
    
    public boolean isExpire() {
        return dateExpiration != null && 
               dateExpiration.isBefore(LocalDateTime.now());
    }
    
    public void valider(String validateurId, String commentaire) {
        this.statut = StatutDocument.VALIDE;
        this.validateurId = validateurId;
        this.commentaireValidation = commentaire;
        this.dateValidation = LocalDateTime.now();
    }
}
```

## 7. Stratégies de Migration et Évolution

### 7.1 Versioning des Schémas

#### PostgreSQL - Flyway
```sql
-- V1.0.0__Initial_schema.sql
-- V1.1.0__Add_document_tags.sql
-- V1.2.0__Add_audit_partitioning.sql
```

#### MongoDB - Migrations
```javascript
// migration_v1_1_0.js
db.documents_metadata.updateMany(
  { "metadata_extended.search_index": { $exists: false } },
  { $set: { "metadata_extended.search_index": {} } }
);
```

### 7.2 Stratégie de Backup

#### Backup Automatisé
```bash
# PostgreSQL
pg_dump -h localhost -U dpj_user -d dpj_db > backup_$(date +%Y%m%d_%H%M%S).sql

# MongoDB
mongodump --host localhost --db dpj_metadata --out /backup/mongo/

# MinIO
mc mirror dpj-minio/dpj-documents-prod /backup/files/
```

## 8. Contraintes et Règles de Validation

### 8.1 Contraintes de Données

#### Règles Métier en Base
```sql
-- Un dossier ne peut avoir qu'une seule étape en cours
CREATE UNIQUE INDEX idx_unique_etape_en_cours 
ON etapes_parcours(dossier_id) 
WHERE statut = 'EN_COURS';

-- Un document ne peut être validé que s'il est en attente
ALTER TABLE documents ADD CONSTRAINT chk_validation_statut 
CHECK (
  (statut = 'VALIDE' AND date_validation IS NOT NULL AND validateur_id IS NOT NULL) OR
  (statut != 'VALIDE')
);
```

### 8.2 Validation Applicative

#### Règles de Validation Document
```java
@Component
public class DocumentValidator {
    
    public ValidationResult validate(Document document) {
        ValidationResult result = new ValidationResult();
        
        // Validation taille
        if (document.getTailleFichier() > getMaxSize(document.getTypeDocument())) {
            result.addError("Taille de fichier dépassée");
        }
        
        // Validation format
        if (!isFormatAutorise(document.getMimeType(), document.getTypeDocument())) {
            result.addError("Format de fichier non autorisé");
        }
        
        // Validation virus
        if (!isVirusScanClean(document.getChecksum())) {
            result.addError("Document infecté détecté");
        }
        
        return result;
    }
}
```

Cette architecture de données robuste permet de gérer efficacement les 500+ utilisateurs simultanés tout en maintenant l'intégrité et la performance du système.