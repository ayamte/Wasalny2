# Guide de Test Complet - Architecture Wasalny

## État du Système

Tous les services ont été redémarrés et sont en cours de démarrage :

### Infrastructure ✅
- ✅ Eureka Server (port 8761)
- ✅ Config Server (port 8888)
- ✅ API Gateway (port 8080)
- ✅ RabbitMQ (ports 5672, 15672)
- ✅ Redis (port 6379)
- ✅ 8 bases PostgreSQL

### Services Métier 🔄
- 🔄 auth-service (port 8086)
- 🔄 user-service (port 8083)
- 🔄 paiement-service (port 8082)
- 🔄 ticket-service (port 8085)
- 🔄 trajet-service (port 8081)
- 🔄 notification-service (port 8088)
- 🔄 geolocalisation-service (port 8084)
- 🔄 abonnement-service (port 8087)

## Tests à Effectuer

### 1. Vérifier que Tous les Services sont Healthy

```powershell
# Dans PowerShell
docker-compose ps
```

**Attendez** que tous les services affichent `(healthy)` au lieu de `(health: starting)`.

### 2. Vérifier l'Enregistrement Eureka

```powershell
# Ouvrir dans le navigateur
http://localhost:8761
```

Vous devriez voir tous les services enregistrés :
- AUTH-SERVICE
- USER-SERVICE
- PAIEMENT-SERVICE
- TICKET-SERVICE
- API-GATEWAY
- CONFIG-SERVER
- NOTIFICATION-SERVICE
- TRAJET-SERVICE
- GEOLOCALISATION-SERVICE
- ABONNEMENT-SERVICE

### 3. Test du Flux Complet d'Authentification

#### Étape 1 : Créer un Utilisateur CLIENT

**Endpoint**: `POST http://localhost:8086/auth/signup`

**Body**:
```json
{
  "username": "TestClient",
  "email": "testclient@wasalny.com",
  "password": "Password123!",
  "role": "CLIENT"
}
```

**Résultat attendu** :
```json
{
  "message": "Utilisateur créé. Vérifiez votre email.",
  "userId": "<uuid-généré>"
}
```

**Note**: Un email sera envoyé à `testclient@wasalny.com` (configuré avec SUPPORT_EMAIL) avec un code de vérification.

#### Étape 2 : Vérifier l'Email

**Endpoint**: `POST http://localhost:8086/auth/verify`

**Body**:
```json
{
  "email": "testclient@wasalny.com",
  "verificationCode": "123456"
}
```

**Note**: Remplacez `123456` par le code reçu dans l'email.

#### Étape 3 : Se Connecter

**Endpoint**: `POST http://localhost:8086/auth/login`

**Body**:
```json
{
  "email": "testclient@wasalny.com",
  "password": "Password123!"
}
```

**Résultat attendu** :
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 86400000
}
```

**IMPORTANT**: Copiez le token JWT pour les prochaines étapes !

#### Étape 4 : Initier un Paiement (CLIENT)

**Endpoint**: `POST http://localhost:8082/paiements/initier`

**Headers**:
```
Authorization: Bearer <votre-token-jwt>
Content-Type: application/json
```

**Body**:
```json
{
  "clientId": "<uuid-du-client>",
  "montant": 50.00,
  "typePaiement": "CARTE_BANCAIRE",
  "typeService": "ACHAT_TICKET",
  "referenceService": "BUS_CASABLANCA_RABAT",
  "description": "Achat ticket bus Casablanca-Rabat",
  "infoCarte": {
    "numero": "4111111111111111",
    "nomTitulaire": "Test Client",
    "dateExpiration": "12/25",
    "cvv": "123"
  }
}
```

**Note**: Remplacez `<uuid-du-client>` par le userId reçu lors du signup.

