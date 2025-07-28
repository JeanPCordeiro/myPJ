# Patterns et Bonnes Pratiques

## 1. Patterns Architecturaux

### 1.1 Microservices Patterns

#### Domain-Driven Design (DDD)
```java
// Exemple d'Aggregate Root pour le domaine Dossier
@Entity
@AggregateRoot
public class Dossier {
    @Id
    private DossierId id;
    private ClientId clientId;
    private TypeParcours typeParcours;
    private StatutDossier statut;
    private List<EtapeParcours> etapes;
    private List<Document> documents;
    
    // Méthodes métier encapsulant la logique business
    public void ajouterDocument(Document document) {
        validateDocumentCanBeAdded(document);
        documents.add(document);
        publishEvent(new DocumentAjouteEvent(this.id, document.getId()));
    }
    
    public void validerEtape(String codeEtape) {
        EtapeParcours etape = findEtapeByCode(codeEtape);
        etape.valider();
        
        if (toutesEtapesCompletes()) {
            this.statut = StatutDossier.COMPLET;
            publishEvent(new DossierCompletEvent(this.id));
        }
    }
    
    private void validateDocumentCanBeAdded(Document document) {
        if (this.statut == StatutDossier.ARCHIVE) {
            throw new DomainException("Impossible d'ajouter un document à un dossier archivé");
        }
        // Autres validations métier...
    }
}
```

#### Event Sourcing Pattern
```java
// Event Store pour la traçabilité complète
@Component
public class DossierEventStore {
    
    private final EventRepository eventRepository;
    private final EventPublisher eventPublisher;
    
    public void saveEvents(AggregateId aggregateId, List<DomainEvent> events, int expectedVersion) {
        // Vérification de la version pour éviter les conflits
        int currentVersion = getCurrentVersion(aggregateId);
        if (currentVersion != expectedVersion) {
            throw new ConcurrencyException("Version conflict detected");
        }
        
        // Sauvegarde des événements
        for (DomainEvent event : events) {
            EventData eventData = EventData.builder()
                .aggregateId(aggregateId)
                .eventType(event.getClass().getSimpleName())
                .eventData(serialize(event))
                .version(++currentVersion)
                .timestamp(Instant.now())
                .build();
                
            eventRepository.save(eventData);
            eventPublisher.publish(event);
        }
    }
    
    public List<DomainEvent> getEvents(AggregateId aggregateId) {
        return eventRepository.findByAggregateIdOrderByVersion(aggregateId)
            .stream()
            .map(this::deserialize)
            .collect(Collectors.toList());
    }
}
```

#### CQRS (Command Query Responsibility Segregation)
```java
// Séparation des commandes et des requêtes
@Component
public class DossierCommandHandler {
    
    private final DossierRepository dossierRepository;
    private final EventStore eventStore;
    
    @CommandHandler
    public void handle(CreerDossierCommand command) {
        // Validation des règles métier
        validateCreerDossierCommand(command);
        
        // Création de l'agrégat
        Dossier dossier = Dossier.creer(
            command.getClientId(),
            command.getTypeParcours(),
            command.getCollaborateurId()
        );
        
        // Sauvegarde
        dossierRepository.save(dossier);
        
        // Publication des événements
        eventStore.saveEvents(dossier.getId(), dossier.getUncommittedEvents(), 0);
    }
}

@Component
public class DossierQueryHandler {
    
    private final DossierReadModelRepository readModelRepository;
    
    @QueryHandler
    public DossierView handle(GetDossierQuery query) {
        return readModelRepository.findById(query.getDossierId())
            .orElseThrow(() -> new DossierNotFoundException(query.getDossierId()));
    }
    
    @QueryHandler
    public List<DossierSummaryView> handle(GetDossiersParClientQuery query) {
        return readModelRepository.findByClientIdAndStatut(
            query.getClientId(),
            query.getStatut(),
            query.getPageable()
        );
    }
}
```

### 1.2 Integration Patterns

#### Saga Pattern pour les Transactions Distribuées
```java
// Orchestration Saga pour le processus d'archivage
@Component
public class ArchivageSaga {
    
    private final DocumentService documentService;
    private final GedService gedService;
    private final NotificationService notificationService;
    
    @SagaOrchestrationStart
    public void handle(DemandeArchivageEvent event) {
        SagaTransaction saga = SagaTransaction.builder()
            .sagaId(UUID.randomUUID())
            .dossierId(event.getDossierId())
            .etape(EtapeSaga.VALIDATION_DOCUMENTS)
            .build();
            
        // Étape 1: Validation des documents
        documentService.validerDocumentsPourArchivage(event.getDossierId())
            .thenCompose(result -> {
                if (result.isSuccess()) {
                    saga.setEtape(EtapeSaga.ARCHIVAGE_GED);
                    return gedService.archiverDocuments(event.getDossierId());
                } else {
                    return CompletableFuture.completedFuture(
                        ArchivageResult.failure("Validation échouée")
                    );
                }
            })
            .thenCompose(result -> {
                if (result.isSuccess()) {
                    saga.setEtape(EtapeSaga.NOTIFICATION);
                    return notificationService.envoyerNotificationArchivage(event.getDossierId());
                } else {
                    // Compensation: annuler l'archivage
                    return compenserArchivage(saga);
                }
            })
            .thenAccept(result -> {
                if (result.isSuccess()) {
                    saga.setStatut(StatutSaga.COMPLETE);
                } else {
                    saga.setStatut(StatutSaga.FAILED);
                }
                sagaRepository.save(saga);
            });
    }
    
    private CompletableFuture<ArchivageResult> compenserArchivage(SagaTransaction saga) {
        // Actions de compensation en ordre inverse
        return gedService.annulerArchivage(saga.getDossierId())
            .thenCompose(result -> 
                notificationService.envoyerNotificationEchec(saga.getDossierId())
            );
    }
}
```

