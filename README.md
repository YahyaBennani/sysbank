# BankSys — Application Bancaire Sécurisée

> Plateforme de transactions financières avec authentification 2FA, contrôle OFAC, paiement QR Code et API mobile Android.

---

Pour tester notre app dans local veuillez placer docker-compose.yml dans le rep du projet et :

```
docker compose up -d
```
---
## Table des Matières

1. [Vue d'ensemble](#1-vue-densemble)
2. [Stack Technologique](#2-stack-technologique)
3. [Architecture du Projet](#3-architecture-du-projet)
4. [Structure des Fichiers](#4-structure-des-fichiers)
5. [Base de Données](#5-base-de-données)
6. [API REST — Endpoints](#6-api-rest--endpoints)
7. [Sécurité](#7-sécurité)
8. [Fonctionnalités](#8-fonctionnalités)
9. [Frontend React](#9-frontend-react)
10. [Installation et Démarrage](#10-installation-et-démarrage)
11. [Guide de Tests](#11-guide-de-tests)
12. [Variables d'Environnement](#12-variables-denvironnement)

---

## 1. Vue d'ensemble

BankSys est une application bancaire web complète simulant les fonctionnalités d'une banque numérique moderne. Elle repose sur une architecture client-serveur découplée :

- **Backend** : API REST Spring Boot (port 8080)
- **Frontend** : SPA React.js (port 3000)
- **Mobile** : APK Android natif Java
- **Base de données** : PostgreSQL persistant

### Rôles

| Rôle | Accès |
|------|-------|
| `ROLE_ADMIN` | Gestion complète — clients, transactions, OFAC, crédits |
| `ROLE_CLIENT` | Ses propres données — solde, virements, historique, QR |

### Flux d'authentification

```
POST /auth/register
        ↓
    Compte PENDING (en attente d'approbation admin)
        ↓
POST /admin/clients/{id}/approve  ← admin
        ↓
POST /auth/login  (email + password)
        ↓ retourne preAuthToken (JWT 5 min, type=PRE_AUTH)
POST /auth/verify-otp  (preAuthToken + code TOTP)
        ↓ retourne accessToken (15 min) + refreshToken (7 jours)
        ↓
    Accès aux routes protégées
```

---

## 2. Stack Technologique

### Backend
| Composant | Technologie | Version |
|-----------|-------------|---------|
| Framework | Spring Boot | 3.5.x |
| Langage | Java | 21 |
| ORM | Spring Data JPA / Hibernate | 6.x |
| Sécurité | Spring Security | 6.x |
| JWT | jjwt | 0.12.6 |
| 2FA | dev.samstevens.totp | 1.7.1 |
| Rate Limiting | Bucket4j | 8.10.1 |
| Build | Maven | 3.x |

### Base de Données
| Composant | Technologie |
|-----------|-------------|
| SGBD | PostgreSQL 16 |
| Pool | HikariCP |
| Migrations | Hibernate `ddl-auto=update` |

### Frontend
| Composant | Technologie |
|-----------|-------------|
| Framework | React.js 18 |
| Router | react-router-dom v6 |
| HTTP | Axios |
| Style | Bootstrap 5 + CSS inline (theme cyberpunk) |
| QR Code | qrcode.react |

### Mobile Android
| Composant | Technologie |
|-----------|-------------|
| Langage | Java |
| SDK min | API 24 (Android 7) |
| HTTP | Retrofit 2 + OkHttp |
| Scanner QR | ZXing Android Embedded |
| Stockage sécurisé | EncryptedSharedPreferences (AES256) |

---

## 3. Architecture du Projet

```
project/
├── model/                          ← Spring Boot backend
│   ├── src/main/java/com/exemple/
│   │   ├── controller/             ← Couche HTTP (REST endpoints)
│   │   ├── service/                ← Logique métier
│   │   ├── repository/             ← Accès base de données (JPA)
│   │   ├── entity/                 ← Entités JPA (tables)
│   │   ├── dto/
│   │   │   ├── request/            ← Objets entrants (JSON → Java)
│   │   │   └── response/           ← Objets sortants (Java → JSON)
│   │   ├── security/               ← JWT, filtres, config Spring Security
│   │   └── exception/              ← Gestion globale des erreurs
│   └── src/main/resources/
│       └── application.properties
│
├── frontend/                       ← React SPA
    └── src/
        ├── pages/                  ← Login, Register, VerifyOtp, Dashboard, AdminPanel
        ├── services/api.js         ← Couche Axios centralisée
        ├── context/AuthContext.js  ← État global authentification
        └── App.js                  ← Router + routes protégées


```

### Couches Backend (MVC enrichi)

```
HTTP Request
     ↓
RateLimitFilter          ← 30 req/min/IP (Bucket4j)
     ↓
JwtAuthFilter            ← Valide JWT, rejette PRE_AUTH sur routes protégées
     ↓
SecurityConfig           ← Règles d'autorisation par rôle
     ↓
Controller               ← Reçoit DTO, délègue au service
     ↓
Service                  ← Logique métier, @Transactional
     ↓
Repository               ← Spring Data JPA → SQL
     ↓
PostgreSQL
```

---

## 4. Structure des Fichiers

### Controllers

| Fichier | Routes | Description |
|---------|--------|-------------|
| `AuthController` | `/auth/**` | Register, login, verify-otp, refresh, logout |
| `UserController` | `/me` | Profil du client connecté |
| `TransactionController` | `/transactions/**` | Virements et historique |
| `AdminController` | `/admin/**` | Gestion clients, transactions, OFAC |
| `QrPaymentController` | `/qr/**` | Génération et paiement QR |

### Services

| Fichier | Responsabilité |
|---------|----------------|
| `AuthService` | Inscription, login 2FA, vérification OTP, tokens |
| `TransactionService` | Virements ACID, historique, crédit admin |
| `TransactionEchecService` | Sauvegarde des transactions échouées (REQUIRES_NEW) |
| `OfacService` | Vérification liste noire OFAC |
| `QrPaymentService` | Génération token QR, preview, paiement |
| `JwtService` | Génération/validation ACCESS, REFRESH, PRE_AUTH, QR tokens |

### Entities (Tables PostgreSQL)

| Entité | Table | Description |
|--------|-------|-------------|
| `User` | `users` | Comptes clients et admins |
| `Transaction` | `transactions` | Historique des opérations |
| `RefreshToken` | `refresh_tokens` | Tokens de renouvellement (hashés) |
| `OfacEntry` | `ofac_entries` | Liste noire OFAC |

### DTOs

**Requests :**
- `RegisterRequest` — username, email, password
- `LoginRequest` — email, password
- `VerifyOtpRequest` — preAuthToken, otpCode
- `TransactionRequest` — destinataireId, montant, devise, description
- `QrPaymentRequest` — montant, description

**Responses :**
- `AuthResponse` — accessToken, refreshToken, role, message
- `RegisterResponse` — userId, message, totpSecret, totpQrUrl
- `TransactionResponse` — id, expediteurUsername, destinataireUsername, montant, statut, createdAt
- `QrPaymentResponse` — qrToken, qrContent, magasinId, montant, expiresAt

---

## 5. Base de Données

### Table `users`

```sql
CREATE TABLE users (
    id             BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username       VARCHAR(255) NOT NULL UNIQUE,
    email          VARCHAR(255) NOT NULL UNIQUE,
    password_hash  VARCHAR(255) NOT NULL,          -- BCrypt
    role           VARCHAR(50)  NOT NULL,           -- ROLE_CLIENT | ROLE_ADMIN
    statut         VARCHAR(50)  NOT NULL,           -- PENDING | APPROVED | BLOCKED
    solde          NUMERIC(19,4) NOT NULL,
    totp_secret    VARCHAR(255),                   -- Secret TOTP Google Authenticator
    two_fa_enabled BOOLEAN NOT NULL,
    created_at     TIMESTAMP NOT NULL
);
```

### Table `transactions`

```sql
CREATE TABLE transactions (
    id               BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    expediteur_id    BIGINT NOT NULL REFERENCES users(id),
    destinataire_id  BIGINT REFERENCES users(id),  -- nullable pour FAILED sans destinataire
    montant          NUMERIC(19,4) NOT NULL,
    devise           VARCHAR(10) NOT NULL DEFAULT 'MAD',
    statut           VARCHAR(50) NOT NULL,          -- COMPLETED | FAILED
    description      VARCHAR(255),
    created_at       TIMESTAMP NOT NULL
);
```

### Table `refresh_tokens`

```sql
CREATE TABLE refresh_tokens (
    id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    token_hash  VARCHAR(255) NOT NULL UNIQUE,       -- SHA-256 du token brut
    expires_at  TIMESTAMP NOT NULL,
    revoked     BOOLEAN NOT NULL DEFAULT FALSE,
    ip          VARCHAR(255),
    created_at  TIMESTAMP NOT NULL
);
```

### Table `ofac_entries`

```sql
CREATE TABLE ofac_entries (
    id         BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    nom        VARCHAR(255) NOT NULL,
    pays       VARCHAR(100),
    actif      BOOLEAN NOT NULL DEFAULT TRUE,
    date_ajout TIMESTAMP NOT NULL
);
```

### Propriétés ACID des transactions

| Propriété | Mécanisme |
|-----------|-----------|
| **Atomique** | `@Transactional` — rollback si exception à n'importe quelle étape |
| **Cohérente** | Vérification solde > montant avant débit |
| **Isolée** | `Isolation.SERIALIZABLE` — empêche les lectures sales et race conditions |
| **Durable** | Commit PostgreSQL sur disque |

> **Note** : Les transactions échouées (FAILED) sont sauvegardées via `TransactionEchecService` annoté `@Transactional(propagation = Propagation.REQUIRES_NEW)`, créant une transaction DB indépendante qui se commite même si la principale fait rollback.

---

## 6. API REST — Endpoints

### Auth (public)

```
POST /auth/register         Corps: { username, email, password }
POST /auth/login            Corps: { email, password }
POST /auth/verify-otp       Corps: { preAuthToken, otpCode }
POST /auth/refresh          Header: X-Refresh-Token: {token}
POST /auth/logout           Header: Authorization: Bearer {token}
```

### Client (authentifié)

```
GET  /me                                        Profil + solde + infos
POST /transactions                              Initier un virement
GET  /transactions/historique?page=0&size=10    Historique paginé
GET  /transactions/{id}                         Détail d'une transaction
```

### Admin (`ROLE_ADMIN` requis)

```
GET  /admin/clients                             Liste tous les utilisateurs
POST /admin/clients/{id}/approve               Approuver un compte PENDING
POST /admin/clients/{id}/block                 Bloquer un compte
POST /admin/clients/{id}/promote               Promouvoir en ROLE_ADMIN
POST /admin/clients/{id}/credit                Corps: { montant }
GET  /admin/transactions?page=0&size=10        Toutes les transactions (COMPLETED + FAILED)
POST /admin/transactions                       Virement depuis compte admin
GET  /admin/ofac                               Liste noire OFAC
POST /admin/ofac                               Corps: { nom, pays }
DELETE /admin/ofac/{id}                        Désactive une entrée (soft delete)
```

### QR Payment (authentifié)

```
POST /qr/generate           Corps: { montant, description }
POST /qr/preview            Corps: { qrToken }  ← public, sans auth
POST /qr/pay                Corps: { qrToken }
```

### Codes HTTP

| Code | Signification |
|------|---------------|
| 200 | Succès |
| 201 | Ressource créée |
| 204 | Succès sans contenu (logout) |
| 400 | Erreur métier (solde insuffisant, OTP invalide...) |
| 401 | Non authentifié ou token expiré |
| 403 | Accès refusé (mauvais rôle) |
| 429 | Trop de requêtes (rate limiting) |

---

## 7. Sécurité

### Authentification 2FA — Flux détaillé

```
Étape 1 — POST /auth/login
  • Vérifie email + password (BCrypt)
  • Vérifie statut APPROVED
  • Pour ROLE_CLIENT → génère preAuthToken (JWT, claim type=PRE_AUTH, 5 min)
  • Pour ROLE_ADMIN  → génère accessToken + refreshToken directement

Étape 2 — POST /auth/verify-otp
  • Vérifie présence et validité du preAuthToken
  • Vérifie claim type=PRE_AUTH (empêche recyclage d'un accessToken)
  • Extrait userId depuis le subject JWT (pas depuis le body — sécurité IDOR)
  • Vérifie code TOTP (RFC 6238, fenêtre 30 secondes)
  • Génère accessToken (15 min) + refreshToken (7 jours)
```

### Types de Tokens JWT

| Type | Durée | Claim `type` | Usage |
|------|-------|-------------|-------|
| `ACCESS` | 15 min | `ACCESS` | Toutes les requêtes API protégées |
| `REFRESH` | 7 jours | `REFRESH` | Renouveler l'accessToken, stocké hashé SHA-256 en DB |
| `PRE_AUTH` | 5 min | `PRE_AUTH` | Lier login → verify-otp, rejeté sur toutes les autres routes |
| `QR_PAYMENT` | 15 min | `QR_PAYMENT` | Paiement QR Code, contient magasinId + montant |

### Chaîne de Filtres Spring Security

```
1. RateLimitFilter        → 30 req/min/IP (Token Bucket, Bucket4j)
2. JwtAuthFilter          → valide JWT, rejette PRE_AUTH sur routes protégées
3. SecurityConfig rules   → /auth/** public, /admin/** ROLE_ADMIN, reste auth
```

### Protections OWASP

| Menace | Protection | Mécanisme |
|--------|-----------|-----------|
| SQLi | ✅ | JPA/Hibernate — requêtes paramétrées systématiques |
| XSS | ✅ | API JSON pure — pas de rendu HTML côté serveur |
| CSRF | ✅ | JWT stateless — pas de cookies de session |
| IDOR | ✅ | Vérification ownership dans chaque service |
| DDoS/Brute force | ✅ | Rate limiting 30 req/min/IP |
| Auth bypass 2FA | ✅ | preAuthToken signé — impossible de sauter l'étape OTP |
| Énumération utilisateurs | ✅ | Message d'erreur générique "Identifiants incorrects" |
| Token replay | ✅ | Rotation des refresh tokens à chaque utilisation |

### Vérification OFAC

```
Avant chaque transaction :
  1. ofacService.verifier(expediteur.username)
  2. ofacService.verifier(expediteur.email)
  3. ofacService.verifier(destinataire.username)
  4. ofacService.verifier(destinataire.email)

Si match → transaction FAILED enregistrée + RuntimeException levée
```

### CORS

Autorisé uniquement depuis `http://localhost:3000` (dev). En production, remplacer par le domaine du frontend.

---

## 8. Fonctionnalités

### Inscription et Approbation

1. Client s'inscrit via `POST /auth/register`
2. Reçoit un `totpSecret` et une URL `otpauth://` à scanner dans Google Authenticator
3. Compte en statut `PENDING` — inaccessible jusqu'à approbation
4. Admin approuve via `POST /admin/clients/{id}/approve`
5. Client peut maintenant se connecter

### Transactions Bancaires

- **Virement normal** : POST /transactions — vérifie OFAC, solde, auto-virement, puis débit + crédit atomique
- **Virement admin** : POST /admin/transactions — même flux, depuis le compte admin
- **Crédit admin** : POST /admin/clients/{id}/credit — simule un versement bancaire, tracé comme transaction
- **Toutes les transactions** : COMPLETED et FAILED sont persistées — l'admin voit tout

### Paiement QR Code

```
Scénario Magasin (client A) :
  1. Ouvre [ QR PAY ] → onglet CRÉER QR
  2. Entre le montant (ex: 150 MAD) + description
  3. QR Code généré (JWT signé 15 min, encodé en QR visuel)
  4. Affiche le QR à l'acheteur

Scénario Acheteur (client B, compte différent) :
  1. Ouvre [ QR PAY ] → onglet PAYER VIA QR
  2. Colle le token JWT (ou scanne via app Android)
  3. Voit l'aperçu : bénéficiaire, montant, expiration
  4. Confirme → transaction exécutée immédiatement
```

> **Important** : Le magasin et l'acheteur doivent être deux comptes différents. Un auto-paiement est bloqué.

---

## 9. Frontend React

### Pages

| Page | Route | Accès |
|------|-------|-------|
| Login | `/login` | Public |
| Register | `/register` | Public |
| Verify OTP | `/verify-otp` | Public (nécessite preAuthToken en sessionStorage) |
| Dashboard | `/dashboard` | ROLE_CLIENT |
| Admin Panel | `/admin` | ROLE_ADMIN |

### AuthContext

```javascript
// Fourni à toute l'application via React Context
const { user, login, logout } = useAuth();

// login() → persiste tokens en localStorage
// logout() → vide localStorage + reset état
// user.role → 'ROLE_CLIENT' | 'ROLE_ADMIN'
```

### Service API (Axios)

```javascript
// Intercepteur request → injecte automatiquement Bearer token
// Intercepteur response → redirige vers /login si 401

export const authService = { register, login, verifyOtp, logout, refresh };
export const clientService = { me, historique, transacter };
export const adminService = { clients, approuver, bloquer, promouvoir, crediter,
                              transactions, transactionAdmin, ofacList, ofacAjouter, ofacSupprimer };
```

### Design — Cyberpunk Banking

- Fond : `#020b18` (bleu nuit) avec grille de fond CSS
- Accent client : `#00ffc8` (cyan néon)
- Accent admin : `#ff5000` (orange vif)
- Typographie : `Share Tech Mono` (monospace) + `Rajdhani`
- Animations : pulse sur le status dot, glows sur les boutons


---

## 10. Installation et Démarrage

### Prérequis

```bash
Java 21+
Maven 3.x
Node.js 18+
PostgreSQL 16
```

### PostgreSQL

```bash
sudo -u postgres psql << 'EOF'
CREATE DATABASE bankapp;
CREATE USER bankuser WITH PASSWORD 'BankPass2026!';
GRANT ALL PRIVILEGES ON DATABASE bankapp TO bankuser;
\c bankapp
GRANT ALL ON SCHEMA public TO bankuser;
EOF
```

### Backend Spring Boot

```bash
cd project/model
./mvnw spring-boot:run
# Démarre sur http://localhost:8080
# Admin créé automatiquement : admin@bank.com / Admin123!
```

### Frontend React

```bash
cd project/frontend
npm install
npm start
# Démarre sur http://localhost:3000
```


### Variables d'environnement (optionnel, production)

```bash
export ADMIN_EMAIL=admin@mabanque.com
export ADMIN_PASSWORD=MotDePasseSecurise2026!
export ADMIN_USERNAME=superadmin
./mvnw spring-boot:run
```

---

## 11. Guide de Tests

### Test Complet — Flux Client

```bash
# 1. Inscription
POST http://localhost:8080/auth/register
{
  "username": "yahya",
  "email": "yahya@bank.com",
  "password": "MonMotDePasse123!"
}
# → Copier totpSecret → scanner dans Google Authenticator

# 2. Approbation admin
POST http://localhost:8080/auth/login
{ "email": "admin@bank.com", "password": "Admin123!" }
# → copier adminToken

POST http://localhost:8080/admin/clients/2/approve
Authorization: Bearer {adminToken}

# 3. Créditer le compte
POST http://localhost:8080/admin/clients/2/credit
Authorization: Bearer {adminToken}
{ "montant": 5000 }

# 4. Login client
POST http://localhost:8080/auth/login
{ "email": "yahya@bank.com", "password": "MonMotDePasse123!" }
# → message: OTP_REQUIRED + accessToken (=preAuthToken)

# 5. Verify OTP
POST http://localhost:8080/auth/verify-otp
{
  "preAuthToken": "eyJ...",
  "otpCode": "123456"
}
# → accessToken + refreshToken

# 6. Virement
POST http://localhost:8080/transactions
Authorization: Bearer {accessToken}
{
  "destinataireId": 3,
  "montant": 500,
  "devise": "MAD",
  "description": "Test"
}
```

### Tests de Sécurité

```bash
# Rate limiting — boucle bash
for i in $(seq 1 35); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    http://localhost:8080/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@test.com","password":"test"}')
  echo "Req $i → HTTP $STATUS"
done
# Attendu : req 1-30 → 200/400, req 31-35 → 429

# SQLi
POST /auth/login
{ "email": "' OR '1'='1", "password": "anything" }
# Attendu : 400 "Identifiants incorrects" (pas d'accès)

# Contournement 2FA — verify-otp sans preAuthToken
POST /auth/verify-otp
{ "preAuthToken": "", "otpCode": "123456" }
# Attendu : 400 "preAuthToken manquant"

# Utiliser preAuthToken comme accessToken sur route protégée
GET /me
Authorization: Bearer {preAuthToken}
# Attendu : 401 "Token temporaire"

# IDOR — accéder à la transaction d'un autre
GET /transactions/1
Authorization: Bearer {tokenClientQuiNEstPasConcerné}
# Attendu : 400 "Accès refusé"

# Auto-virement QR
# Générer QR avec compte A → scanner avec compte A
# Attendu : 400 "Vous ne pouvez pas scanner votre propre QR"

# OFAC
POST /admin/ofac
Authorization: Bearer {adminToken}
{ "nom": "yahya" }

POST /transactions
Authorization: Bearer {tokenYahya}
{ "destinataireId": 3, "montant": 10 }
# Attendu : 400 "Transaction bloquée : yahya figure sur la liste OFAC"
```

### Scénarios Transactions FAILED (visibles dans historique admin)

| Scénario | Corps | Erreur attendue |
|----------|-------|----------------|
| Solde insuffisant | `{ "montant": 999999 }` | "Solde insuffisant" |
| Auto-virement | `{ "destinataireId": <son_propre_id> }` | "Auto-virement interdit" |
| Destinataire OFAC | après blacklist | "figure sur la liste OFAC" |
| Montant invalide | `{ "montant": -100 }` | "Montant invalide" |

### Test QR Paiement (deux comptes requis)

```
Compte A (magasin) :
  → Ouvrir QR Pay → CRÉER QR → entrer 100 MAD → générer
  → Copier le qrToken (JWT commençant par eyJ...)

Compte B (acheteur, autre navigateur) :
  → Ouvrir QR Pay → PAYER VIA QR → coller le token
  → VÉRIFIER QR → voir aperçu
  → CONFIRMER → transaction COMPLETED
```

---

## 12. Variables d'Environnement

### application.properties

```properties
# PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/bankapp
spring.datasource.username=bankuser
spring.datasource.password=BankPass2026!
spring.jpa.hibernate.ddl-auto=update

# JWT
jwt.secret=c2VjcmV0LWJhbmstYXBwLWtleS0yMDI2LXN1cGVyLXNlY3VyZS1taW5pbXVtLTI1Ni1iaXRz
jwt.access-token-expiration=900000       # 15 minutes
jwt.refresh-token-expiration=604800000   # 7 jours

# Admin
admin.email=${ADMIN_EMAIL:admin@bank.com}
admin.password=${ADMIN_PASSWORD:Admin123!}
admin.username=${ADMIN_USERNAME:admin}
```

> **Production** : Changer `jwt.secret` pour une clé aléatoire 256+ bits. Ne jamais committer les credentials en clair.

---

## Résumé des Choix Techniques

| Décision | Pourquoi |
|----------|---------|
| `Isolation.SERIALIZABLE` | Niveau d'isolation le plus strict — empêche les race conditions sur les soldes |
| `REQUIRES_NEW` dans TransactionEchecService | Spring AOP n'intercepte pas les appels intra-classe — bean séparé obligatoire pour REQUIRES_NEW |
| preAuthToken type=PRE_AUTH | Empêche le recyclage d'un accessToken valide pour bypasser le 2FA |
| SHA-256 des refresh tokens | Si la DB est compromise, les tokens bruts ne sont pas exposés |
| Message d'erreur générique au login | Empêche l'énumération des emails (user enumeration attack) |
| `Propagation.REQUIRES_NEW` pour les échecs | Commit indépendant même si la transaction principale rollback |
| PostgreSQL > H2 | Persistance entre redémarrages + support SERIALIZABLE réel + comportement production |
