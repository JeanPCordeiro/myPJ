# ADR-0011: Adoption des Patterns DDD et CQRS

## Statut

Accepté

## Contexte

Le système DPJ présente une complexité métier importante avec :
- Domaines métier distincts (Documents, Dossiers, Utilisateurs, GED)
- Besoins de performance différenciés (lecture vs écriture)
- Évolutions métier fréquentes
- Équipes multiples travaillant sur différents domaines
- Besoin de traçabilité et d'audit complets

Les approches traditionnelles CRUD ne permettent pas de :
- Capturer la richesse du domaine métier
- Optimiser séparément les lectures et écritures
- Maintenir un historique complet des changements
- Faciliter l'évolution indépendante des domaines

## Options Considérées

### Option 1: Architecture CRUD traditionnelle
- **Avantages** : Simplicité, familiarité équipe
- **Inconvénients** : Couplage fort, pas d'optimisation lecture/écriture

### Option 2: Domain Driven Design (DDD) seul
- **Avantages** : Modélisation métier riche, bounded contexts
- **Inconvénients** : Pas d'optimisation performance lecture/écriture

### Option 3: CQRS seul
- **Avantages** : Séparation lecture/écriture, performance
- **Inconvénients** : Pas de modélisation métier structurée

### Option 4: DDD + CQRS + Event Sourcing
- **Avantages** : Modélisation riche, performance, traçabilité complète
- **Inconvénients** : Complexité élevée, courbe d'apprentissage

## Décision

Nous adoptons **DDD + CQRS avec Event Sourcing partiel** :

### Domain Driven Design (DDD)

#### Bounded Contexts
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Document      │  │    Dossier      │  │   Utilisateur   │
│   Context       │  │    Context      │  │    Context      │
│                 │  │                 │  │                 │
│ - Document      │  │ - Dossier       │  │ - User          │
│ - Metadata      │  │ - Parcours      │  │ - Role          │
│ - Version       │  │ - Validation    │  │ - Permission    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                    ┌─────────────────┐
                    │      GED        │
                    │   Integration   │
                    │    Context      │
                    │                 │
                    │ - GedDocument   │
                    │ - Synchronizer  │
                    │ - Adapter       │
                    └─────────────────┘
```

#### Aggregates et Entities
```java
// Document Aggregate
@Entity
public class Document {
    @Id
    private DocumentId id;
    private String nom;
    private TypeDocument type;
    private StatutDocument statut;
    private List<DocumentEvent> events = new ArrayList<>();
    
    public void valider(ValidateurId validateur) {
        if (!this.statut.peutEtreValide()) {
            throw new DocumentNePeutPasEtreValideException();
        }
        
        this.statut = StatutDocument.VALIDE;
        this.addEvent(new DocumentValideEvent(this.id, validateur));
    }
    
    private void addEvent(DomainEvent event) {
        this.events.add(event);
    }
}

// Value Objects
@Value
public class DocumentId {
    private final String value;
}

@Value
public class TypeDocument {
    private final String code;
    private final String libelle;
}
```

### CQRS (Command Query Responsibility Segregation)

#### Architecture CQRS
```
Commands → Command Handlers → Domain Model → Events → Event Handlers → Read Models
                                    ↓
                              Write Database
                                    
Queries → Query Handlers → Read Models → Read Database
```

#### Command Side (Écriture)
```java
// Commands
public record CreerDocumentCommand(
    String nom,
    String type,
    byte[] contenu,
    String clientId
) {}

// Command Handlers
@Component
public class DocumentCommandHandler {
    
    private final DocumentRepository repository;
    private final EventPublisher eventPublisher;
    
    @CommandHandler
    public void handle(CreerDocumentCommand command) {
        Document document = Document.creer(
            command.nom(),
            TypeDocument.of(command.type()),
            command.contenu(),
            ClientId.of(command.clientId())
        );
        
        repository.save(document);
        
        // Publier les événements
        document.getEvents().forEach(eventPublisher::publish);
        document.clearEvents();
    }
}
```

#### Query Side (Lecture)
```java
// Read Models optimisés
@Document(collection = "document_views")
public class DocumentView {
    private String id;
    private String nom;
    private String type;
    private String statut;
    private String clientId;
    private LocalDateTime dateCreation;
    private LocalDateTime derniereModification;
    