#### Circuit Breaker Pattern
```java
// Protection contre les pannes en cascade
@Component
public class ResilientGedService {
    
    private final CircuitBreaker circuitBreaker;
    private final GedApiClient gedApiClient;
    private final FallbackService fallbackService;
    
    public ResilientGedService() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("ged-service");
        
        // Configuration du circuit breaker
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                log.info("Circuit breaker transition: {} -> {}", 
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState()));
    }
    
    public CompletableFuture<ArchivageResult> archiverDocument(String documentId) {
        Supplier<CompletableFuture<ArchivageResult>> decoratedSupplier = 
            CircuitBreaker.decorateSupplier(circuitBreaker, () -> 
                gedApiClient.archiverDocument(documentId)
            );
        
        return decoratedSupplier.get()
            .exceptionally(throwable -> {
                log.warn("Archivage GED échoué, utilisation du fallback", throwable);
                return fallbackService.planifierArchivageDiffere(documentId);
            });
    }
}
```

### 1.3 Data Patterns

#### Repository Pattern avec Spécifications
```java
// Repository avec spécifications pour des requêtes complexes
public interface DossierRepository extends JpaRepository<Dossier, UUID> {
    
    default List<Dossier> findBySpecification(Specification<Dossier> spec, Pageable pageable) {
        return findAll(spec, pageable).getContent();
    }
}

// Spécifications réutilisables
public class DossierSpecifications {
    
    public static Specification<Dossier> parClient(String clientId) {
        return (root, query, criteriaBuilder) ->
            criteriaBuilder.equal(root.get("clientId"), clientId);
    }
    
    public static Specification<Dossier> parStatut(StatutDossier statut) {
        return (root, query, criteriaBuilder) ->
            criteriaBuilder.equal(root.get("statut"), statut);
    }
    
    public static Specification<Dossier> parPeriode(LocalDate debut, LocalDate fin) {
        return (root, query, criteriaBuilder) ->
            criteriaBuilder.between(root.get("dateCreation"), debut, fin);
    }
    
    public static Specification<Dossier> avecDocumentsManquants() {
        return (root, query, criteriaBuilder) -> {
            Subquery<Long> subquery = query.subquery(Long.class);
            Root<Document> documentRoot = subquery.from(Document.class);
            
            subquery.select(criteriaBuilder.count(documentRoot))
                .where(
                    criteriaBuilder.equal(documentRoot.get("dossier"), root),
                    criteriaBuilder.equal(documentRoot.get("estObligatoire"), true),
                    criteriaBuilder.notEqual(documentRoot.get("statut"), StatutDocument.VALIDE)
                );
                
            return criteriaBuilder.greaterThan(subquery, 0L);
        };
    }
}

// Usage
@Service
public class DossierQueryService {
    
    public List<Dossier> rechercherDossiers(DossierSearchCriteria criteria) {
        Specification<Dossier> spec = Specification.where(null);
        
        if (criteria.getClientId() != null) {
            spec = spec.and(DossierSpecifications.parClient(criteria.getClientId()));
        }
        
        if (criteria.getStatut() != null) {
            spec = spec.and(DossierSpecifications.parStatut(criteria.getStatut()));
        }
        
        if (criteria.isSeulementAvecDocumentsManquants()) {
            spec = spec.and(DossierSpecifications.avecDocumentsManquants());
        }
        
        return dossierRepository.findBySpecification(spec, criteria.getPageable());
    }
}
```

#### Unit of Work Pattern
```java
// Gestion transactionnelle cohérente
@Component
public class UnitOfWork {
    
    private final Map<Class<?>, Repository<?, ?>> repositories = new HashMap<>();
    private final List<DomainEvent> events = new ArrayList<>();
    private final EntityManager entityManager;
    
    public <T, ID> Repository<T, ID> getRepository(Class<T> entityClass) {
        return (Repository<T, ID>) repositories.computeIfAbsent(entityClass, 
            clazz -> createRepository(clazz));
    }
    
    public void registerEvent(DomainEvent event) {
        events.add(event);
    }
    
    @Transactional
    public void commit() {
        try {
            // Flush toutes les modifications
            entityManager.flush();
            
            // Publier les événements après le commit
            events.forEach(eventPublisher::publish);
            
        } finally {
            // Nettoyer l'état
            events.clear();
        }
    }
    
    public void rollback() {
        entityManager.clear();
        events.clear();
    }
}
```

## 2. Patterns de Sécurité

### 2.1 Authentication & Authorization Patterns