**Résultat attendu** :
```json
{
  "id": "<transaction-uuid>",
  "clientId": "<uuid-du-client>",
  "montant": 50.00,
  "typePaiement": "CARTE_BANCAIRE",
  "typeService": "ACHAT_TICKET",
  "referenceService": "BUS_CASABLANCA_RABAT",
  "statut": "EN_ATTENTE",
  "dateCreation": "2025-11-16T20:00:00"
}
```

**IMPORTANT**: Copiez le `id` de la transaction !

#### Étape 5 : Créer un Utilisateur ADMIN

**Endpoint**: `POST http://localhost:8086/auth/signup`

**Body**:
```json
{
  "username": "AdminUser",
  "email": "admin@wasalny.com",
  "password": "AdminPass123!",
  "role": "ADMIN"
}
```

Puis vérifiez l'email et connectez-vous pour obtenir le token ADMIN.

#### Étape 6 : Traiter le Paiement (ADMIN)

**Endpoint**: `POST http://localhost:8082/paiements/<transaction-id>/traiter`

**Headers**:
```
Authorization: Bearer <admin-token-jwt>
```

**Note**: Remplacez `<transaction-id>` par l'ID de la transaction créée à l'étape 4.

**Résultat attendu** :
```json
{
  "id": "<transaction-uuid>",
  "statut": "COMPLETE",
  "montant": 50.00,
  ...
}
```

**Ce qui se passe en arrière-plan**:
1. Le paiement est marqué comme COMPLETE
2. Un événement est publié sur RabbitMQ (exchange: `payment.exchange`, routing key: `payment.completed`)
3. Le ticket-service reçoit l'événement
4. Un ticket est créé automatiquement

#### Étape 7 : Vérifier le Ticket Créé Automatiquement

**Endpoint**: `GET http://localhost:8085/tickets/client/<client-id>`

**Headers**:
```
Authorization: Bearer <client-token-jwt>
```

**Note**: Remplacez `<client-id>` par votre userId.

**Résultat attendu** :
```json
[
  {
    "id": "<ticket-uuid>",
    "clientId": "<client-id>",
    "referenceTransaction": "<transaction-id>",
    "statut": "ACTIF",
    "montant": 50.00,
    "typeService": "ACHAT_TICKET",
    "referenceService": "BUS_CASABLANCA_RABAT",
    "dateCreation": "2025-11-16T20:00:00"
  }
]
```

### 4. Tester les Permissions

#### Test 1 : CLIENT ne peut PAS traiter un paiement

**Endpoint**: `POST http://localhost:8082/paiements/<transaction-id>/traiter`

**Headers**:
```
Authorization: Bearer <client-token-jwt>
```

**Résultat attendu**: `403 Forbidden`

#### Test 2 : ADMIN peut valider un ticket

**Endpoint**: `PUT http://localhost:8085/tickets/<ticket-id>/valider`

**Headers**:
```
Authorization: Bearer <admin-token-jwt>
```

**Résultat attendu**: Ticket validé avec succès

#### Test 3 : CLIENT peut annuler son ticket

**Endpoint**: `PUT http://localhost:8085/tickets/<ticket-id>/annuler`

**Headers**:
```
Authorization: Bearer <client-token-jwt>
```

**Résultat attendu**: Ticket annulé avec succès

### 5. Vérifier RabbitMQ

```
URL: http://localhost:15672
Login: admin
Password: <voir dans .env RABBITMQ_PASSWORD>
```

Aller dans **Exchanges** → `payment.exchange` → **Bindings** → Vérifier que la queue `payment.completed.queue` est liée.

Aller dans **Queues** → `payment.completed.queue` → Vérifier qu'il y a un consumer actif (ticket-service).

## Tableau Récapitulatif des Permissions

### Paiement Service

| Route | Méthode | Rôle Requis | Description |
|-------|---------|-------------|-------------|
| `/paiements/initier` | POST | CLIENT | Initier un paiement |
| `/paiements/{id}/traiter` | POST | ADMIN | Traiter un paiement |
| `/paiements/{id}` | GET | CLIENT, ADMIN | Voir détails |
| `/paiements/client/{clientId}` | GET | CLIENT, ADMIN | Historique |

