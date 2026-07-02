# architecte

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— ARCHITECTE

# Livrable Architecture — Travel Companion App

---

## 1. VUE D'ENSEMBLE DE L'ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                  │
│                                                                 │
│   ┌─────────────────────┐      ┌──────────────────────────┐    │
│   │   App Mobile        │      │   Web App (futur)        │    │
│   │   iOS / Android     │      │   Back-office B2B        │    │
│   │   (React Native)    │      │   (hors MVP)             │    │
│   └──────────┬──────────┘      └──────────────────────────┘    │
└──────────────┼──────────────────────────────────────────────────┘
               │ HTTPS / REST + WebSocket (notifications)
┌──────────────▼──────────────────────────────────────────────────┐
│                        API GATEWAY                              │
│   Authentification · Rate Limiting · Routing · Versioning       │
└──────┬────────────┬────────────┬────────────┬───────────────────┘
       │            │            │            │
┌──────▼──┐  ┌──────▼──┐  ┌─────▼───┐  ┌────▼────────┐
│  Auth   │  │ Voyage  │  │ Outils  │  │  Souvenirs  │
│ Service │  │ Service │  │ Service │  │  Service    │
└──────┬──┘  └──────┬──┘  └─────┬───┘  └────┬────────┘
       │            │            │            │
┌──────▼────────────▼────────────▼────────────▼────────┐
│                   BASE DE DONNÉES                     │
│              PostgreSQL (données primaires)           │
│              + Redis (cache / sessions)               │
└───────────────────────────────────────────────────────┘
       │            │                        │
┌──────▼──┐  ┌──────▼───────────┐   ┌───────▼────────┐
│  Auth   │  │  Object Storage  │   │  Push Notif.   │
│Provider │  │  (documents,     │   │  Service       │
│(OAuth)  │  │   médias)        │   │  (FCM / APNs)  │
└─────────┘  └──────────────────┘   └────────────────┘
                        │
             ┌──────────▼──────────┐
             │  APIs Tierces       │
             │  · Traduction       │
             │  · Taux de change   │
             │  · Géolocalisation  │
             └─────────────────────┘
```

---

## 2. ARCHITECTURE MOBILE — COUCHES INTERNES

```
┌────────────────────────────────────────────────┐
│               APPLICATION MOBILE               │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │           UI LAYER (Composants)          │  │
│  │  Écrans · Composants réutilisables       │  │
│  └──────────────┬───────────────────────────┘  │
│                 │                              │
│  ┌──────────────▼───────────────────────────┐  │
│  │         STATE MANAGEMENT                 │  │
│  │  État global · Contexte voyage actif     │  │
│  └──────────────┬───────────────────────────┘  │
│                 │                              │
│  ┌──────────────▼───────────────────────────┐  │
│  │         SERVICE LAYER                    │  │
│  │  Appels API · Logique métier client      │  │
│  └──────────┬──────────────────────────────┘   │
│             │              │                   │
│  ┌──────────▼───┐  ┌───────▼──────────────┐    │
│  │  API Client  │  │   LOCAL STORAGE      │    │
│  │  (connecté)  │  │   SQLite / AsyncSto. │    │
│  └──────────────┘  │   (offline cache)    │    │
│                    └──────────────────────┘    │
└────────────────────────────────────────────────┘
```

---

## 3. MODULES ET RESPONSABILITÉS

### 3.1 Backend — Services

| Module | Responsabilités | Technologie recommandée |
|---|---|---|
| **Auth Service** | Inscription, connexion, gestion tokens JWT, OAuth social, refresh tokens | Node.js + JWT + OAuth2 |
| **Voyage Service** | CRUD voyages, itinéraires, étapes, participants, invitations, statuts | Node.js / Python |
| **Document Service** | Upload, stockage, association catégorie, génération URL signées, quota | Node.js + S3-compatible |
| **Outils Service** | Proxy vers APIs tierces (traduction, devises, fuseaux), cache des taux | Node.js |
| **Souvenirs Service** | Gestion médias, albums, génération liens partage, accès invités | Node.js |
| **Notification Service** | Planification rappels, envoi push FCM/APNs, préférences utilisateur | Node.js + job queue |
| **API Gateway** | Routage, auth middleware, rate limiting, versioning `/v1/` | Nginx / Kong |

### 3.2 Mobile — Modules

| Module | Responsabilités |
|---|---|
| **Auth Module** | Formulaires login/register, gestion tokens locaux, session persistante |
| **Dashboard Module** | Liste voyages, voyage actif en avant, navigation contextuelle |
| **Voyage Module** | Itinéraire jour/j, détail étape, formulaires création/édition |
| **Document Module** | Picker fichiers, upload, liste catégorisée, viewer offline |
| **Outils Module** | Traducteur, convertisseur, horloge — tous contextualisés via voyage actif |
| **Souvenirs Module** | Galerie, ajout photo/note/lieu, génération et partage de lien album |
| **Offline Module** | Stratégie de cache, synchronisation, indicateurs visuels offline |
| **Notification Module** | Enregistrement device token, réception et affichage notifications |

---

## 4. DÉPENDANCES ENTRE MODULES

```
Auth Service
    └──► tous les services (middleware obligatoire)