#### JWT Token Pattern avec Refresh
```java
@Service
public class JwtTokenService {
    
    private final JwtProperties jwtProperties;
    private final RedisTemplate<String, Object> redisTemplate;
    
    public TokenPair generateTokens(UserDetails userDetails) {
        // Access Token (courte durée)
        String accessToken = Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(jwtProperties.getAccessTokenExpiry())))
            .claim("role", userDetails.getAuthorities())
            .claim("permissions", extractPermissions(userDetails))
            .signWith(getSigningKey(), SignatureAlgorithm.RS256)
            .compact();
        
        // Refresh Token (longue durée)
        String refreshToken = Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(jwtProperties.getRefreshTokenExpiry())))
            .claim("type", "refresh")
            .signWith(getSigningKey(), SignatureAlgorithm.RS256)
            .compact();
        
        // Stocker le refresh token en Redis avec TTL
        String key = "refresh_token:" + userDetails.getUsername();
        redisTemplate.opsForValue().set(key, refreshToken, jwtProperties.getRefreshTokenExpiry());
        
        return new TokenPair(accessToken, refreshToken);
    }
    
    public TokenPair refreshTokens(String refreshToken) {
        // Valider le refresh token
        Claims claims = validateToken(refreshToken);
        
        if (!"refresh".equals(claims.get("type"))) {
            throw new InvalidTokenException("Token type invalide");
        }
        
        // Vérifier que le token existe en Redis
        String key = "refresh_token:" + claims.getSubject();
        String storedToken = (String) redisTemplate.opsForValue().get(key);
        
        if (!refreshToken.equals(storedToken)) {
            throw new InvalidTokenException("Refresh token invalide");
        }
        
        // Générer de nouveaux tokens
        UserDetails userDetails = loadUserByUsername(claims.getSubject());
        return generateTokens(userDetails);
    }
}
```

#### Role-Based Access Control (RBAC)
```java
// Annotation personnalisée pour l'autorisation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("@permissionEvaluator.hasPermission(authentication, #targetDomainObject, #permission)")
public @interface RequirePermission {
    String value(); // Permission requise
    String targetParam() default ""; // Paramètre contenant l'objet cible
}

// Évaluateur de permissions personnalisé
@Component
public class DpjPermissionEvaluator implements PermissionEvaluator {
    
    @Override
    public boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission) {
        if (authentication == null || !authentication.isAuthenticated()) {
            return false;
        }
        
        UserPrincipal user = (UserPrincipal) authentication.getPrincipal();
        String permissionStr = permission.toString();
        
        // Vérification des permissions de base
        if (!user.hasPermission(permissionStr)) {
            return false;
        }
        
        // Vérification contextuelle selon l'objet
        return evaluateContextualPermission(user, targetDomainObject, permissionStr);
    }
    
    private boolean evaluateContextualPermission(UserPrincipal user, Object target, String permission) {
        if (target instanceof Dossier) {
            Dossier dossier = (Dossier) target;
            
            switch (user.getRole()) {
                case CLIENT:
                    return dossier.getClientId().equals(user.getId());
                case COLLABORATEUR:
                    return dossier.getCollaborateurId().equals(user.getId()) ||
                           user.getClientIds().contains(dossier.getClientId());
                case SUPERVISEUR:
                    return user.getAgenceId().equals(dossier.getAgenceId());
                case ADMIN:
                    return true;
                default:
                    return false;
            }
        }
        
        return true; // Autoriser par défaut pour les autres objets
    }
}

// Usage dans les contrôleurs
@RestController
public class DossierController {
    
    @GetMapping("/dossiers/{id}")
    @RequirePermission(value = "dossier:read", targetParam = "dossier")
    public ResponseEntity<DossierDto> getDossier(@PathVariable String id) {
        Dossier dossier = dossierService.findById(id);
        return ResponseEntity.ok(dossierMapper.toDto(dossier));
    }
}
```

### 2.2 Data Protection Patterns

#### Encryption at Rest Pattern
```java
// Chiffrement transparent des données sensibles
@Component
public class FieldEncryptionService {
    
    private final AESUtil aesUtil;
    private final KeyManagementService keyService;
    
    @EventListener
    @Async
    public void handleEntityPrePersist(EntityPrePersistEvent event) {
        Object entity = event.getEntity();
        encryptSensitiveFields(entity);
    }
    
    @EventListener
    @Async
    public void handleEntityPostLoad(EntityPostLoadEvent event) {
        Object entity = event.getEntity();
        decryptSensitiveFields(entity);
    }
    
    private void encryptSensitiveFields(Object entity) {
        Field[] fields = entity.getClass().getDeclaredFields();
        
        for (Field field : fields) {
            if (field.isAnnotationPresent(Encrypted.class)) {
                try {
                    field.setAccessible(true);
                    String value = (String) field.get(entity);
                    
                    if (value != null) {
                        String encryptedValue = aesUtil.encrypt(value, getEncryptionKey(entity));
                        field.set(entity, encryptedValue);
                    }
                } catch (Exception e) {
                    throw new EncryptionException("Erreur chiffrement champ " + field.getName(), e);
                }
            }
        }
    }
    
    private SecretKey getEncryptionKey(Object entity) {
        // Génération d'une clé spécifique à l'entité
        String keyId = entity.getClass().getSimpleName() + "_encryption_key";
        return keyService.getOrCreateKey(keyId);
    }
}

// Annotation pour marquer les champs sensibles
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Encrypted {
    String algorithm() default "AES-256-GCM";
}

// Usage dans les entités
@Entity
public class Document {
    @Id
    private String id;
    
    @Encrypted
    private String nomOriginal; // Chiffré automatiquement
    
    private String mimeType; // Non chiffré
    
    // getters/setters
}
```

## 3. Patterns de Performance