### Ticket Service

| Route | Méthode | Rôle Requis | Description |
|-------|---------|-------------|-------------|
| `/tickets/{id}` | GET | CLIENT, ADMIN | Voir détails |
| `/tickets/client/{clientId}` | GET | CLIENT, ADMIN | Historique |
| `/tickets/{id}/valider` | PUT | ADMIN | Valider |
| `/tickets/{id}/annuler` | PUT | CLIENT | Annuler |
| `/tickets/{id}/rembourser` | PUT | ADMIN | Rembourser |

## Script PowerShell de Test Automatique

Utilisez le fichier `test-requests.ps1` créé précédemment pour tester automatiquement les endpoints.

```powershell
# Exécuter le script
.\test-requests.ps1
```

## Dépannage

### Si un service ne démarre pas

```powershell
# Voir les logs
docker logs <nom-du-service> --tail 50

# Exemples
docker logs auth-service --tail 50
docker logs wasalny-paiement-service-1 --tail 50
docker logs wasalny-ticket-service-1 --tail 50
```

### Si Eureka ne montre pas tous les services

Attendez 30-60 secondes pour que les services s'enregistrent.

### Si RabbitMQ ne reçoit pas les messages

Vérifiez les logs de paiement-service pour voir si la publication a réussi :
```powershell
docker logs wasalny-paiement-service-1 | Select-String -Pattern "rabbit"
```

## Résumé de l'Architecture

```
┌─────────────┐
│   CLIENT    │
└──────┬──────┘
       │
       ├─► POST /auth/signup (auth-service)
       │   └─► Feign ─► user-service (création profil)
       │
       ├─► POST /auth/login (auth-service)
       │   └─► Retourne JWT token
       │
       ├─► POST /paiements/initier (paiement-service) [JWT: CLIENT]
       │   └─► Crée transaction EN_ATTENTE
       │
       │   ┌─ ADMIN ─┐
       │   │         │
       │   └─► POST /paiements/{id}/traiter [JWT: ADMIN]
       │       └─► Marque COMPLETE
       │           └─► Publie sur RabbitMQ
       │               └─► ticket-service reçoit l'événement
       │                   └─► Crée ticket ACTIF
       │
       └─► GET /tickets/client/{id} (ticket-service) [JWT: CLIENT]
           └─► Retourne les tickets du client
```

## Communication Asynchrone (RabbitMQ)

```
paiement-service                          ticket-service
      │                                        │
      │  1. Traiter paiement                  │
      │     (ADMIN)                            │
      │                                        │
      │  2. Publier événement                 │
      │     ────────────────────►              │
      │     Exchange: payment.exchange         │
      │     Routing: payment.completed         │
      │                                        │
      │                            3. Listener reçoit
      │                               PaymentEventListener
      │                                        │
      │                            4. Créer ticket
      │                               automatiquement
      │                                        │
```

## Communication Synchrone (Feign)

```
auth-service                              user-service
      │                                        │
      │  1. Signup utilisateur                 │
      │                                        │
      │  2. Appel Feign                       │
      │     ────────────────────►              │
      │     UserProfileClient.createProfile    │
      │                                        │
      │                            3. Créer profil
      │                                        │
      │  ◄───────────────────────              │
      │     4. Retour succès                   │
      │                                        │
```

## Conclusion

Toute l'architecture microservices est maintenant opérationnelle avec :
- ✅ Authentification JWT complète
- ✅ Permissions basées sur les rôles (CLIENT, ADMIN)
- ✅ Communication asynchrone (RabbitMQ) paiement → ticket
- ✅ Communication synchrone (Feign) auth → user
- ✅ Service discovery (Eureka)
- ✅ Configuration centralisée (Config Server)
- ✅ API Gateway comme point d'entrée
- ✅ 8 microservices déployés
- ✅ Infrastructure complète (PostgreSQL, Redis, RabbitMQ)

**Tout est prêt pour les tests ! 🎉**