Voyage Service
    ├──► Document Service (association documents ↔ étapes)
    ├──► Notification Service (planification rappels sur étapes)
    └──► Souvenirs Service (album lié à un voyage)

Outils Service
    └──► Voyage Service (lecture destination/devise du voyage actif)

Souvenirs Service
    └──► Document Service (stockage médias partagé)

Notification Service
    └──► Voyage Service (données étapes pour calcul timing)
```

**Règle de dépendance** : aucun service ne dépend du module Outils. Les APIs tierces sont encapsulées dans l'Outils Service uniquement — zéro appel direct depuis le mobile vers DeepL, Fixer.io, etc.

---

## 5. SCHÉMA DE DONNÉES

### Entités principales

```
USER
├── id (UUID)
├── email (unique)
├── password_hash
├── display_name
├── avatar_url
├── preferred_locale
├── preferred_currency
└── created_at

VOYAGE
├── id (UUID)
├── owner_id → USER
├── title
├── destination_label
├── destination_coords (lat/lng)
├── date_start
├── date_end
├── status (upcoming / active / past)
├── cover_image_url
└── created_at

VOYAGE_PARTICIPANT
├── voyage_id → VOYAGE
├── user_id → USER
├── role (admin / member / viewer)
└── invited_at

ETAPE
├── id (UUID)
├── voyage_id → VOYAGE
├── title
├── type (vol / hébergement / activité / transport / autre)
├── date_time (avec timezone)
├── duration_minutes
├── location_label
├── location_coords (lat/lng)
├── notes (text)
└── sort_order

DOCUMENT
├── id (UUID)
├── voyage_id → VOYAGE
├── etape_id → ETAPE (nullable)
├── uploader_id → USER
├── filename
├── storage_key (chemin objet S3)
├── mime_type
├── size_bytes
├── category (vol / hébergement / activité / autre)
└── uploaded_at

SOUVENIR
├── id (UUID)
├── voyage_id → VOYAGE
├── author_id → USER
├── media_url
├── media_type (photo / note / audio)
├── caption (text)
├── location_label
├── location_coords (lat/lng)
├── taken_at
└── created_at

ALBUM_PARTAGE
├── id (UUID)
├── voyage_id → VOYAGE
├── created_by → USER
├── share_token (unique, aléatoire)
├── expires_at (nullable)
└── created_at

NOTIFICATION_PREFS
├── user_id → USER
├── voyage_id → VOYAGE
├── enabled (bool)
├── delay_24h (bool)
├── delay_2h (bool)
└── custom_delay_minutes