### 3.1 Caching Patterns

#### Multi-Level Caching Strategy
```java
@Service
public class DocumentCacheService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final CaffeineCache localCache;
    private final DocumentRepository documentRepository;
    
    @Cacheable(value = "documents", key = "#documentId", unless = "#result == null")
    public Document getDocument(String documentId) {
        // Niveau 1: Cache local (Caffeine)
        Document document = localCache.get(documentId);
        if (document != null) {
            return document;
        }
        
        // Niveau 2: Cache distribué (Redis)
        document = (Document) redisTemplate.opsForValue().get("doc:" + documentId);
        if (document != null) {
            localCache.put(documentId, document);
            return document;
        }
        
        // Niveau 3: Base de données
        document = documentRepository.findById(documentId).orElse(null);
        if (document != null) {
            // Mise en cache avec TTL différencié
            redisTemplate.opsForValue().set("doc:" + documentId, document, Duration.ofMinutes(30));
            localCache.put(documentId, document);
        }
        
        return document;
    }
    
    @CacheEvict(value = "documents", key = "#documentId")
    public void evictDocument(String documentId) {
        localCache.invalidate(documentId);
        redisTemplate.delete("doc:" + documentId);
    }
    
    // Cache warming pour les documents fréquemment accédés
    @Scheduled(fixedRate = 300000) // 5 minutes
    public void warmupCache() {
        List<String> frequentDocuments = documentRepository.findMostAccessedDocuments(100);
        
        frequentDocuments.parallelStream().forEach(documentId -> {
            if (!localCache.containsKey(documentId)) {
                getDocument(documentId); // Charge en cache
            }
        });
    }
}
```

#### Cache-Aside Pattern avec Write-Behind
```java
@Service
public class DocumentMetadataCache {
    
    private final RedisTemplate<String, DocumentMetadata> redisTemplate;
    private final DocumentMetadataRepository repository;
    private final AsyncTaskExecutor asyncExecutor;
    
    public DocumentMetadata getMetadata(String documentId) {
        // Lecture depuis le cache
        String cacheKey = "metadata:" + documentId;
        DocumentMetadata metadata = redisTemplate.opsForValue().get(cacheKey);
        
        if (metadata == null) {
            // Cache miss - lecture depuis la DB
            metadata = repository.findById(documentId).orElse(null);
            
            if (metadata != null) {
                // Mise en cache
                redisTemplate.opsForValue().set(cacheKey, metadata, Duration.ofHours(1));
            }
        }
        
        return metadata;
    }
    
    public void updateMetadata(String documentId, DocumentMetadata metadata) {
        // Mise à jour immédiate du cache
        String cacheKey = "metadata:" + documentId;
        redisTemplate.opsForValue().set(cacheKey, metadata, Duration.ofHours(1));
        
        // Écriture asynchrone en base (Write-Behind)
        asyncExecutor.execute(() -> {
            try {
                repository.save(metadata);
            } catch (Exception e) {
                log.error("Erreur écriture asynchrone métadonnées {}", documentId, e);
                // En cas d'erreur, invalider le cache pour forcer la relecture
                redisTemplate.delete(cacheKey);
            }
        });
    }
}
```

### 3.2 Database Optimization Patterns

#### Connection Pool Optimization
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/dpj_db");
        config.setUsername("dpj_user");
        config.setPassword("password");
        
        // Optimisations du pool de connexions
        config.setMaximumPoolSize(20); // Basé sur le nombre de threads
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000); // 30 secondes
        config.setIdleTimeout(600000); // 10 minutes
        config.setMaxLifetime(1800000); // 30 minutes
        config.setLeakDetectionThreshold(60000); // 1 minute
        
        // Optimisations PostgreSQL
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public DataSource readOnlyDataSource() {
        // Configuration similaire pour les read replicas
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5433/dpj_db");
        // Configuration optimisée pour la lecture
        config.setReadOnly(true);
        config.setMaximumPoolSize(15);
        
        return new HikariDataSource(config);
    }
}
```

#### Query Optimization Pattern
```java
// Optimisation des requêtes avec projections
public interface DossierProjection {
    String getId();
    String getNumeroDossier();
    String getClientId();
    StatutDossier getStatut();
    LocalDateTime getDateCreation();
    Long getDocumentCount();
}

@Repository
public interface DossierRepository extends JpaRepository<Dossier, String> {
    
    // Requête optimisée avec projection pour les listes
    @Query("""
        SELECT d.id as id, 
               d.numeroDossier as numeroDossier,
               d.clientId as clientId,
               d.statut as statut,
               d.dateCreation as dateCreation,
               COUNT(doc.id) as documentCount
        FROM Dossier d 
        LEFT JOIN d.documents doc 
        WHERE d.clientId = :clientId 
        GROUP BY d.id, d.numeroDossier, d.clientId, d.statut, d.dateCreation
        ORDER BY d.dateCreation DESC
        """)
    Page<DossierProjection> findDossierSummaryByClientId(
        @Param("clientId") String clientId, 
        Pageable pageable
    );
    
    // Requête avec fetch join pour éviter N+1
    @Query("""
        SELECT DISTINCT d 
        FROM Dossier d 
        LEFT JOIN FETCH d.etapes e 
        LEFT JOIN FETCH d.documents doc 
        WHERE d.id = :dossierId
        """)
    Optional<Dossier> findByIdWithDetails(@Param("dossierId") String dossierId);
    
