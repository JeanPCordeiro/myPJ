# ADR-0007: Infrastructure Kubernetes sur AWS

## Statut

Accepté

## Contexte

Le système DPJ nécessite une infrastructure capable de :
- Supporter 500+ utilisateurs simultanés
- Garantir 99.9% de disponibilité
- Permettre le scaling automatique
- Faciliter les déploiements et rollbacks
- Assurer la sécurité bancaire
- Optimiser les coûts d'infrastructure

## Options Considérées

### Option 1: Serveurs physiques traditionnels
- **Avantages** : Contrôle total, sécurité physique
- **Inconvénients** : Pas de scalabilité, coûts élevés, maintenance

### Option 2: Machines virtuelles (VMware)
- **Avantages** : Flexibilité, isolation
- **Inconvénients** : Gestion complexe, pas d'orchestration native

### Option 3: Kubernetes on-premise
- **Avantages** : Contrôle total, pas de dépendance cloud
- **Inconvénients** : Complexité opérationnelle, expertise requise

### Option 4: Amazon EKS (Kubernetes managé)
- **Avantages** : Kubernetes managé, intégration AWS, expertise disponible
- **Inconvénients** : Dépendance AWS, coûts cloud

## Décision

Nous adoptons **Amazon EKS (Elastic Kubernetes Service)** avec l'architecture suivante :

### Architecture EKS
- **Cluster EKS** : Multi-AZ (3 zones de disponibilité)
- **Node Groups** : Auto Scaling Groups avec instances mixtes
- **Networking** : VPC dédié avec subnets privés/publics
- **Load Balancer** : Application Load Balancer (ALB)
- **Storage** : EBS CSI Driver + EFS pour stockage partagé

### Configuration Cluster
```yaml
# Cluster EKS
Cluster: dpj-prod-cluster
Version: 1.28
Regions: eu-west-1 (Paris)
AZs: eu-west-1a, eu-west-1b, eu-west-1c

# Node Groups
- name: system-nodes
  instance_types: [t3.medium, t3.large]
  min_size: 3
  max_size: 6
  desired_size: 3

- name: app-nodes
  instance_types: [c5.large, c5.xlarge, m5.large]
  min_size: 3
  max_size: 20
  desired_size: 6
```

### Services AWS Intégrés
- **EKS** : Orchestration Kubernetes managée
- **EC2** : Instances de calcul pour les nœuds
- **VPC** : Réseau isolé et sécurisé
- **ALB** : Load balancing applicatif
- **EBS** : Stockage persistant pour bases de données
- **EFS** : Stockage partagé pour logs et configs
- **ECR** : Registry Docker privé
- **CloudWatch** : Monitoring et logging
- **IAM** : Gestion des accès et permissions

## Conséquences

### Positives
- **Scalabilité** : Auto-scaling horizontal et vertical
- **Disponibilité** : Multi-AZ avec failover automatique
- **Sécurité** : Intégration IAM, VPC, security groups
- **Maintenance** : Control plane managé par AWS
- **Écosystème** : Intégration native avec services AWS
- **Expertise** : Équipe déjà formée sur AWS

### Négatives
- **Coûts** : Coûts cloud potentiellement élevés
- **Dépendance** : Vendor lock-in avec AWS
- **Complexité** : Courbe d'apprentissage Kubernetes

### Risques
- Panne régionale AWS
- Augmentation des coûts cloud
- Complexité opérationnelle Kubernetes

### Mitigations
- **Multi-AZ** : Déploiement sur 3 zones de disponibilité
- **Monitoring** : Surveillance proactive des coûts
- **Formation** : Formation équipe sur Kubernetes
- **Backup** : Stratégie de backup multi-région
- **Cost Optimization** : Spot instances, reserved instances

## Architecture Réseau

### VPC Configuration
```
VPC: 10.0.0.0/16
├── Public Subnets (ALB, NAT Gateway)
│   ├── 10.0.1.0/24 (AZ-a)
│   ├── 10.0.2.0/24 (AZ-b)
│   └── 10.0.3.0/24 (AZ-c)
└── Private Subnets (EKS Nodes, RDS)
    ├── 10.0.10.0/24 (AZ-a)
    ├── 10.0.20.0/24 (AZ-b)
    └── 10.0.30.0/24 (AZ-c)
```

### Sécurité
- **Security Groups** : Règles de firewall granulaires
- **NACLs** : Contrôle d'accès réseau par subnet
- **IAM Roles** : Permissions minimales par service
- **Pod Security Standards** : Policies de sécurité Kubernetes
- **Network Policies** : Isolation réseau entre namespaces

## Déploiement et CI/CD

### GitOps avec ArgoCD
- **Repository** : Git comme source de vérité
- **ArgoCD** : Déploiement automatique et synchronisation
- **Helm Charts** : Packaging des applications
- **Environments** : dev, staging, production

### Pipeline CI/CD
```
Code Push → GitHub Actions → Build → Test → Push ECR → ArgoCD → Deploy EKS
```

### Monitoring et Observabilité
- **Prometheus** : Métriques cluster et applications
- **Grafana** : Dashboards et visualisation
- **Jaeger** : Distributed tracing
- **Fluentd** : Agrégation des logs
- **CloudWatch** : Monitoring infrastructure AWS