CACHE_TAUX_CHANGE
├── from_currency
├── to_currency
├── rate
└── fetched_at
```

---

## 6. STRATÉGIE OFFLINE

```
┌────────────────────────────────────────────────────────┐
│              STRATÉGIE CACHE OFFLINE                   │
│                                                        │
│  TOUJOURS DISPONIBLE OFFLINE (synchronisé au login     │
│  et à chaque modification) :                           │
│  · Itinéraire du voyage actif (étapes, horaires)       │
│  · Documents importés (fichiers locaux)                │
│  · Derniers taux de change connus (horodatés)          │
│  · Dernières traductions effectuées (LRU cache 50)     │
│  · Profil utilisateur                                  │
│                                                        │
│  NÉCESSITE CONNEXION :                                 │
│  · Nouvelles traductions                               │
│  · Taux de change temps réel                           │
│  · Upload de documents / souvenirs                     │
│  · Partage d'albums                                    │
│  · Invitations participants                            │
│                                                        │
│  INDICATEUR VISUEL :                                   │
│  Bandeau discret "Mode hors ligne" + icône ☁️✕         │
│  sur chaque élément non disponible offline             │
└────────────────────────────────────────────────────────┘
```

**Mécanisme** : SQLite embarqué pour données structurées + système de fichiers local pour documents. Synchronisation en arrière-plan dès retour réseau (conflict resolution : last-write-wins sur les données non-critiques, merge manuel sur itinéraire si conflit).

---

## 7. INTÉGRATIONS TIERCES

| Service | Usage | Alternative de repli |
|---|---|---|
| **DeepL / Google Translate API** | Traduction texte + audio TTS | Bascule automatique si quota dépassé |
| **Fixer.io / Open Exchange Rates** | Taux de change en temps réel | Cache local 24h en cas d'indisponibilité |
| **Google Maps / Mapbox** | Affichage lieu, lien navigation | Lien deep link natif Maps/Plans en fallback |
| **Firebase Cloud Messaging + APNs** | Notifications push cross-platform | File de secours en base si échec push |
| **Auth0 / Supabase Auth** | OAuth social (Google, Apple) | Email/password natif obligatoire en parallèle |
| **AWS S3 / Cloudflare R2** | Stockage documents et médias | Multi-bucket avec redondance |

**Règle d'isolation** : toute intégration tierce passe par une interface abstraite (`TranslatorPort`, `CurrencyPort`) côté backend. Le remplacement d'un fournisseur n'impacte aucune couche applicative.

---

## 8. SÉCURITÉ — DÉCISIONS STRUCTURANTES

| Périmètre | Décision |
|---|---|
| **Authentification** | JWT access token (15 min) + refresh token (30 jours) en HttpOnly cookie |
| **Documents** | URLs signées à durée limitée (1h) pour accès S3 — jamais d'URL permanente publique |
| **Albums partagés** | Token opaque aléatoire (256 bits) — pas d'auth requise pour le visiteur mais accès en lecture seule |
| **Chiffrement** | TLS 1.3 obligatoire en transit · Documents chiffrés at-rest côté S3 |
| **Quota stockage** | Limite par utilisateur (ex. 2 Go MVP) appliquée côté backend, jamais côté client seul |
| **Rate limiting** | Par IP sur l'API Gateway + par user_id sur les endpoints sensibles (upload, traduction) |

---

## 9. RISQUES TECHNIQUES

| # | Risque | Niveau | Mitigation |
|---|---|---|---|
| R1 | **Dérive de complexité** — app "tout-en-un" → base de code ingérable | 🔴 Élevé | Architecture modulaire stricte, règle : un module = un domaine, revues d'architecture bi-mensuelles |
| R2 | **Expérience offline dégradée** — synchronisation conflictuelle | 🔴 Élevé | Définir dès le départ la stratégie conflict resolution, tester offline en CI avec profil réseau simulé |
| R3 | **Coût APIs tierces incontrôlé** — traduction/devises facturées à la requête | 🟠 Moyen | Cache agressif côté backend, quota par user/jour, alertes de coût sur dashboard cloud |
| R4 | **Stockage documents non borné** — utilisateurs uploadent sans limite | 🟠 Moyen | Quota 2 Go/user enforced backend + nettoyage automatique voyages >2 ans |
| R5 | **Sécurité liens partage albums** — token devinable ou réutilisé abusivement | 🟠 Moyen | Token 256 bits, expiration optionnelle, révocation à la demande, rate limit sur endpoint public |
| R6 | **Performance sur connexion lente** — voyageurs souvent sur réseau étranger bas débit | 🟠 Moyen | Images compressées à l'upload, pagination des listes, lazy loading documents |
| R7 | **Notifications fuseau horaire** — calcul décalé si timezone mal gérée | 🟡 Faible | Stocker toujours les horaires en UTC + timezone IANA de l'étape, calcul serveur uniquement |
| R8 | **Scalabilité future B2B** — architecture B2C non préparée au multi-tenant | 🟡 Faible | Introduire `tenant_id` nullable dès le MVP sur les entités Voyage et User, sans l'activer |

---

## 10. DÉCISIONS D'ARCHITECTURE DOCUMENTÉES

### DA-01 — Monorepo modulaire vs. microservices distribués
**Décision**

— ARCHITECTE

*(Reprise à partir de la section DA-01)*

---

### DA-01 — Monorepo modulaire vs. microservices distribués

**Décision** : Monorepo modulaire avec séparation logique des domaines, déploiement en services distincts uniquement si besoin avéré.

**Justification** : Le MVP ne justifie pas la complexité opérationnelle des microservices (service mesh, distributed tracing, latence inter-services). Un monorepo avec modules bien isolés permet de découper plus tard sans réécriture majeure.

**Conséquence acceptée** : Couplage de déploiement à court terme. Acceptable jusqu'à une charge de ~50 000 utilisateurs actifs.

---

### DA-02 — Backend unique vs. BFF (Backend for Frontend)

**Décision** : API REST unique versionnée (`/api/v1/`) exposée aux clients mobiles. Pas de BFF dédié pour le MVP.

**Justification** : Un seul client mobile consommateur. Le BFF apporte de la valeur quand plusieurs surfaces (web, mobile, partenaires) ont des besoins de payload radicalement différents — ce qui n'est pas le cas au démarrage.

**Conséquence acceptée** : Si un client web ou une API partenaire B2B est ajouté plus tard, un BFF ou une couche GraphQL pourra être introduit en surcouche sans casser l'API existante.

---

### DA-03 — Choix stack mobile : natif vs. cross-platform

**Décision** : React Native (Expo managed workflow) pour le MVP.

**Justification** : Cible iOS + Android simultanément avec une seule base de code. Le profil des fonctionnalités (offline storage, notifications, caméra, fichiers) est intégralement couvert par les APIs Expo. Le gain de time-to-market est décisif pour un MVP.

**Conséquence acceptée** : Performances légèrement inférieures au natif sur les animations complexes. Non-bloquant pour ce type d'application.

**Ligne rouge** : Si une fonctionnalité critique ne peut être implémentée sans module natif non supporté par Expo, passage en Expo bare workflow — pas de réécriture Swift/Kotlin.

---

### DA-04 — Gestion de l'état offline : SQLite embarqué

**Décision** : WatermelonDB (sur SQLite) pour la persistance locale structurée + Expo FileSystem pour les fichiers binaires (documents, médias).

**Justification** : WatermelonDB est conçu pour la synchronisation offline-first avec des performances adaptées aux listes larges. Il offre un mécanisme de sync serveur documenté. Alternative écartée : Redux Persist — insuffisant pour des données relationnelles complexes.

**Conséquence acceptée** : Courbe d'apprentissage plus raide que AsyncStorage. Compensée par la robustesse long-terme.

---

### DA-05 — Stratégie d'authentification

**Décision** : Supabase Auth (email/password + OAuth Google + OAuth Apple). JWT côté client, refresh token géré par Supabase SDK.

**Justification** : Supabase Auth couvre nativement les deux providers OAuth obligatoires (Apple requis pour App Store), gère le refresh automatiquement, et s'intègre directement avec le reste de la stack Supabase si choisie pour le backend. Auth0 reste une alternative viable mais introduit un coût supplémentaire à faible volume.

**Conséquence acceptée** : Dépendance à un fournisseur d'auth externe. Mitigée par l'abstraction derrière un `AuthPort` côté backend.

---

### DA-06 — Stockage fichiers : solution managée

**Décision** : Cloudflare R2 pour le stockage des documents et médias (compatible S3, sans frais d'egress).

**Justification** : Les documents et photos de voyage génèrent du trafic sortant important (consultations fréquentes). Cloudflare R2 élimine les frais d'egress contrairement à AWS S3, décisif pour la maîtrise des coûts à l'échelle.

**Conséquence acceptée** : Écosystème moins mature qu'AWS S3. Compensé par la compatibilité API S3 qui permet une migration sans changement de code applicatif.

---

## 11. ARCHITECTURE GLOBALE — SCHÉMA

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENTS MOBILES                              │
│               iOS (React Native / Expo)                             │
│               Android (React Native / Expo)                         │
│                                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  Auth    │ │ Voyage   │ │  Outils  │ │Souvenirs │ │ Offline  │ │
│  │  Module  │ │  Module  │ │  Module  │ │  Module  │ │  Module  │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       │            │            │             │            │        │
│  ┌────▼────────────▼────────────▼─────────────▼────────────▼─────┐ │
│  │              WatermelonDB (SQLite) + Expo FileSystem           │ │
│  │                    Cache local offline-first                   │ │
│  └────────────────────────────┬───────────────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────────┘
                                │ HTTPS / TLS 1.3
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                                 │
│              Rate limiting · Auth middleware · Versioning           │
│                         /api/v1/                                    │
└──────────┬──────────┬──────────┬──────────┬──────────┬─────────────┘
           │          │          │          │          │
           ▼          ▼          ▼          ▼          ▼
┌──────────────────────────────────────────────────────────────────┐
│                      BACKEND MODULAIRE                           │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────────┐  │
│  │   Auth    │ │  Voyage   │ │ Document  │ │  Notification  │  │
│  │  Service  │ │  Service  │ │  Service  │ │    Service     │  │
│  └───────────┘ └───────────┘ └───────────┘ └────────────────┘  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐                     │
│  │  Outils   │ │Souvenirs  │ │   Share   │                     │
│  │  Service  │ │  Service  │ │  Service  │                     │
│  │(adapters) │ └───────────┘ └───────────┘                     │
│  └───────────┘                                                  │
└──────┬───────────────┬───────────────┬──────────────────────────┘
       │               │               │
       ▼               ▼               ▼
┌────────────┐  ┌────────────┐  ┌──────────────────────────────┐
│  Supabase  │  │Cloudflare  │  │     SERVICES TIERS           │
│  Postgres  │  │    R2      │  │  DeepL / Google Translate    │
│  +Auth     │  │  (médias   │  │  Fixer.io / Open Exchange    │
│            │  │  +docs)    │  │  FCM + APNs (notifications)  │
└────────────┘  └────────────┘  │  Google Maps / Mapbox        │
                                └──────────────────────────────┘
```