    // Requête batch pour optimiser les mises à jour
    @Modifying
    @Query("UPDATE Dossier d SET d.statut = :statut WHERE d.id IN :dossierIds")
    int updateStatutBatch(@Param("statut") StatutDossier statut, @Param("dossierIds") List<String> dossierIds);
}
```

## 4. Patterns de Monitoring et Observabilité

### 4.1 Distributed Tracing Pattern

#### Custom Tracing avec Micrometer
```java
@Component
public class DocumentProcessingTracer {
    
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;
    
    public void traceDocumentProcessing(String documentId, Runnable processing) {
        Span span = tracer.nextSpan()
            .name("document.processing")
            .tag("document.id", documentId)
            .tag("service", "document-service")
            .start();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            processing.run();
            span.tag("status", "success");
        } catch (Exception e) {
            span.tag("status", "error");
            span.tag("error.message", e.getMessage());
            throw e;
        } finally {
            sample.stop(Timer.builder("document.processing.duration")
                .tag("document.id", documentId)
                .register(meterRegistry));
            span.end();
        }
    }
    
    // Trace automatique des méthodes avec annotation
    @TraceAsync
    @Async
    public CompletableFuture<ProcessingResult> processDocumentAsync(String documentId) {
        return CompletableFuture.supplyAsync(() -> {
            // Traitement du document
            return new ProcessingResult(documentId, "success");
        });
    }
}

// Aspect pour le tracing automatique
@Aspect
@Component
public class TracingAspect {
    
    private final Tracer tracer;
    
    @Around("@annotation(traceAsync)")
    public Object traceAsyncMethod(ProceedingJoinPoint joinPoint, TraceAsync traceAsync) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        Span span = tracer.nextSpan()
            .name(className + "." + methodName)
            .tag("method", methodName)
            .tag("class", className)
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            Object result = joinPoint.proceed();
            
            if (result instanceof CompletableFuture) {
                CompletableFuture<?> future = (CompletableFuture<?>) result;
                return future.whenComplete((res, ex) -> {
                    if (ex != null) {
                        span.tag("error", ex.getMessage());
                    }
                    span.end();
                });
            } else {
                span.end();
                return result;
            }
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            span.end();
            throw e;
        }
    }
}
```

### 4.2 Metrics Collection Pattern

#### Business Metrics avec Micrometer
```java
@Component
public class BusinessMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final Counter documentUploadCounter;
    private final Timer documentProcessingTimer;
    private final Gauge activeUsersGauge;
    
    public BusinessMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.documentUploadCounter = Counter.builder("dpj.documents.uploaded")
            .description("Nombre de documents uploadés")
            .register(meterRegistry);
            
        this.documentProcessingTimer = Timer.builder("dpj.documents.processing.duration")
            .description("Durée de traitement des documents")
            .register(meterRegistry);
            
        this.activeUsersGauge = Gauge.builder("dpj.users.active")
            .description("Nombre d'utilisateurs actifs")
            .register(meterRegistry, this, BusinessMetricsCollector::getActiveUsersCount);
    }
    
    public void recordDocumentUpload(String documentType, String clientType) {
        documentUploadCounter.increment(
            Tags.of(
                "document.type", documentType,
                "client.type", clientType
            )
        );
    
}
    
    public void recordDocumentProcessingTime(String documentId, Duration duration) {
        documentProcessingTimer.record(duration, 
            Tags.of("document.id", documentId)
        );
    }
    
    private double getActiveUsersCount() {
        // Logique pour compter les utilisateurs actifs
        return sessionService.getActiveSessionCount();
    }
    
    // Métriques personnalisées pour le business
    @EventListener
    public void handleDossierCompleted(DossierCompletedEvent event) {
        meterRegistry.counter("dpj.dossiers.completed",
            "type", event.getTypeParcours(),
            "agence", event.getAgenceId()
        ).increment();
    }
    
    @EventListener
    public void handleDocumentValidated(DocumentValidatedEvent event) {
        meterRegistry.counter("dpj.documents.validated",
            "type", event.getDocumentType(),
            "validator", event.getValidatorId()
        ).increment();
    }
}
```

## 5. Patterns de Résilience

### 5.1 Retry Pattern avec Backoff Exponentiel

```java
@Component
public class ResilientServiceClient {
    
    private final RetryTemplate retryTemplate;
    private final CircuitBreaker circuitBreaker;
    
    public ResilientServiceClient() {
        // Configuration du retry avec backoff exponentiel
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(1000, 2, 10000) // 1s, 2s, 4s, max 10s
            .retryOn(ConnectException.class, SocketTimeoutException.class, TransientException.class)
            .build();
            
        // Configuration du circuit breaker
        this.circuitBreaker = CircuitBreaker.ofDefaults("external-service");
    }
    
    public <T> T executeWithResilience(String operation, Supplier<T> supplier) {
        return retryTemplate.execute(context -> {
            log.debug("Tentative {} pour l'opération: {}", 
                     context.getRetryCount() + 1, operation);
            
            return circuitBreaker.executeSupplier(supplier);
        });
    }
    
    // Exemple d'utilisation
    public GedArchiveResult archiveToGed(String documentId) {
        return executeWithResilience("archive-to-ged", () -> {
            return gedApiClient.archive(documentId);
        });
    }
}
```

### 5.2 Bulkhead Pattern

```java
// Isolation des ressources par domaine métier
@Configuration
public class ThreadPoolConfig {
    
