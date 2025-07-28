# ADR-0008: Intégration GED avec Pattern Adapter

## Statut

Accepté

## Contexte

Le système DPJ doit s'intégrer avec le système GED (Gestion Électronique de Documents) existant pour :
- Éviter la duplication des documents
- Maintenir la cohérence des données
- Respecter les processus métier établis
- Minimiser l'impact sur le système existant
- Permettre une migration progressive

Contraintes :
- GED existante basée sur une API SOAP legacy
- Performance limitée du système existant
- Pas de modification possible de la GED
- Besoin de résilience en cas de panne GED

## Options Considérées

### Option 1: Intégration directe synchrone
- **Avantages** : Simplicité, cohérence immédiate
- **Inconvénients** : Couplage fort, performance dégradée, point de défaillance

### Option 2: Réplication complète des données
- **Avantages** : Performance, indépendance
- **Inconvénients** : Duplication, synchronisation complexe, incohérences

### Option 3: Pattern Adapter avec traitement asynchrone
- **Avantages** : Découplage, résilience, performance
- **Inconvénients** : Complexité, cohérence éventuelle

### Option 4: Remplacement complet de la GED
- **Avantages** : Architecture unifiée
- **Inconvénients** : Risque élevé, coût important, résistance au changement

## Décision

Nous adoptons le **Pattern Adapter avec traitement asynchrone** :

### Architecture d'Intégration
```
DPJ Services → GED Adapter → Message Queue → GED Sync Worker → GED Legacy
                    ↓
                Cache Local ← Event Store
```

### Composants Clés

#### GED Adapter Service
- **Responsabilité** : Interface unifiée vers la GED
- **API** : REST moderne pour les services DPJ
- **Cache** : Cache local des métadonnées fréquentes
- **Fallback** : Mode dégradé en cas de panne GED

#### Message Queue (Kafka)
- **Topics** : `ged-sync-requests`, `ged-sync-responses`, `ged-events`
- **Partitioning** : Par type de document pour parallélisation
- **Retention** : 7 jours pour rejeu en cas d'erreur

#### GED Sync Worker
- **Pattern** : Consumer Kafka avec retry et dead letter queue
- **Transformation** : Conversion REST → SOAP
- **Monitoring** : Métriques de synchronisation et erreurs

### Stratégies de Synchronisation

#### Documents Entrants (DPJ → GED)
1. **Upload DPJ** : Document stocké dans MinIO
2. **Event** : Événement publié dans Kafka
3. **Async Sync** : Worker synchronise vers GED
4. **Confirmation** : Mise à jour statut dans DPJ

#### Documents Sortants (GED → DPJ)
1. **Polling** : Vérification périodique des nouveaux documents
2. **Delta Sync** : Synchronisation des modifications
3. **Cache Update** : Mise à jour du cache local

## Conséquences

### Positives
- **Découplage** : Services DPJ indépendants de la GED
- **Performance** : Cache local et traitement asynchrone
- **Résilience** : Mode dégradé en cas de panne GED
- **Évolutivité** : Possibilité de remplacer la GED progressivement
- **Monitoring** : Visibilité complète sur les échanges

### Négatives
- **Complexité** : Architecture distribuée plus complexe
- **Cohérence** : Cohérence éventuelle entre systèmes
- **Latence** : Délai de synchronisation asynchrone
- **Debugging** : Traçabilité plus difficile

### Risques
- Perte de messages dans la queue
- Désynchronisation entre systèmes
- Performance dégradée de la GED legacy

### Mitigations
- **Idempotence** : Opérations idempotentes pour rejeu
- **Monitoring** : Alertes sur les échecs de synchronisation
- **Reconciliation** : Processus de réconciliation périodique
- **Circuit Breaker** : Protection contre les pannes GED
- **Audit Trail** : Traçabilité complète des échanges

## Implémentation

### GED Adapter Interface
```java
@RestController
@RequestMapping("/api/ged")
public class GedAdapterController {
    
    @PostMapping("/documents")
    public ResponseEntity<DocumentResponse> uploadDocument(
            @RequestBody DocumentRequest request) {
        // Validation et transformation
        // Publication événement Kafka
        // Retour immédiat avec statut "PENDING"
    }
    
    @GetMapping("/documents/{id}")
    public ResponseEntity<DocumentResponse> getDocument(
            @PathVariable String id) {
        // Vérification cache local
        // Fallback vers GED si nécessaire
    }
}
```

### Configuration Kafka
```yaml
kafka:
  topics:
    ged-sync-requests:
      partitions: 6
      replication-factor: 3
      retention-ms: 604800000  # 7 jours
    ged-sync-responses:
      partitions: 6
      replication-factor: 3
    ged-events:
      partitions: 3
      replication-factor: 3
```

### Métriques de Monitoring
- **Latence de synchronisation** : Temps entre événement et confirmation
- **Taux d'erreur** : Pourcentage d'échecs de synchronisation
- **Débit** : Nombre de documents synchronisés par minute
- **Santé GED** : Disponibilité et temps de réponse
- **Taille queue** : Backlog des messages en attente

### Stratégie de Récupération
1. **Retry automatique** : 3 tentatives avec backoff exponentiel
2. **Dead Letter Queue** : Messages en échec pour analyse
3. **Reconciliation** : Comparaison périodique DPJ ↔ GED
4. **Alerting** : Notifications en cas de désynchronisation