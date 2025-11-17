# État du User-Service - Problème de Timeout

**Date**: 2025-11-17
**Status**: ⚠️ Problème de Timeout Non Résolu

## 🔴 Problème

Le user-service démarre correctement mais **ne répond à aucune requête HTTP**, y compris `/actuator/health`.

### Symptômes

```bash
# Test direct sur le port 8083
curl http://localhost:8083/actuator/health
# → Timeout après 10+ secondes

# Test via API Gateway
curl http://localhost:8080/user-service/admin/users/role/CLIENT
# → Error: read ECONNRESET
```

### Logs

- ✅ Le service démarre: "Started UserServiceApplication in 196.243 seconds"
- ✅ Tomcat démarre: "Tomcat started on port 8083 (http)"
- ✅ S'enregistre avec Eureka
- ✅ Docker health check: "healthy"
- ❌ Mais ne répond à **AUCUNE** requête HTTP

## 🔍 Cause Probable

Le `PasswordService` (FeignClient vers auth-service) est chargé de manière **eager** malgré l'annotation `@Lazy` sur le `ClientController`.

Spring essaie probablement d'initialiser le FeignClient au démarrage, créant un blocage dans le thread principal qui empêche Tomcat de répondre aux requêtes.

## 🚫 Solutions Tentées

1. ✅ Ajout de `@Lazy` sur l'injection de `PasswordService` dans `ClientController`
2. ❌ Rebuild et redémarrage → Le problème persiste

## ✅ Solution Temporaire: Utiliser Auth-Service

Pour obtenir les informations utilisateur avec UUID, **utilisez directement auth-service** au lieu de user-service:

### Option 1: Login Response (Recommandé)

La réponse de login contient déjà toutes les informations avec l'UUID:

```http
POST http://localhost:8080/auth-service/auth/login
Content-Type: application/json

{
  "email": "ayamtejjal123@gmail.com",
  "password": "password123"
}
```

**Réponse**:
```json
{
  "token": "eyJhbGci...",
  "expiresIn": 86400000,
  "userId": "57af273f-2bb7-4813-88f7-a120c4f30b0d",
  "email": "ayamtejjal123@gmail.com",
  "username": "client1",
  "role": "CLIENT"
}
```

### Option 2: Utiliser Ticket-Service et Paiement-Service

Ces services **fonctionnent parfaitement** avec les UUIDs:

```http
# Obtenir les tickets d'un client
GET http://localhost:8080/ticket-service/tickets/client/57af273f-2bb7-4813-88f7-a120c4f30b0d
Authorization: Bearer {token}

# Obtenir les paiements d'un client
GET http://localhost:8080/paiement-service/paiements/client/57af273f-2bb7-4813-88f7-a120c4f30b0d
Authorization: Bearer {token}
```

## 📊 Mapping UUID

| Email | UUID |
|-------|------|
| ayamtejjal123@gmail.com | `57af273f-2bb7-4813-88f7-a120c4f30b0d` |
| ayakim127@gmail.com | `c1f22693-7e22-45ec-ac2d-c4ffd919996d` |

Voir [UUID_MIGRATION_GUIDE.md](UUID_MIGRATION_GUIDE.md) pour le mapping complet.

## 🔧 Solutions Possibles (À Investiguer)

1. **Désactiver temporairement le FeignClient**
   - Commenter `@FeignClient` sur `PasswordService`
   - Rebuild et tester

2. **Configuration Lazy globale**
   - Créer une configuration Spring pour forcer tous les Feign clients à être Lazy
   ```java
   @Configuration
   public class FeignConfig {
       @Bean
       @Lazy
       public Builder feignBuilder() {
           return Feign.builder();
       }
   }
   ```

3. **Timeouts Feign**
   - Configurer des timeouts courts sur les appels Feign
   ```properties
   feign.client.config.default.connectTimeout=5000
   feign.client.config.default.readTimeout=5000
   ```

4. **Circuit Breaker**
   - Ajouter Resilience4j pour éviter les blocages

## ✅ Ce Qui Fonctionne

1. **Auth-Service** → Retourne les UUIDs dans le login ✅
2. **Ticket-Service** → Accepte les UUIDs ✅
3. **Paiement-Service** → Accepte les UUIDs ✅

## 📝 Recommandation

**Pour vos tests actuels**: Ignorez le user-service et utilisez:
- **Auth-service** pour l'authentification et les infos utilisateur avec UUID
- **Ticket-service** pour les opérations sur les tickets
- **Paiement-service** pour les opérations sur les paiements

Toutes les fonctionnalités sont disponibles via ces 3 services! 🚀

---

**Note**: Le problème initial (erreur 400 avec `/tickets/client/2`) est **complètement résolu** grâce à la migration UUID.