    @Bean("documentProcessingExecutor")
    public TaskExecutor documentProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("doc-processing-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean("gedIntegrationExecutor")
    public TaskExecutor gedIntegrationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(6);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("ged-integration-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean("notificationExecutor")
    public TaskExecutor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("notification-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// Usage avec isolation
@Service
public class DocumentProcessingService {
    
    @Async("documentProcessingExecutor")
    public CompletableFuture<ProcessingResult> processDocument(String documentId) {
        // Traitement isolé dans son propre pool de threads
        return CompletableFuture.completedFuture(
            performDocumentProcessing(documentId)
        );
    }
}

@Service
public class GedIntegrationService {
    
    @Async("gedIntegrationExecutor")
    public CompletableFuture<ArchiveResult> archiveDocument(String documentId) {
        // Intégration GED isolée
        return CompletableFuture.completedFuture(
            performGedArchiving(documentId)
        );
    }
}
```

## 6. Patterns de Testing

### 6.1 Test Containers Pattern

```java
// Tests d'intégration avec containers
@SpringBootTest
@Testcontainers
class DocumentServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("test_db")
            .withUsername("test_user")
            .withPassword("test_password");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @Container
    static MinIOContainer minio = new MinIOContainer("minio/minio:latest")
            .withUserName("minioadmin")
            .withPassword("minioadmin");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
        
        registry.add("minio.endpoint", minio::getS3URL);
        registry.add("minio.access-key", minio::getUserName);
        registry.add("minio.secret-key", minio::getPassword);
    }
    
    @Autowired
    private DocumentService documentService;
    
    @Test
    void shouldUploadAndRetrieveDocument() {
        // Given
        MockMultipartFile file = new MockMultipartFile(
            "file", "test.pdf", "application/pdf", "test content".getBytes()
        );
        
        DocumentUploadRequest request = DocumentUploadRequest.builder()
            .dossierId("test-dossier")
            .typeDocument("PIECE_IDENTITE")
            .build();
        
        // When
        Document document = documentService.uploadDocument(request, file);
        
        // Then
        assertThat(document).isNotNull();
        assertThat(document.getNomFichier()).isEqualTo("test.pdf");
        assertThat(document.getStatut()).isEqualTo(StatutDocument.EN_ATTENTE);
        
        // Verify file is stored
        byte[] retrievedContent = documentService.getDocumentContent(document.getId());
        assertThat(retrievedContent).isEqualTo("test content".getBytes());
    }
}
```

### 6.2 Contract Testing Pattern

```java
// Tests de contrat avec Pact
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "ged-service", port = "8080")
class GedServiceContractTest {
    
    @Pact(consumer = "document-service")
    public RequestResponsePact archiveDocumentPact(PactDslWithProvider builder) {
        return builder
            .given("document exists and is valid")
            .uponReceiving("archive document request")
            .path("/api/v1/ged/archive")
            .method("POST")
            .body(new PactDslJsonBody()
                .stringType("documentId", "doc-123")
                .stringType("documentType", "PIECE_IDENTITE")
                .stringType("clientId", "client-456"))
            .willRespondWith()
            .status(201)
            .body(new PactDslJsonBody()
                .stringType("gedId", "GED-2024-001")
                .datetime("archiveDate", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
                .stringType("status", "ARCHIVED"))
            .toPact();
    }
    
    @Test
    @PactTestFor(pactMethod = "archiveDocumentPact")
    void shouldArchiveDocumentSuccessfully() {
        // Given
        GedArchiveRequest request = GedArchiveRequest.builder()
            .documentId("doc-123")
            .documentType("PIECE_IDENTITE")
            .clientId("client-456")
            .build();
        
        // When
        GedArchiveResponse response = gedServiceClient.archiveDocument(request);
        
        // Then
        assertThat(response.getGedId()).isEqualTo("GED-2024-001");
        assertThat(response.getStatus()).isEqualTo("ARCHIVED");
    }
}
```

### 6.3 Architecture Testing Pattern

```java
// Tests d'architecture avec ArchUnit
class ArchitectureTest {
    
    private final JavaClasses classes = ClassFileImporter()
        .withImportOption(ImportOption.Predefined.DO_NOT_INCLUDE_TESTS)
        .importPackages("com.banque.dpj");
    
    @Test
    void servicesShouldNotDependOnControllers() {
        noClasses()
            .that().resideInAPackage("..service..")
            .should().dependOnClassesThat()
            .resideInAPackage("..controller..")
            .check(classes);
    }
    
    @Test
    void controllersShouldOnlyBeAccessedByControllersPackage() {
        classes()
            .that().resideInAPackage("..controller..")
            .should().onlyBeAccessed().byClassesThat()
            .resideInAnyPackage("..controller..", "..config..")
            .check(classes);
    }
    
    @Test
    void repositoriesShouldOnlyBeAccessedByServices() {
        classes()
            .that().resideInAPackage("..repository..")
            .should().onlyBeAccessed().byClassesThat()
            .resideInAnyPackage("..service..", "..config..")
            .check(classes);
    }
    
    @Test
    void serviceClassesShouldBeAnnotatedWithService() {
        classes()
            .that().resideInAPackage("..service..")
            .and().areNotInterfaces()
            .should().beAnnotatedWith(Service.class)
            .check(classes);
    }
    
