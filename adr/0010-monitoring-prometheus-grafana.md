# ADR-0010: Monitoring avec Prometheus et Grafana

## Statut

Accepté

## Contexte

Le système DPJ nécessite une solution de monitoring complète pour :
- Surveiller la santé des microservices
- Détecter les problèmes de performance
- Alerter en cas d'incidents
- Fournir des métriques métier aux équipes
- Assurer la conformité SLA (99.9% disponibilité)
- Faciliter le debugging et l'optimisation

Exigences :
- Métriques temps réel avec historique
- Alerting intelligent avec escalade
- Dashboards métier et technique
- Intégration avec l'infrastructure Kubernetes
- Rétention des données sur 1 an minimum

## Options Considérées

### Option 1: Prometheus + Grafana
- **Avantages** : Open source, écosystème riche, intégration K8s native
- **Inconvénients** : Configuration complexe, pas de clustering natif

### Option 2: Amazon CloudWatch
- **Avantages** : Managé AWS, intégration native, simplicité
- **Inconvénients** : Coûts élevés, vendor lock-in, fonctionnalités limitées

### Option 3: Datadog
- **Avantages** : SaaS complet, APM intégré, facilité d'usage
- **Inconvénients** : Coûts très élevés, dépendance externe

### Option 4: Elastic Stack (ELK)
- **Avantages** : Recherche puissante, visualisations riches
- **Inconvénients** : Complexité, consommation ressources, coûts licensing

## Décision

Nous adoptons **Prometheus + Grafana** avec l'écosystème suivant :

### Stack de Monitoring
- **Prometheus** : Collecte et stockage des métriques
- **Grafana** : Visualisation et dashboards
- **AlertManager** : Gestion des alertes et notifications
- **Jaeger** : Distributed tracing
- **Fluentd** : Agrégation des logs
- **Node Exporter** : Métriques système
- **Blackbox Exporter** : Monitoring externe

### Architecture
```
Applications → Prometheus → Grafana → Dashboards
     ↓              ↓
  Metrics      AlertManager → Notifications
     ↓              ↓
Jaeger Traces   PagerDuty/Slack/Email
```

## Conséquences

### Positives
- **Open Source** : Pas de coûts de licensing
- **Écosystème** : Large écosystème d'exporters et intégrations
- **Performance** : Optimisé pour les métriques time-series
- **Flexibilité** : Configuration et customisation complètes
- **Kubernetes** : Intégration native avec service discovery
- **Communauté** : Support communautaire actif

### Négatives
- **Complexité** : Configuration et maintenance complexes
- **Scalabilité** : Limitations pour très gros volumes
- **Haute Disponibilité** : Configuration HA complexe
- **Expertise** : Courbe d'apprentissage importante

### Risques
- Perte de métriques en cas de panne Prometheus
- Saturation du stockage des métriques
- Complexité de configuration des alertes

### Mitigations
- **Haute Disponibilité** : Déploiement multi-instance avec Thanos
- **Rétention** : Stratégie de rétention et archivage
- **Backup** : Sauvegarde régulière des configurations
- **Documentation** : Runbooks détaillés pour les opérations
- **Formation** : Formation équipe sur Prometheus/Grafana

## Configuration

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'dpj-prod'
    region: 'eu-west-1'

rule_files:
  - "alert-rules.yml"
  - "recording-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # Kubernetes API Server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
    - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

  # Kubernetes Nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
    - role: node
    relabel_configs:
    - source_labels: [__address__]
      regex: '(.*):10250'
      target_label: __address__
      replacement: '${1}:9100'

  # Application Services
  - job_name: 'dpj-services'
    kubernetes_sd_configs:
    - role: endpoints
    relabel_configs:
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
      action: keep
      regex: true
```

### Métriques Applicatives

#### Métriques Techniques
```java
// Configuration Spring Boot Actuator
@Component
public class CustomMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter documentUploadCounter;
    private final Timer documentProcessingTimer;
    private final Gauge activeUsersGauge;
    
    public CustomMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.documentUploadCounter = Counter.builder("dpj.documents.uploaded.total")
            .description("Total documents uploaded")
            .tag("service", "document-service")
            .register(meterRegistry);
            
        this.documentProcessingTimer = Timer.builder("dpj.documents.processing.duration")
            .description("Document processing duration")
            .register(meterRegistry);
            
        this.activeUsersGauge = Gauge.builder("dpj.users.active.current")
            .description("Current active users")
            .register(meterRegistry, this, CustomMetrics::getActiveUsersCount);
    }
}
```

#### Métriques Métier
- **Documents** : Upload rate, processing time, error rate
- **Dossiers** : Creation rate, completion time, status distribution
- **Utilisateurs** : Active users, login rate, session duration
- **GED Integration** : Sync rate, error rate, latency
- **Performance** : Response time, throughput, error rate

### Dashboards Grafana

#### Dashboard Technique
- **Infrastructure** : CPU, mémoire, disque, réseau
- **Kubernetes** : Pods, services, ingress, PVC
- **Applications** : JVM metrics, Spring Boot metrics
- **Bases de données** : Connexions, requêtes, performance

#### Dashboard Métier
- **Vue d'ensemble** : KPIs principaux, tendances
- **Documents** : Volume uploads, types, statuts
- **Dossiers** : Création, completion, SLA
- **Utilisateurs** : Activité, géographie, comportement

### Alerting Rules
```yaml
# alert-rules.yml
groups:
- name: dpj-application-alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 2m
    labels:
      severity: warning
      service: "{{ $labels.service }}"
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for service {{ $labels.service }}"

  - alert: ServiceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
      service: "{{ $labels.job }}"
    annotations:
      summary: "Service is down"
      description: "Service {{ $labels.job }} has been down for more than 1 minute"

  - alert: HighResponseTime
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High response time"
      description: "95th percentile response time is {{ $value }}s"
```

### Distributed Tracing avec Jaeger
```java
// Configuration OpenTelemetry
@Configuration
public class TracingConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        return OpenTelemetrySdk.builder()
            .setTracerProvider(
                SdkTracerProvider.builder()
                    .addSpanProcessor(BatchSpanProcessor.builder(
                        JaegerGrpcSpanExporter.builder()
                            .setEndpoint("http://jaeger-collector:14250")
                            .build())
                        .build())
                    .setResource(Resource.getDefault()
                        .merge(Resource.builder()
                            .put(ResourceAttributes.SERVICE_NAME, "dpj-document-service")
                            .build()))
                    .build())
            .buildAndRegisterGlobal();
    }
}
```

### Rétention et Archivage
- **Métriques haute résolution** : 7 jours
- **Métriques moyennées** : 30 jours  
- **Métriques agrégées** : 1 an
- **Archivage long terme** : Thanos ou S3 pour historique