    // Données dénormalisées pour performance
    private String nomClient;
    private String typeLibelle;
    private List<String> tags;
}

// Query Handlers
@Component
public class DocumentQueryHandler {
    
    private final DocumentViewRepository repository;
    
    @QueryHandler
    public List<DocumentView> handle(RechercherDocumentsQuery query) {
        return repository.findByClientIdAndType(
            query.clientId(),
            query.type()
        );
    }
}
```

### Event Sourcing Partiel

#### Events de Domaine
```java
// Base Event
public abstract class DomainEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final LocalDateTime occurredOn = LocalDateTime.now();
    private final String aggregateId;
    
    protected DomainEvent(String aggregateId) {
        this.aggregateId = aggregateId;
    }
}

// Events spécifiques
public class DocumentCreeEvent extends DomainEvent {
    private final String nom;
    private final String type;
    private final String clientId;
    
    public DocumentCreeEvent(String documentId, String nom, String type, String clientId) {
        super(documentId);
        this.nom = nom;
        this.type = type;
        this.clientId = clientId;
    }
}

public class DocumentValideEvent extends DomainEvent {
    private final String validateurId;
    private final LocalDateTime dateValidation;
    
    public DocumentValideEvent(String documentId, String validateurId) {
        super(documentId);
        this.validateurId = validateurId;
        this.dateValidation = LocalDateTime.now();
    }
}
```

#### Event Store
```java
@Entity
@Table(name = "event_store")
public class StoredEvent {
    @Id
    private String eventId;
    private String aggregateId;
    private String eventType;
    private String eventData; // JSON
    private LocalDateTime occurredOn;
    private Long version;
}

@Repository
public interface EventStoreRepository extends JpaRepository<StoredEvent, String> {
    List<StoredEvent> findByAggregateIdOrderByVersionAsc(String aggregateId);
}
```

## Conséquences

### Positives
- **Modélisation Riche** : Capture de la complexité métier
- **Séparation des Responsabilités** : Optimisation lecture/écriture
- **Évolutivité** : Bounded contexts indépendants
- **Performance** : Read models optimisés
- **Traçabilité** : Historique complet via Event Sourcing
- **Testabilité** : Domain logic isolée et testable

### Négatives
- **Complexité** : Architecture plus complexe
- **Courbe d'Apprentissage** : Nouveaux concepts pour l'équipe
- **Cohérence Éventuelle** : Délai entre écriture et lecture
- **Debugging** : Plus difficile de suivre le flux

### Risques
- Surengineering pour des domaines simples
- Incohérence entre command et query sides
- Performance dégradée par la complexité

### Mitigations
- **Formation** : Formation équipe sur DDD/CQRS
- **Implémentation Progressive** : Commencer par un bounded context
- **Monitoring** : Surveillance de la cohérence des données
- **Documentation** : Documentation détaillée des domaines
- **Tests** : Tests complets des command et query handlers

## Implémentation

### Structure des Packages
```
src/main/java/com/banque/dpj/
├── domain/
│   ├── document/
│   │   ├── model/          # Aggregates, Entities, Value Objects
│   │   ├── repository/     # Domain repositories
│   │   └── service/        # Domain services
│   ├── dossier/
│   └── user/
├── application/
│   ├── command/            # Command handlers
│   ├── query/              # Query handlers
│   └── service/            # Application services
├── infrastructure/
│   ├── persistence/        # JPA repositories
│   ├── messaging/          # Event publishing
│   └── web/               # REST controllers
└── shared/
    ├── event/             # Domain events
    └── common/            # Shared kernel
```

### Configuration Axon Framework
```java
@Configuration
public class AxonConfig {
    
    @Bean
    public EventStore eventStore(DataSource dataSource) {
        return JdbcEventStore.builder()
            .connectionProvider(dataSource::getConnection)
            .build();
    }
    
    @Bean
    public CommandBus commandBus() {
        return SimpleCommandBus.builder().build();
    }
    
    @Bean
    public QueryBus queryBus() {
        return SimpleQueryBus.builder().build();
    }
}
```

Cette approche DDD + CQRS permet de gérer la complexité métier tout en optimisant les performances et en maintenant une architecture évolutive.