    @Test
    void noClassShouldUseJavaUtilLogging() {
        noClasses()
            .should().accessClassesThat()
            .resideInAPackage("java.util.logging..")
            .because("Use SLF4J instead")
            .check(classes);
    }
}
```

## 7. Bonnes Pratiques de Développement

### 7.1 Clean Code Practices

#### Value Objects Pattern
```java
// Value Objects pour l'encapsulation des données métier
@Value
@Builder
public class DocumentId {
    private final String value;
    
    public static DocumentId of(String value) {
        if (value == null || value.trim().isEmpty()) {
            throw new IllegalArgumentException("DocumentId ne peut pas être vide");
        }
        return new DocumentId(value);
    }
    
    public static DocumentId generate() {
        return new DocumentId(UUID.randomUUID().toString());
    }
}

@Value
@Builder
public class FileSize {
    private final long bytes;
    
    public static FileSize of(long bytes) {
        if (bytes < 0) {
            throw new IllegalArgumentException("La taille ne peut pas être négative");
        }
        return new FileSize(bytes);
    }
    
    public boolean exceedsLimit(FileSize limit) {
        return this.bytes > limit.bytes;
    }
    
    public String toHumanReadable() {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        if (bytes < 1024 * 1024 * 1024) return String.format("%.1f MB", bytes / (1024.0 * 1024));
        return String.format("%.1f GB", bytes / (1024.0 * 1024 * 1024));
    }
}
```

#### Domain Services Pattern
```java
// Services métier encapsulant la logique complexe
@Service
public class DocumentValidationService {
    
    private final List<DocumentValidator> validators;
    
    public ValidationResult validate(Document document) {
        ValidationResult result = ValidationResult.success();
        
        for (DocumentValidator validator : validators) {
            if (validator.supports(document.getTypeDocument())) {
                ValidationResult validationResult = validator.validate(document);
                result = result.merge(validationResult);
                
                if (result.hasErrors() && validator.isBlocking()) {
                    break; // Arrêter à la première erreur bloquante
                }
            }
        }
        
        return result;
    }
}

// Interface pour les validateurs spécialisés
public interface DocumentValidator {
    boolean supports(String documentType);
    ValidationResult validate(Document document);
    boolean isBlocking();
}

@Component
public class PieceIdentiteValidator implements DocumentValidator {
    
    @Override
    public boolean supports(String documentType) {
        return "PIECE_IDENTITE".equals(documentType);
    }
    
    @Override
    public ValidationResult validate(Document document) {
        ValidationResult result = ValidationResult.success();
        
        // Validation de la taille
        if (document.getTailleFichier() > 5 * 1024 * 1024) { // 5MB
            result.addError("La pièce d'identité ne peut pas dépasser 5MB");
        }
        
        // Validation du format
        if (!Arrays.asList("image/jpeg", "image/png", "application/pdf")
                .contains(document.getMimeType())) {
            result.addError("Format non autorisé pour une pièce d'identité");
        }
        
        return result;
    }
    