---

## 12. DÉCOUPAGE DÉTAILLÉ DES MODULES BACKEND

### Auth Service
**Responsabilités** : Inscription, connexion, refresh token, déconnexion, gestion des sessions.
**Interfaces exposées** : `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`
**Dépendances** : Supabase Auth (fournisseur externe), `USER` table Postgres
**Règle** : Tout autre service consomme le JWT validé — aucun service ne re-vérifie les credentials.

---

### Voyage Service
**Responsabilités** : CRUD voyages, CRUD étapes, gestion des participants, calcul de statut (upcoming/active/past).
**Interfaces exposées** : `/voyages`, `/voyages/:id`, `/voyages/:id/etapes`, `/voyages/:id/participants`
**Dépendances** : `VOYAGE`, `ETAPE`, `VOYAGE_PARTICIPANT` tables · Notification Service (planification rappels à la création/modification d'étape)
**Règle** : Le statut du voyage est toujours calculé côté serveur — jamais côté client.

---

### Document Service
**Responsabilités** : Upload, catégorisation, association à un voyage/étape, génération d'URLs signées, quota utilisateur.
**Interfaces exposées** : `/voyages/:id/documents`, `/documents/:id/url` (URL signée), `DELETE /documents/:id`
**Dépendances** : Cloudflare R2 (stockage), `DOCUMENT` table · Voyage Service (vérification appartenance)
**Règle** : Aucune URL permanente publique n'est jamais générée. Toute URL signée expire dans 1h maximum.

---

### Outils Service
**Responsabilités** : Proxy vers APIs tierces de traduction et taux de change. Cache des réponses. Exposition d'une interface neutre vis-à-vis du fournisseur.
**Interfaces exposées** : `POST /tools/translate`, `GET /tools/currency?from=EUR&to=THB&amount=100`, `GET /tools/timezones?voyage_id=:id`
**Dépendances** : `TranslatorPort` → DeepL ou Google Translate · `CurrencyPort` → Fixer.io · `CACHE_TAUX_CHANGE` table · Voyage Service (lecture destination/devise du voyage actif)
**Règle critique** : Zéro appel direct depuis le mobile vers les APIs tierces. Tout passe par ce service. Le remplacement d'un fournisseur est transparent pour le reste du système.

---

### Souvenirs Service
**Responsabilités** : CRUD souvenirs (photo, note, audio), association à un voyage, gestion des albums partagés, génération de tokens de partage.
**Interfaces exposées** : `/voyages/:id/souvenirs`, `/voyages/:id/album`, `GET /shared/:token` (endpoint public, lecture seule)
**Dépendances** : Cloudflare R2 (médias) · `SOUVENIR`, `ALBUM_PARTAGE` tables · Voyage Service
**Règle** : L'endpoint `/shared/:token` est public (pas de JWT requis) mais retourne uniquement des données en lecture seule. Rate limiting renforcé sur cet endpoint.

---

### Notification Service
**Responsabilités** : Enregistrement des device tokens, planification des rappels, envoi via FCM/APNs, gestion des préférences par voyage.
**Interfaces exposées** : `POST /notifications/register`, `PUT /notifications/prefs/:voyage_id`, `POST /notifications/test`
**Dépendances** : Firebase Cloud Messaging + APNs · `NOTIFICATION_PREFS` table · Voyage Service (données étapes pour calcul timing)
**Règle** : Tous les timings de notification sont calculés côté serveur en UTC puis convertis selon la timezone IANA de l'étape. Aucun calcul de timezone côté client.

---

### Share Service
**Responsabilités** : Génération et révocation de tokens de partage d'albums, validation d'accès, accès en lecture aux souvenirs via token.
**Note** : Ce service est volontairement léger. Il délègue au Souvenirs Service pour les données. Sa seule responsabilité est la gestion du cycle de vie des tokens.
**Dépendances** : `ALBUM_PARTAGE` table · Souvenirs Service

---

## 13. MATRICE DE DÉPENDANCES COMPLÈTE

```
                    AUTH  VOYAGE  DOC  OUTILS  SOUVENIR  NOTIF  SHARE
Auth Service         —     non    non   non      non      non    non
Voyage Service       ◄     —      non   non      non      ►      non
Document Service     ◄     ◄      —     non      non      non    non
Outils Service       ◄     ◄(R)   non   —        non      non    non
Souvenirs Service    ◄     ◄      ◄(R2) non      —        non    ◄
Notification Service ◄     ◄(R)   non   non      non      —      non
Share Service        non   non    non   non      ◄        non    —

Légende :
◄     = dépend de (consomme)
►     = est consommé par
◄(R)  = lecture seule (pas d'écriture cross-service)
◄(R2) = accès Cloudflare R2 partagé (via référence storage_key)
```

**Principe d'isolation** : Aucune communication synchrone de service à service n'est autorisée sauf via interfaces définies. Les communications asynchrones (ex. planification notification après création d'étape) passent par une file légère (Supabase Edge Functions ou job queue simple au MVP).

---

## 14. CONSIDÉRATIONS POUR LA SCALABILITÉ B2B FUTURE

Bien que le MVP soit centré B2C, les décisions suivantes sont prises dès maintenant pour ne pas bloquer l'évolution :

| Décision préventive | Impact MVP | Bénéfice futur |
|---|---|---|
| `tenant_id` nullable sur `VOYAGE` et `USER` | Aucun (champ ignoré) | Multi-tenant agences sans migration de schéma |
| Abstraction `AuthPort` | Légère complexité | Remplacement Auth0/Supabase ou ajout SSO entreprise |
| Isolation Outils Service | Overhead minimal | Quotas différenciés par tenant, facturation à l'usage |
| Token partage opaque | Implémentation standard | Base pour futures permissions granulaires B2B |
| Modules backend découplés | Organisation du code | Extraction en microservice si charge l'exige |

---

## Transmission à l'équipe

**À l'attention du Tech Lead**

Le livrable ci-dessus constitue la référence d'architecture pour le MVP Travel Companion. Voici les points d'attention prioritaires pour la mise en œuvre :

**🔴 Décisions à valider immédiatement avant le premier sprint**

1. **Choix Supabase comme fondation** : Supabase couvre Auth + Postgres + potentiellement Edge Functions. Si le Tech Lead a des réserves sur la dépendance à ce fournisseur, la décision DA-05 doit être revisitée avant que le code d'infrastructure commence.

2. **WatermelonDB vs. alternative offline** : La complexité de WatermelonDB est réelle. Si l'

— ARCHITECTE

*(Continuation exacte depuis le dernier mot produit)*

équipe mobile n'a pas d'expérience avec WatermelonDB, envisager SQLite via `expo-sqlite` + synchronisation manuelle simplifiée est une alternative valide pour le MVP. L'impact est une synchronisation moins sophistiquée mais une courbe d'apprentissage drastiquement réduite.

3. **Stratégie de cache des taux de change** : La décision de rafraîchir toutes les 4h est un compromis. Si Fixer.io impose des limites d'appels sur le plan gratuit, ce TTL devra être ajusté. Vérifier les quotas API avant le premier sprint outils.

4. **Limite de stockage par utilisateur** : Le schéma prévoit des fichiers jusqu'à 20 Mo unitaires (contrainte PO) mais aucune limite globale par utilisateur n'est définie dans ce livrable. Le Tech Lead doit définir un quota total (recommandation : 500 Mo / utilisateur MVP) et implémenter la vérification côté Document Service avant upload.

**🟡 Points de surveillance pendant le développement**

5. **Row Level Security Supabase** : Les politiques RLS sont la ligne de défense principale pour l'isolation des données entre voyageurs. Elles doivent être implémentées et testées dès la mise en place du schéma, pas en fin de développement.

6. **Gestion des timezones IANA** : Le Notification Service porte toute la logique de conversion timezone. S'assurer que la bibliothèque choisie (recommandation : `date-fns-tz` ou `luxon`) est cohérente entre backend et mobile. Une divergence de bibliothèques est une source classique de bugs de rappel.

7. **Endpoint public `/shared/:token`** : Cet endpoint ne nécessite pas de JWT mais est exposé sur internet. Le rate limiting doit être configuré de manière restrictive dès le départ (recommandation : 20 req/min par IP) pour éviter l'énumération de tokens.

8. **Taille des bundles mobile hors ligne** : Le cache offline (WatermelonDB ou SQLite) combiné aux documents stockés localement peut atteindre des tailles significatives. Définir une politique d'éviction du cache local (ex. : purge automatique des voyages passés depuis plus de 90 jours, hors documents explicitement marqués "conserver").

**🟢 Ce qui est délibérément simple et doit le rester**

9. **Share Service** : Ce service est volontairement minimaliste. Ne pas être tenté d'y ajouter de la logique métier. Toute évolution de permissions doit rester dans Souvenirs Service ou dans une future couche de permissions dédiée.

10. **Outils Service comme proxy pur** : La valeur de ce service est son rôle de façade. Ne jamais y implémenter de logique métier complexe. Si un fournisseur tiers ajoute des fonctionnalités, les évaluer via une extension des ports, pas par couplage direct.

---

**Ordre de construction recommandé pour les sprints**

```
Sprint 0 (Fondations)
  └── Infrastructure Supabase · Schéma DB + RLS · Auth Service · CI/CD baseline

Sprint 1 (Cœur voyage)
  └── Voyage Service (CRUD) · App mobile shell + navigation · Vue itinéraire (online)

Sprint 2 (Documents + Offline)
  └── Document Service · Cloudflare R2 · Stratégie cache offline · Sync mobile

Sprint 3 (Outils utilitaires)
  └── Outils Service · Traducteur · Convertisseur · Fuseaux horaires

Sprint 4 (Notifications + Souvenirs)
  └── Notification Service · FCM/APNs · Souvenirs Service · Galerie mobile

Sprint 5 (Partage + Polish)
  └── Share Service · Endpoint public album · Tests end-to-end · Préparation stores
```

---

**Référentiel des décisions d'architecture (récapitulatif)**

| ID | Décision | Statut |
|---|---|---|
| DA-01 | Architecture modulaire monorepo, backend découplé par domaine | ✅ Validée |
| DA-02 | React Native (Expo) + TypeScript pour le mobile | ✅ Validée |
| DA-03 | WatermelonDB pour la persistance offline (à confirmer par Tech Lead) | ⚠️ À confirmer |
| DA-04 | Cloudflare R2 pour le stockage fichiers et médias | ✅ Validée |
| DA-05 | Supabase comme fondation Auth + Postgres | ⚠️ À confirmer |
| DA-06 | Proxy mandatory via Outils Service — zéro appel direct mobile → tiers | ✅ Validée |
| DA-07 | URLs signées uniquement pour documents (TTL 1h max) | ✅ Validée |
| DA-08 | `tenant_id` nullable dès le MVP pour évolution B2B | ✅ Validée |
| DA-09 | Timezones IANA, calcul serveur exclusivement | ✅ Validée |
| DA-10 | Endpoint `/shared/:token` public en lecture seule avec rate limiting renforcé | ✅ Validée |

---

*Livrable Architecture — Travel Companion App MVP — Complété.*
*Prêt pour transmission au Tech Lead et à l'équipe de développement.*