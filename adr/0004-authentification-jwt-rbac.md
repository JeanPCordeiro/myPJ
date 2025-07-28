# ADR-0004: Authentification JWT avec Autorisation RBAC

## Statut

Accepté

## Contexte

Le système DPJ nécessite une solution d'authentification et d'autorisation pour :
- Clients bancaires (accès à leurs propres dossiers)
- Collaborateurs bancaires (accès selon leur rôle)
- Intégration avec l'Active Directory existant
- Support des interfaces web et mobile
- Sécurité bancaire renforcée

## Options Considérées

### Option 1: Sessions serveur traditionnelles
- **Avantages** : Simplicité, contrôle serveur
- **Inconvénients** : Pas de scalabilité horizontale, pas de stateless

### Option 2: OAuth 2.0 + OpenID Connect
- **Avantages** : Standard industrie, délégation
- **Inconvénients** : Complexité, dépendance externe

### Option 3: JWT + RBAC interne
- **Avantages** : Stateless, performance, contrôle total
- **Inconvénients** : Gestion des tokens, révocation

## Décision

Nous adoptons **JWT (JSON Web Tokens) avec autorisation RBAC** :

### Authentification JWT
- **Access Token** : JWT signé, durée 15 minutes
- **Refresh Token** : Token opaque, durée 7 jours, stocké en base
- **Algorithme** : RS256 (RSA + SHA-256)
- **Claims** : `sub`, `roles`, `permissions`, `exp`, `iat`, `jti`

### Autorisation RBAC
- **Rôles** : `CLIENT`, `CONSEILLER`, `VALIDATEUR`, `ADMIN`
- **Permissions** : Granulaires par ressource et action
- **Hiérarchie** : Héritage de permissions entre rôles
- **Contexte** : Permissions contextuelles (agence, client)

### Intégration Active Directory
- **Authentification** : Délégation vers AD via LDAP
- **Synchronisation** : Import périodique des utilisateurs
- **Mapping** : Correspondance groupes AD → rôles DPJ

## Architecture

```
Client → API Gateway → JWT Validation → Service
                    ↓
                 RBAC Check → Resource Access
```

## Conséquences

### Positives
- **Stateless** : Scalabilité horizontale des services
- **Performance** : Pas de lookup base pour chaque requête
- **Sécurité** : Tokens signés, expiration courte
- **Flexibilité** : Permissions granulaires et contextuelles
- **Intégration** : Compatible avec l'AD existant

### Négatives
- **Révocation** : Difficulté de révocation immédiate des access tokens
- **Taille** : JWT plus volumineux que les IDs de session
- **Complexité** : Gestion des clés de signature

### Risques
- Compromission des clés de signature
- Tokens volés avant expiration
- Complexité de la gestion des permissions

### Mitigations
- **Rotation des clés** : Rotation périodique des clés RSA
- **Durée courte** : Access tokens de 15 minutes maximum
- **Blacklist** : Liste noire des tokens compromis
- **Audit** : Logging complet des accès et permissions
- **Chiffrement** : Tokens sensibles chiffrés en transit (TLS)
- **Rate limiting** : Protection contre les attaques par force brute

## Implémentation

### Structure JWT
```json
{
  "sub": "user123",
  "roles": ["CONSEILLER"],
  "permissions": ["document:read", "dossier:write"],
  "agence": "AG001",
  "exp": 1640995200,
  "iat": 1640994300,
  "jti": "token-unique-id"
}
```

### Validation des Permissions
```java
@PreAuthorize("hasPermission(#dossier, 'READ')")
public Dossier getDossier(String dossierId) {
    // Logique métier
}