    @Override
    public boolean isBlocking() {
        return true; // Erreurs bloquantes
    }
}
```

### 7.2 Error Handling Patterns

#### Global Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private final MessageSource messageSource;
    
    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationException(ValidationException ex, Locale locale) {
        return ErrorResponse.builder()
            .code("VALIDATION_ERROR")
            .message(getMessage("error.validation", locale))
            .details(ex.getValidationErrors())
            .timestamp(Instant.now())
            .build();
    }
    
    @ExceptionHandler(DocumentNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleDocumentNotFound(DocumentNotFoundException ex, Locale locale) {
        return ErrorResponse.builder()
            .code("DOCUMENT_NOT_FOUND")
            .message(getMessage("error.document.not.found", locale))
            .details(Map.of("documentId", ex.getDocumentId()))
            .timestamp(Instant.now())
            .build();
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex, Locale locale) {
        return ErrorResponse.builder()
            .code("ACCESS_DENIED")
            .message(getMessage("error.access.denied", locale))
            .timestamp(Instant.now())
            .build();
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex, Locale locale) {
        log.error("Erreur inattendue", ex);
        
        return ErrorResponse.builder()
            .code("INTERNAL_ERROR")
            .message(getMessage("error.internal", locale))
            .timestamp(Instant.now())
            .build();
    }
    
    private String getMessage(String key, Locale locale) {
        return messageSource.getMessage(key, null, locale);
    }
}
```

#### Result Pattern pour la Gestion d'Erreurs
```java
// Pattern Result pour éviter les exceptions dans la logique métier
@Value
@Builder
public class Result<T> {
    private final T value;
    private final boolean success;
    private final String errorMessage;
    private final List<String> errors;
    
    public static <T> Result<T> success(T value) {
        return Result.<T>builder()
            .value(value)
            .success(true)
            .errors(Collections.emptyList())
            .build();
    }
    
    public static <T> Result<T> failure(String errorMessage) {
        return Result.<T>builder()
            .success(false)
            .errorMessage(errorMessage)
            .errors(Collections.emptyList())
            .build();
    }
    
    public static <T> Result<T> failure(List<String> errors) {
        return Result.<T>builder()
            .success(false)
            .errors(errors)
            .build();
    }
    
    public <U> Result<U> map(Function<T, U> mapper) {
        if (success) {
            return Result.success(mapper.apply(value));
        } else {
            return Result.<U>builder()
                .success(false)
                .errorMessage(errorMessage)
                .errors(errors)
                .build();
        }
    }
    
    public <U> Result<U> flatMap(Function<T, Result<U>> mapper) {
        if (success) {
            return mapper.apply(value);
        } else {
            return Result.<U>builder()
                .success(false)
                .errorMessage(errorMessage)
                .errors(errors)
                .build();
        }
    }
}

// Usage dans les services
@Service
public class DocumentService {
    
    public Result<Document> uploadDocument(DocumentUploadRequest request, MultipartFile file) {
        // Validation
        Result<Void> validationResult = validateUploadRequest(request, file);
        if (!validationResult.isSuccess()) {
            return Result.failure(validationResult.getErrors());
        }
        
        // Traitement
        try {
            Document document = processUpload(request, file);
            return Result.success(document);
        } catch (StorageException e) {
            return Result.failure("Erreur lors du stockage: " + e.getMessage());
        }
    }
    
    private Result<Void> validateUploadRequest(DocumentUploadRequest request, MultipartFile file) {
        List<String> errors = new ArrayList<>();
        
        if (file.isEmpty()) {
            errors.add("Le fichier ne peut pas être vide");
        }
        
        if (file.getSize() > MAX_FILE_SIZE) {
            errors.add("Le fichier dépasse la taille maximale autorisée");
        }
        
        if (!ALLOWED_MIME_TYPES.contains(file.getContentType())) {
            errors.add("Type de fichier non autorisé");
        }
        
        return errors.isEmpty() ? Result.success(null) : Result.failure(errors);
    }
}
```

## 8. Patterns de Configuration

### 8.1 Configuration Externalisée

```java
// Configuration par environnement avec validation
@ConfigurationProperties(prefix = "dpj")
@Validated
@Data
public class DpjProperties {
    
    @Valid
    private Database database = new Database();
    
    @Valid
    private Storage storage = new Storage();
    
    @Valid
    private Security security = new Security();
    
    @Data
    public static class Database {
        @NotBlank
        private String url;
        
        @NotBlank
        private String username;
        
        @NotBlank
        private String password;
        
        @Min(1)
        @Max(100)
        private int maxPoolSize = 20;
        
        @Min(0)
        @Max(50)
        private int minIdle = 5;
    }
    
    @Data
    public static class Storage {
        @NotBlank
        private String endpoint;
        
        @NotBlank
        private String accessKey;
        
        @NotBlank
        private String secretKey;
        
        @NotBlank
        private String bucketName;
        
        @Min(1)
        @Max(1000)
        private long maxFileSize = 50; // MB
    }
    
    @Data
    public static class Security {
        @NotBlank
        private String jwtSecret;
        
        @Min(300) // 5 minutes minimum
        @Max(86400) // 24 heures maximum
        private long accessTokenExpiry = 3600; // 1 heure
        
        @Min(86400) // 1 jour minimum
        @Max(2592000) // 30 jours maximum
        private long refreshTokenExpiry = 604800; // 7 jours
    }
}
```

### 8.2 Feature Flags Pattern

```java
// Gestion des feature flags
@Component
public class FeatureToggleService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final DpjProperties properties;
    
    public boolean isFeatureEnabled(String featureName) {
        return isFeatureEnabled(featureName, null);
    }
    
    public boolean isFeatureEnabled(String featureName, String userId) {
        // Vérification en cache Redis d'abord
        String cacheKey = "feature:" + featureName;
        Boolean cachedValue = (Boolean) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedValue != null) {
            return cachedValue;
        }
        
        // Logique de feature toggle
        boolean enabled = evaluateFeature(featureName, userId);
        
        // Mise en cache avec TTL
        redisTemplate.opsForValue().set(cacheKey, enabled, Duration.ofMinutes(5));
        
        return enabled;
    }
    
    private boolean evaluateFeature(String featureName, String userId) {
        switch (featureName) {
            case "NEW_DOCUMENT_VALIDATION":
                return properties.getFeatures().isNewDocumentValidation();
            case "ADVANCED_SEARCH":
                return properties.getFeatures().isAdvancedSearch();
            case "BETA_FEATURES":
                return userId != null && isBetaUser(userId);
            default:
                return false;
        }
    }
    
    private boolean isBetaUser(String userId) {
        // Logique pour déterminer si l'utilisateur est un beta testeur
        return redisTemplate.opsForSet().isMember("beta_users", userId);
    }
}

// Usage avec annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FeatureToggle {
    String value();
    boolean defaultValue() default false;
}

@Aspect
@Component
public class FeatureToggleAspect {
    
    private final FeatureToggleService featureToggleService;
    
    @Around("@annotation(featureToggle)")
    public Object checkFeatureToggle(ProceedingJoinPoint joinPoint, FeatureToggle featureToggle) throws Throwable {
        String featureName = featureToggle.value();
        boolean defaultValue = featureToggle.defaultValue();
        
        if (featureToggleService.isFeatureEnabled(featureName)) {
            return joinPoint.proceed();
        } else {
            if (defaultValue) {
                return joinPoint.proceed();
            } else {
                throw new FeatureNotEnabledException("Feature " + featureName + " is not enabled");
            }
        }
    }
}
```

Ces patterns et bonnes pratiques constituent un guide complet pour développer et maintenir le système DPJ de manière robuste, scalable et maintenable, en respectant les standards de l'industrie bancaire.