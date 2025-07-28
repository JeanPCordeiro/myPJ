# ADR-0009: Messaging Asynchrone avec Apache Kafka

## Statut

Accepté

## Contexte

Le système DPJ nécessite une solution de messaging pour :
- Communication asynchrone entre microservices
- Intégration avec la GED existante
- Notifications en temps réel
- Audit et traçabilité des événements
- Gestion des pics de charge
- Résilience et tolérance aux pannes

Exigences :
- Haute disponibilité et durabilité des messages
- Scalabilité horizontale
- Ordre des messages garanti par partition
- Support des patterns Event Sourcing et CQRS

## Options Considérées

### Option 1: Apache Kafka
- **Avantages** : Performance élevée, durabilité, écosystème riche
- **Inconvénients** : Complexité opérationnelle, courbe d'apprentissage

### Option 2: Amazon SQS/SNS
- **Avantages** : Managé AWS, simplicité, intégration native
- **Inconvénients** : Vendor lock-in, limitations fonctionnelles

### Option 3: RabbitMQ
- **Avantages** : Simplicité, patterns messaging riches
- **Inconvénients** : Performance limitée, pas d'Event Sourcing natif

### Option 4: Redis Pub/Sub
- **Avantages** : Simplicité, performance
- **Inconvénients** : Pas de durabilité, pas de replay

## Décision

Nous adoptons **Apache Kafka** comme solution de messaging :

### Architecture Kafka
- **Cluster** : 3 brokers minimum en mode distribué
- **Replication Factor** : 3 pour haute disponibilité
- **Partitioning** : Stratégie par clé métier pour ordre garanti
- **Retention** : 7 jours par défaut, configurable par topic

### Topics et Organisation
```
dpj-events/
├── document-events          # Événements documents (upload, validation, etc.)
├── dossier-events          # Événements dossiers (création, completion, etc.)
├── user-events             # Événements utilisateurs (login, actions, etc.)
├── notification-events     # Événements notifications
├── ged-sync-requests       # Demandes synchronisation GED
├── ged-sync-responses      # Réponses synchronisation GED
├── audit-events            # Événements d'audit
└── dead-letter-queue       # Messages en erreur
```

### Patterns d'Usage

#### Event Sourcing
- **Event Store** : Kafka comme source de vérité
- **Snapshots** : Projections périodiques pour performance
- **Replay** : Reconstruction d'état à partir des événements

#### CQRS (Command Query Responsibility Segregation)
- **Commands** : Modifications via événements Kafka
- **Queries** : Lectures depuis projections optimisées
- **Projections** : Vues matérialisées mises à jour par événements

## Conséquences

### Positives
- **Performance** : Débit élevé (millions de messages/sec)
- **Durabilité** : Persistance sur disque avec réplication
- **Scalabilité** : Scaling horizontal par ajout de partitions
- **Ordre** : Garantie d'ordre par partition
- **Replay** : Possibilité de rejouer les événements
- **Écosystème** : Kafka Connect, Kafka Streams, Schema Registry

### Négatives
- **Complexité** : Configuration et tuning complexes
- **Opérations** : Monitoring et maintenance spécialisés
- **Latence** : Latence plus élevée que solutions in-memory
- **Ressources** : Consommation mémoire et disque importante

### Risques
- Perte de messages en cas de mauvaise configuration
- Hotspots sur certaines partitions
- Complexité de gestion des schémas

### Mitigations
- **Configuration robuste** : Replication factor 3, acks=all
- **Monitoring** : Métriques détaillées avec Prometheus
- **Schema Registry** : Gestion centralisée des schémas Avro
- **Partitioning Strategy** : Clés de partition équilibrées
- **Backup** : Sauvegarde périodique des topics critiques

## Configuration

### Cluster Kafka
```yaml
# Configuration Broker
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Réplication et durabilité
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Rétention
log.retention.hours=168  # 7 jours
log.segment.bytes=1073741824  # 1GB
log.retention.check.interval.ms=300000
```

### Topics Configuration
```yaml
document-events:
  partitions: 6
  replication-factor: 3
  retention-ms: 604800000  # 7 jours
  cleanup-policy: delete

dossier-events:
  partitions: 3
  replication-factor: 3
  retention-ms: 2592000000  # 30 jours
  cleanup-policy: delete

audit-events:
  partitions: 12
  replication-factor: 3
  retention-ms: 31536000000  # 1 an
  cleanup-policy: delete
```

## Implémentation

### Producer Configuration
```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Durabilité et performance
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

### Consumer Configuration
```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "dpj-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        
        // Fiabilité
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

### Event Publishing
```java
@Service
public class EventPublisher {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public void publishDocumentEvent(DocumentEvent event) {
        String topic = "document-events";
        String key = event.getDocumentId(); // Partitioning par document
        
        kafkaTemplate.send(topic, key, event)
            .addCallback(
                result -> log.info("Event published: {}", event),
                failure -> log.error("Failed to publish event: {}", event, failure)
            );
    }
}
```

### Monitoring et Métriques
- **JMX Metrics** : Métriques natives Kafka
- **Prometheus Exporter** : Export vers Prometheus
- **Grafana Dashboards** : Visualisation des métriques
- **Alerting** : Alertes sur lag consumer, erreurs, etc.

### Métriques Clés
- **Producer** : Throughput, latence, erreurs
- **Consumer** : Lag, throughput, temps de traitement
- **Broker** : CPU, mémoire, I/O disque, réplication
- **Topics** : Taille, messages/sec, partitions actives