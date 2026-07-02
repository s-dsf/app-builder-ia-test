# documentation_manager

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— DOCUMENTATION MANAGER

# Documentation Projet — Travel Companion App
## Version 1.0 · MVP B2C · Livrable Documentation Manager

---

## SOMMAIRE

1. [Fiche Projet Synthétique](#1-fiche-projet-synthétique)
2. [Glossaire Métier](#2-glossaire-métier)
3. [Architecture de la Documentation](#3-architecture-de-la-documentation)
4. [Documentation Technique](#4-documentation-technique)
5. [Documentation Fonctionnelle](#5-documentation-fonctionnelle)
6. [Journal des Décisions (ADR)](#6-journal-des-décisions-adr)
7. [Release Notes — MVP v1.0](#7-release-notes--mvp-v10)
8. [Matrice de Couverture](#8-matrice-de-couverture)
9. [Risques Documentés](#9-risques-documentés)
10. [Plan de Documentation — Prochains Cycles](#10-plan-de-documentation--prochains-cycles)

---

## 1. FICHE PROJET SYNTHÉTIQUE

| Champ | Valeur |
|---|---|
| **Nom du projet** | Travel Companion App |
| **Version documentée** | MVP v1.0 — B2C uniquement |
| **Statut** | En développement |
| **Durée estimée MVP** | 12 semaines |
| **Équipe cible** | 2 développeurs fullstack |
| **Canaux de distribution** | B2C (MVP) · Agences / Travel Planners (hors périmètre MVP) |
| **Plateformes cibles** | iOS · Android (React Native) · Web companion (Next.js 14) |
| **Date de création doc** | Cycle 1 |

### Problème résolu

Les voyageurs jonglent entre de multiples applications et documents pour gérer un voyage (confirmations email, PDF de réservation, traduction, conversion de devises). Cette fragmentation génère de la friction, du stress et une perte d'information.

### Proposition de valeur

Un **carnet de voyage numérique centralisé** permettant de : consulter son itinéraire, accéder à ses documents, utiliser des outils utilitaires contextualisés (traduction, devises, fuseaux horaires), et partager ses souvenirs avec ses proches.

### Hors périmètre MVP — Explicitement documenté

> ⚠️ Les éléments suivants ont été **unanimement exclus** de ce premier cycle par le Product Owner, l'Architecte et le QA Engineer. Leur exclusion est une décision structurante, pas un oubli.

| Élément exclu | Raison documentée |
|---|---|
| Back-office agence / travel planner | Modèle d'usage fondamentalement différent, traité en cycle 2 |
| White label / marque blanche | Complexité technique et commerciale découplée du MVP |
| Interface web companion complète | Priorité mobile ; le web couvre uniquement le partage public et le dashboard |
| Page publique album (couverture QA minimale) | Fonctionnalité secondaire, risque limité en V1 |

---

## 2. GLOSSAIRE MÉTIER

> Ce glossaire est la **référence sémantique du projet**. Tout terme technique ou métier utilisé dans le code, les PR, les tickets ou les communications d'équipe doit s'aligner sur ces définitions.

| Terme | Définition | Utilisé dans |
|---|---|---|
| **Voyage** (`trip`) | Espace central regroupant toutes les données d'un déplacement : itinéraire, documents, participants, souvenirs. Identifié par un `trip_id`. | BDD, API, Mobile, Web |
| **Étape** (`step`) | Élément chronologique d'un itinéraire (vol, hébergement, activité, restaurant, transport, autre). Appartient à un voyage. | BDD, API, Mobile |
| **Voyage actif** (`active trip`) | Voyage dont la date en cours est comprise entre `start_date` et `end_date`. Contexte permanent de l'expérience utilisateur. | UX, Mobile, State |
| **Participant** | Utilisateur invité à accéder à un voyage. Peut avoir un rôle `admin`, `editor` ou `viewer`. | BDD, API, Mobile |
| **Document** | Fichier (PDF, JPG, PNG, max 20 Mo) importé dans un voyage et catégorisé (vol / hébergement / activité / autre). | BDD, API, Mobile |
| **Souvenir** (`memory`) | Photo, note ou lieu ajouté volontairement par un participant pendant ou après le voyage. | BDD, API, Mobile |
| **Album partagé** (`shared_album`) | Collection de souvenirs rendue accessible via un lien public à des personnes non inscrites. | BDD, API, Web, Mobile |
| **Outils utilitaires** | Traducteur, convertisseur de devises, horloge/fuseaux horaires — contextualisés automatiquement par le voyage actif. | Mobile, API (proxy) |
| **Mode offline** | État de l'application lorsque la connexion réseau est absente. Les données en cache restent consultables ; les actions de création sont mises en file d'attente. | Mobile, Architecture |
| **RLS** | Row Level Security — mécanisme PostgreSQL natif Supabase garantissant qu'un utilisateur ne peut accéder qu'aux données qui lui appartiennent ou qui lui sont partagées. | BDD, Sécurité |
| **Travel Planner** | Professionnel indépendant préparant des voyages pour des clients. Persona B2B identifié, hors périmètre MVP. | Product, Roadmap |
| **Agence de voyages** | Structure professionnelle créant des voyages clé en main pour ses clients. Persona B2B identifié, hors périmètre MVP. | Product, Roadmap |
| **Statut voyage** | État calculé d'un voyage : `upcoming` (à venir) / `active` (en cours) / `past` (passé). | BDD, Mobile, UX |

---

## 3. ARCHITECTURE DE LA DOCUMENTATION

### 3.1 Carte des livrables produits et leur relation

```
VISION PROJET (porteur)
        │
        ▼
┌───────────────────┐
│   PRODUCT OWNER   │ → Personas, User Stories, Critères d'acceptation
│   (référence      │   Périmètre MVP, Priorités MoSCoW
│    fonctionnelle) │
└────────┬──────────┘
         │ nourrit
         ▼
┌───────────────────┐     ┌─────────────────────┐
│   UX DESIGNER     │────►│   ARCHITECTE        │
│   Parcours, IA,   │     │   Stack, Services,  │
│   Principes UX    │     │   Schéma données    │
└────────┬──────────┘     └──────────┬──────────┘
         │                           │
         └──────────┬────────────────┘
                    │ implémentés par
          ┌─────────┴──────────┐
          │                    │
┌─────────▼──────┐   ┌─────────▼──────────┐
│  TECH LEAD     │   │  FRONTEND ENGINEER │
│  Plan 12 sem., │   │  Next.js 14, DS,   │
│  Blocs, Tests  │   │  Composants, Hooks │
└─────────┬──────┘   └─────────┬──────────┘
          │                    │
          └──────────┬─────────┘
                     │
           ┌─────────▼──────────┐
           │  BACKEND ENGINEER  │
           │  Supabase, SQL,    │
           │  API Routes, RLS   │
           └─────────┬──────────┘
                     │ validé par
           ┌─────────▼──────────┐
           │    QA ENGINEER     │
           │  Stratégie tests,  │
           │  Cas de validation │
           └─────────┬──────────┘
                     │ documenté par
           ┌─────────▼──────────┐
           │ DOCUMENTATION MGR  │← vous êtes ici
           │ Mémoire collective │
           └────────────────────┘
```

### 3.2 Sources de vérité par domaine

| Domaine | Source de vérité | Agent responsable |
|---|---|---|
| Priorités fonctionnelles | User Stories + critères d'acceptation | Product Owner |
| Parcours utilisateurs | Arborescence + flux UX | UX Designer |
| Contrats API | Endpoints documentés dans le livrable Backend | Backend Engineer |
| Schéma de données | Schéma SQL Supabase | Backend Engineer |
| Composants UI | Design System + structure dossiers | Frontend Engineer |
| Découpage temporel | Blocs 0–9, estimations semaines | Tech Lead |
| Critères de validation | Cas de tests QA | QA Engineer |

---

## 4. DOCUMENTATION TECHNIQUE

### 4.1 Stack Technique — Référence Officielle

```
┌─────────────────────────────────────────────────────────┐
│                    STACK MVP v1.0                       │
├─────────────────┬───────────────────────────────────────┤
│ Mobile          │ React Native (iOS + Android)           │
│ Web companion   │ Next.js 14 (App Router)               │
│ Backend         │ Next.js API Routes                    │
│ Base de données │ Supabase PostgreSQL                   │
│ Auth            │ Supabase Auth (JWT + OAuth)           │
│ Storage fichiers│ Supabase Storage                      │
│ Cache           │ Supabase Edge Cache + in-memory       │
│ Push notif.     │ FCM (Android) + APNs (iOS)            │
│ Traduction      │ DeepL API (encapsulée serveur)        │
│ Devises         │ Fixer.io / Open Exchange Rates        │
│ Fuseaux horaires│ Timezone API                          │
│ CI/CD           │ GitHub Actions                        │
│ Tests mobile E2E│ Detox                                 │
│ Tests unitaires │ Jest + React Testing Library          │
│ Tests API       │ Jest + Supertest                      │
│ Tests perf      │ k6                                    │
└─────────────────┴───────────────────────────────────────┘
```

### 4.2 Référence des Modules

#### Backend — Services

| Module | Responsabilité principale | Route de base |
|---|---|---|
| Auth | Inscription, connexion, tokens, OAuth | `/api/auth/*` |
| Voyage | CRUD voyages, itinéraires, étapes, participants | `/api/trips/*` |
| Document | Upload, stockage, catégorisation, URL signées | `/api/documents/*` |
| Outils | Proxy traduction, devises, fuseaux (APIs tierces) | `/api/tools/*` |
| Souvenirs | Médias, albums, liens de partage public | `/api/memories/*` |
| Notification | Planification rappels, push FCM/APNs | `/api/notifications/*` |

> **Règle absolue** : Aucun appel aux APIs tierces (DeepL, Fixer.io, Timezone) ne s'effectue depuis le client. Le module Outils est le seul point d'entrée autorisé.

#### Mobile — Modules

| Module | Responsabilité | Dépendances |
|---|---|---|
| Auth Module | Formulaires, tokens locaux, session | — |
| Dashboard Module | Liste voyages, voyage actif | Auth, Voyage |
| Voyage Module | Itinéraire, étapes, formulaires | Auth |
| Document Module | Picker, upload, viewer offline | Voyage |
| Outils Module | Traducteur, convertisseur, horloge | Voyage (contexte) |
| Souvenirs Module | Galerie, ajout, partage | Voyage, Document |
| Offline Module | Cache, sync, indicateurs visuels | Tous les modules |
| Notification Module | Token device, réception, affichage | Voyage, Auth |

### 4.3 Schéma de Données — Entités Principales

| Table | Rôle | Clé primaire | Relations principales |
|---|---|---|---|
| `profiles` | Extension du profil utilisateur | `id` (UUID, ref. `auth.users`) | 1:1 avec `auth.users` |
| `trips` | Voyage (entité centrale) | `id` UUID | Appartient à `profiles` (owner) |
| `trip_participants` | Membres d'un voyage | `id` UUID | N:M `trips` ↔ `auth.users` |
| `trip_steps` | Étapes de l'itinéraire | `id` UUID | N:1 `trips` |
| `documents` | Fichiers importés | `id` UUID | N:1 `trips`, N:M `trip_steps` |
| `memories` | Souvenirs (photo/note/lieu) | `id` UUID | N:1 `trips` |
| `memory_media` | Médias d'un souvenir | `id` UUID | N:1 `memories` |
| `shared_albums` | Albums publics partageables | `id` UUID | N:1 `trips` |
| `notifications` | Rappels et alertes | `id` UUID | N:1 `auth.users` |

### 4.4 Règles de Sécurité (RLS) — Principes Documentés

| Règle | Portée | Justification |
|---|---|---|
| Un utilisateur ne peut lire que ses propres voyages OU les voyages où il est participant accepté | `trips`, `trip_steps`, `documents` | Isolation stricte des données voyageurs |
| Seul le créateur (`owner_id`) peut supprimer un voyage | `trips` | Prévention des suppressions accidentelles par participants |
| Un album partagé est lisible sans authentification via son token | `shared_albums`, `memories` | Permet le partage avec des non-inscrits |
| Les fichiers dans Supabase Storage sont accessibles via URL signées uniquement | Storage | Aucun fichier n'est public par défaut |
| Aucune clé API tierce n'est exposée côté client | API Routes | Sécurité des secrets d'intégration |

### 4.5 Stratégie Offline — Résumé Documenté

| Donnée | Disponible offline | Mécanisme |
|---|---|---|
| Itinéraire du voyage actif | ✅ Oui | Cache SQLite local (lecture seule) |
| Documents importés | ✅ Oui | Cache local au téléchargement |
| Traducteur | ⚠️ Partiel | Message informatif + pack hors ligne optionnel |
| Convertisseur de devises | ⚠️ Partiel | Dernier taux connu affiché avec horodatage |
| Création d'étapes/souvenirs | 🔄 Différé | File d'attente de synchronisation |
| Notifications push | ❌ Non disponible | Nécessite connexion active |

### 4.6 Architecture de Navigation Mobile

```
ROOT NAVIGATOR
├── ONBOARDING STACK (non authentifié)
│   ├── Écran valeur proposition
│   ├── Inscription
│   └── Connexion
│
└── APP NAVIGATOR (authentifié)
    ├── BOTTOM TABS
    │   ├── Dashboard (home)
    │   ├── Voyage actif

— DOCUMENTATION MANAGER

*(Continuation exacte depuis le dernier mot produit)*

```
    │   ├── Outils
    │   └── Paramètres
    │
    ├── VOYAGE STACK (depuis Dashboard)
    │   ├── Vue voyage (tabs internes)
    │   │   ├── Itinéraire
    │   │   ├── Documents
    │   │   ├── Souvenirs
    │   │   └── Participants
    │   ├── Détail étape
    │   ├── Formulaire création/édition étape
    │   ├── Formulaire création voyage
    │   └── Import document
    │
    └── MODAUX GLOBAUX
        ├── Traducteur (bottom sheet persistant)
        ├── Convertisseur (bottom sheet)
        └── Horloge / Fuseaux (bottom sheet)
```

---

## 5. PLAN D'EXÉCUTION — RÉFÉRENCE SYNTHÉTIQUE

### 5.1 Calendrier des 12 semaines

| Bloc | Semaine(s) | Contenu | Dépendances |
|---|---|---|---|
| **Bloc 0** | S1 | Infrastructure & Fondations | — |
| **Bloc 1** | S2 | Authentification | Bloc 0 |
| **Bloc 2** | S3–S4 | Gestion des voyages & Itinéraire | Bloc 1 |
| **Bloc 3** | S5 | Gestion des documents | Bloc 2 |
| **Bloc 4** | S6 | Outils utilitaires | Bloc 2 |
| **Bloc 5** | S7–S8 | Souvenirs & Partage | Bloc 3 |
| **Bloc 6** | S9 | Notifications & Rappels | Bloc 2 |
| **Bloc 7** | S10 | Offline & Synchronisation | Blocs 2, 3, 5 |
| **Bloc 8** | S11 | UX Polish & États vides | Blocs 1–6 |
| **Bloc 9** | S12 | Tests, QA & Préparation Stores | Tous les blocs |

### 5.2 Dépendances critiques entre blocs

```
Bloc 0 ──► Bloc 1 ──► Bloc 2 ──┬──► Bloc 3 ──► Bloc 5
                                ├──► Bloc 4
                                ├──► Bloc 6
                                └──► Bloc 7 (partiel)

Blocs 3 + 5 ──► Bloc 7 (complet)
Tous les blocs ──► Bloc 8 ──► Bloc 9
```

**Point de vigilance critique** : Le Bloc 0 est bloquant absolu. Aucun développement parallèle n'est possible avant sa validation complète. Les critères de sortie du Bloc 0 doivent être validés en premier.

### 5.3 Équipe et répartition

| Profil | Blocs principaux | Charge estimée |
|---|---|---|
| Développeur 1 (fullstack, orientation backend) | Blocs 0–1 (BE), 2–6 (endpoints) | 12 semaines |
| Développeur 2 (fullstack, orientation frontend/mobile) | Blocs 0–1 (FE), 2–8 (mobile) | 12 semaines |

---

## 6. STRATÉGIE QA — RÉFÉRENCE DOCUMENTÉE

### 6.1 Pyramide de test

```
                    ┌─────────────┐
                    │   E2E / UAT │  10% — Parcours critiques (Detox)
                    ├─────────────┤
                │   Intégration   │  30% — Contrats API + RLS (Jest + Supertest)
                ├─────────────────┤
            │     Tests unitaires │  60% — Logique métier, utils, stores (Jest + RTL)
            └─────────────────────┘
```

### 6.2 Seuils de couverture minimaux

| Scope | Seuil |
|---|---|
| Logique métier (utils, stores) | ≥ 80% |
| Composants UI critiques | ≥ 70% |
| Endpoints API (couverture contrat) | 100% des endpoints documentés |
| Parcours E2E critiques (Must Have PO) | 100% |

### 6.3 Définition du "Done" — Rappel

Une User Story est **validée** lorsque :
- Tous les critères d'acceptation PO sont couverts par au moins un cas de test
- Tous les cas limites identifiés ont un cas de test associé
- Aucune anomalie de sévérité **Critique** ou **Haute** ouverte
- Tests unitaires et d'intégration passants en CI
- Validation manuelle sur device physique iOS **ET** Android

### 6.4 Modules couverts par la QA

| Module | Priorité QA | Types de tests |
|---|---|---|
| F — Authentification & Onboarding | Must Have | Unitaire, Intégration, E2E |
| A — Gestion voyage & Itinéraire | Must Have | Unitaire, Intégration, E2E |
| C — Documents de voyage | Must Have | Intégration, Sécurité, E2E |
| B — Outils utilitaires | Must Have | Unitaire, Intégration |
| D — Souvenirs & Partage | Should Have | Intégration, E2E |
| E — Notifications & Rappels | Should Have | Intégration, E2E |
| G — Offline & Synchronisation | Must Have | E2E, Simulation réseau |

---

## 7. GLOSSAIRE DU PROJET

| Terme | Définition | Domaine |
|---|---|---|
| **MVP** | Minimum Viable Product — périmètre B2C du premier cycle de développement, excluant toutes fonctionnalités B2B | Produit, Général |
| **B2C** | Business to Consumer — canal de distribution direct vers le voyageur final | Produit, Distribution |
| **B2B** | Business to Business — canal de distribution via agences de voyages ou travel planners. Hors périmètre MVP | Produit, Roadmap |
| **Voyage actif** | Voyage dont les dates encadrent la date du jour (`status = 'active'`). Voyage affiché en priorité dans le dashboard | Mobile, BDD |
| **Étape (Step)** | Événement planifié dans l'itinéraire d'un voyage (vol, hébergement, activité, transport, restaurant, autre) | Produit, BDD |
| **Itinéraire** | Ensemble ordonné chronologiquement des étapes d'un voyage, consultable jour par jour | Produit, UX |
| **Owner** | Utilisateur créateur d'un voyage, automatiquement désigné administrateur. Seul habilité à supprimer le voyage | Produit, Sécurité |
| **Participant** | Utilisateur invité à un voyage par l'owner. Peut avoir le rôle `viewer`, `editor` ou `admin` | Produit, BDD |
| **RLS** | Row Level Security — mécanisme natif PostgreSQL/Supabase garantissant qu'un utilisateur ne peut accéder qu'aux données qui lui appartiennent ou lui sont partagées | BDD, Sécurité |
| **Travel Planner** | Professionnel indépendant préparant des voyages pour des clients. Persona B2B, hors périmètre MVP | Produit, Roadmap |
| **Agence de voyages** | Structure professionnelle créant des voyages clé en main. Persona B2B, hors périmètre MVP | Produit, Roadmap |
| **Statut voyage** | État calculé : `upcoming` (à venir) / `active` (en cours) / `past` (passé) | BDD, Mobile, UX |
| **Album partagé** | Collection de souvenirs générée depuis un voyage, accessible via lien public tokenisé sans authentification | Produit, Sécurité |
| **URL signée** | URL temporaire et sécurisée générée par Supabase Storage pour accéder à un fichier privé | Backend, Sécurité |
| **Offline cache** | Données stockées localement sur le device (SQLite / AsyncStorage) permettant la consultation sans connexion | Mobile, Architecture |
| **File de synchronisation** | Queue locale stockant les actions effectuées hors connexion (création étape, souvenir) et les envoyant au serveur lors du retour en ligne | Mobile, Architecture |
| **Pack hors ligne** | Pack de données linguistiques téléchargeable pour le traducteur, permettant une traduction partielle sans connexion | Produit, Mobile |
| **Taux horodaté** | Dernier taux de change connu affiché avec la date/heure de sa dernière mise à jour, en cas d'indisponibilité de l'API devises | Mobile, UX |
| **Design System** | Ensemble de tokens (couleurs, typographie, espacement) et de composants atomiques partagés entre la surface web et mobile | Frontend, Mobile |
| **Tokens CSS / Design tokens** | Variables de design (couleurs, typographies, espacements) définies une seule fois et consommées par les composants web et React Native | Frontend |
| **Blocs de développement** | Unités de planification du Tech Lead regroupant des tâches par cohérence fonctionnelle et ordre d'exécution | Tech Lead, Planning |
| **Critères de sortie** | Conditions vérifiables à satisfaire avant de clore un Bloc et démarrer les Blocs dépendants | Tech Lead, QA |
| **JWT** | JSON Web Token — format de token d'authentification sans état utilisé pour sécuriser les échanges client/serveur | Backend, Auth |
| **OAuth2** | Protocole d'autorisation permettant la connexion via fournisseurs tiers (Google, Apple) | Backend, Auth |
| **FCM** | Firebase Cloud Messaging — service de push notifications pour Android | Backend, Mobile |
| **APNs** | Apple Push Notification service — service de push notifications pour iOS | Backend, Mobile |
| **Supabase** | Plateforme BaaS (Backend as a Service) fournissant PostgreSQL + Auth + Storage + Realtime. Remplace l'infrastructure Express/Redis/S3 prévue initialement | Backend, Architecture |
| **DeepL** | API de traduction tierce utilisée par le module Traducteur | Backend, Outils |
| **Fixer.io** | API de taux de change en temps réel utilisée par le module Convertisseur | Backend, Outils |
| **Detox** | Framework de tests E2E pour applications React Native (iOS + Android) | QA |
| **k6** | Outil de test de performance et de charge pour les API | QA |
| **MoSCoW** | Méthode de priorisation : Must Have / Should Have / Could Have / Won't Have | Produit |

---

## 8. JOURNAL DES DÉCISIONS

> *Document obligatoire — trace chronologique des décisions structurantes du projet.*

---

### DEC-001 — Priorité MVP B2C, exclusion B2B du premier cycle

| Champ | Valeur |
|---|---|
| **ID** | DEC-001 |
| **Date** | Cycle 1 — Phase Product Owner |
| **Agent** | Product Owner |
| **Contexte** | La vision initiale du porteur de projet cible trois canaux : agences de voyages, travel planners (B2B) et voyageurs directs (B2C). Ces modèles impliquent des architectures d'usage fondamentalement différentes (back-office, white label, gestion clients). |
| **Décision** | Le MVP est intégralement centré sur le canal B2C. Les fonctionnalités B2B (back-office agence, marque blanche, gestion portefeuille clients) sont exclues du premier cycle de développement. |
| **Justification** | Réduire la complexité initiale, valider le produit core avec les utilisateurs finaux avant d'investir dans l'infrastructure B2B. Éviter de sur-architect pour un cas d'usage non encore validé. |
| **Impacts** | Architecture, BDD, QA : aucun module B2B à prévoir. Roadmap : B2B identifié comme Phase 2. Toute l'équipe doit refuser les dérives vers le B2B en cours de Bloc 0–9. |
| **Réversibilité** | Décision réversible en Phase 2. L'architecture technique (Supabase RLS, multi-tenant latent) a été pensée pour accommoder une extension B2B sans refonte majeure. |

---

### DEC-002 — Stack technique : Next.js API Routes + Supabase en lieu d'Express/PostgreSQL/Redis

| Champ | Valeur |
|---|---|
| **ID** | DEC-002 |
| **Date** | Cycle 1 — Phase Backend Engineer |
| **Agent** | Backend Engineer (aligné avec Architecte) |
| **Contexte** | L'Architecte avait prescrit une stack Node.js/Express + PostgreSQL + Redis + S3-compatible. Le Backend Engineer a proposé une adaptation pour la V1. |
| **Décision** | Remplacement de l'infrastructure classique par Supabase (PostgreSQL + Auth + Storage + Edge Cache) et Next.js API Routes en lieu d'Express. |
| **Justification** | Supabase fournit nativement Auth (JWT + OAuth), RLS, Storage et Realtime — éliminant la configuration manuelle de plusieurs services. Accélère le time-to-market sur le MVP. Réduit la charge opérationnelle DevOps pour une équipe de 2. |
| **Impacts** | Élimination de Redis comme service indépendant (remplacé par Supabase Edge Cache + cache in-memory sur les routes). Supabase Auth remplace la gestion JWT custom. La logique RLS est déplacée côté base de données (gain sécurité). Réduction du coût infrastructure initial. |
| **Réversibilité** | Décision partiellement réversible. La logique métier dans les API Routes peut migrer vers Express. Le schéma SQL est standard PostgreSQL, donc portable. La migration hors Supabase impliquerait de re-implémenter Auth et Storage. |

---

### DEC-003 — APIs tierces encapsulées exclusivement côté serveur

| Champ | Valeur |
|---|---|
| **ID** | DEC-003 |
| **Date** | Cycle 1 — Phase Backend Engineer |
| **Agent** | Backend Engineer |
| **Contexte** | Les outils utilitaires (traducteur, convertisseur, fuseaux horaires) reposent sur des APIs tierces payantes avec clés secrètes (DeepL, Fixer.io, Timezone API). |
| **Décision** | Aucun appel aux APIs tierces ne s'effectue depuis le client mobile ou web. Le module Outils (API Routes `/api/tools/*`) est le seul point d'entrée autorisé. |
| **Justification** | Sécurité des clés API (exposition dans le bundle client = vol de clé). Contrôle du rate limiting et du quota consommé côté serveur. Possibilité de changer de fournisseur sans impacter le client. Centralisation du cache des résultats (taux de change). |
| **Impacts** | Le module Outils mobile doit toujours passer par le backend, même pour des opérations apparemment simples. En mode offline, les données affichées sont les dernières données mises en cache côté client

— DOCUMENTATION MANAGER

*(Continuation exacte à partir du dernier mot produit)*

côté client.

---

### DEC-004 — React Native comme technologie mobile principale

| Champ | Valeur |
|---|---|
| **ID** | DEC-004 |
| **Date** | Cycle 1 — Phase Architecte / Tech Lead |
| **Agent** | Architecte, confirmé par Tech Lead |
| **Contexte** | L'application cible iOS et Android simultanément. Deux approches principales : développement natif séparé (Swift + Kotlin) ou framework cross-platform. |
| **Décision** | React Native retenu comme technologie de développement mobile unique pour iOS et Android. |
| **Justification** | Codebase unique pour deux plateformes = réduction significative du coût de développement sur le MVP. Équipe de 2 développeurs fullstack peut couvrir les deux plateformes. Écosystème mature avec support Expo disponible. Partage des tokens de design system avec la surface web Next.js. |
| **Impacts** | Performance native limitée pour des cas d'usage lourds (traitement d'image), acceptable pour le périmètre MVP. Le développeur mobile doit maîtriser React Native. Certaines fonctionnalités natives (APNs, Keychain) nécessitent des modules spécifiques par plateforme. |
| **Réversibilité** | Décision structurante difficile à inverser sans refonte. Acceptable dans le contexte MVP — réévaluation possible en Phase 2 si la performance s'avère insuffisante. |

---

### DEC-005 — Architecture frontend web : Next.js 14 App Router pour la surface companion

| Champ | Valeur |
|---|---|
| **ID** | DEC-005 |
| **Date** | Cycle 1 — Phase Frontend Engineer |
| **Agent** | Frontend Engineer |
| **Contexte** | Le projet est centré sur une app mobile. Une surface web est identifiée pour le tableau de bord B2C, l'onboarding web et la page publique des albums partagés. |
| **Décision** | Next.js 14 avec App Router retenu pour la surface web, en complément de l'app mobile React Native. |
| **Justification** | App Router Next.js 14 permet le Server-Side Rendering natif, optimisant le SEO de la page publique album. Route groups permettent de séparer proprement les zones auth/app. Partage des tokens de design system avec React Native via variables CSS. Cohérence technologique (React) dans l'équipe. |
| **Impacts** | La surface web n'est pas une PWA de remplacement — c'est une surface complémentaire. Les deux surfaces (web + mobile) consomment les mêmes endpoints API. Le Design System doit être documenté une seule fois et implémenté deux fois (CSS pour web, StyleSheet pour RN). |
| **Réversibilité** | Décision réversible indépendamment du mobile. La couche API est découplée de la surface web. |

---

### DEC-006 — Pyramide de tests 60/30/10 (unitaire / intégration / E2E)

| Champ | Valeur |
|---|---|
| **ID** | DEC-006 |
| **Date** | Cycle 1 — Phase QA Engineer |
| **Agent** | QA Engineer |
| **Contexte** | L'application couvre 7 modules fonctionnels avec des interactions offline/online complexes. Plusieurs stratégies de test étaient envisageables. |
| **Décision** | Pyramide 60% tests unitaires (Jest + RTL) / 30% tests intégration (Jest + Supertest) / 10% tests E2E (Detox). |
| **Justification** | Les tests unitaires offrent le meilleur ratio coût/fiabilité pour la logique métier et les utilitaires. Les tests intégration couvrent les contrats API et les règles RLS — couche critique pour la sécurité. Les tests E2E sont limités aux parcours utilisateurs critiques (Must Have PO) pour éviter une suite de tests fragile et coûteuse à maintenir. |
| **Impacts** | Chaque module doit atteindre ≥ 80% de couverture sur la logique métier. 100% des endpoints documentés couverts par des tests d'intégration. 100% des parcours PO Must Have couverts par un test E2E. |
| **Réversibilité** | Décision de méthode, ajustable à chaque cycle sans impact architectural. |

---

### DEC-007 — Offline : consultation en cache, création en file de synchronisation

| Champ | Valeur |
|---|---|
| **ID** | DEC-007 |
| **Date** | Cycle 1 — Phase Architecte / UX Designer / Tech Lead |
| **Agent** | Architecte (décision technique), UX Designer (expression utilisateur) |
| **Contexte** | Le voyageur peut se retrouver sans connexion (avion, zone isolée). L'app doit rester utilisable. Deux approches : read-only strict offline, ou offline complet avec synchronisation différée. |
| **Décision** | Mode offline à deux niveaux : (1) consultation complète depuis le cache local (SQLite/AsyncStorage) pour l'itinéraire et les documents déjà chargés ; (2) actions de création/modification stockées dans une file de synchronisation locale et envoyées au serveur lors du retour en ligne. |
| **Justification** | Le cas d'usage principal en offline est la consultation (vérifier son vol, retrouver sa réservation d'hôtel). La création en offline (ajouter une étape, un souvenir) est un cas secondaire mais attendu. La file de synchronisation évite la perte de données sans imposer une complexité de résolution de conflits au MVP. |
| **Impacts** | Le module Offline est un Bloc de développement entier (Bloc 7). Les composants UI doivent intégrer des indicateurs visuels de l'état de connexion. La traduction et la conversion de devises ne sont pas disponibles hors ligne (dépendance APIs tierces serveur) — un message informatif remplace l'erreur. Les documents importés sont mis en cache automatiquement. |
| **Réversibilité** | La stratégie de cache est implémentée dans un module dédié — évolutive sans impact sur les autres modules. |

---

### DEC-008 — Périmètre QA : B2B, interface web et page album exclus du premier cycle

| Champ | Valeur |
|---|---|
| **ID** | DEC-008 |
| **Date** | Cycle 1 — Phase QA Engineer |
| **Agent** | QA Engineer (aligné avec DEC-001) |
| **Contexte** | Le livrable QA couvre la stratégie de test du MVP. Plusieurs surfaces ont été identifiées dans les livrables Frontend et Architecture. |
| **Décision** | Le cycle QA 1 couvre exclusivement l'app mobile MVP B2C. L'interface web companion, la page publique album (couverture minimale uniquement) et toutes les fonctionnalités B2B sont hors périmètre de test pour ce cycle. |
| **Justification** | Cohérence avec DEC-001. Concentrer les ressources QA sur les parcours Must Have validés par le Product Owner. La page album public reçoit une couverture minimale (smoke test) car elle est critique pour le partage mais non complexe. |
| **Impacts** | La surface web Next.js n'a pas de suite de tests dédiée en Cycle 1. Elle sera couverte en Cycle 2. La page album public a un smoke test de disponibilité uniquement. |
| **Réversibilité** | Périmètre extensible en Cycle 2 sans impact sur les décisions existantes. |

---

## 9. RELEASE NOTES — VERSION 0.1.0 (MVP B2C — Cycle 1)

> *Document destiné aux équipes internes, parties prenantes et porteur de projet.*

---

### Informations de version

| Champ | Valeur |
|---|---|
| **Version** | 0.1.0 — MVP B2C |
| **Statut** | En développement (Cycle 1) |
| **Durée estimée** | 12 semaines (Blocs 0–9) |
| **Équipe** | 2 développeurs fullstack + QA |
| **Périmètre** | B2C uniquement — iOS + Android |

---

### Fonctionnalités incluses dans cette version

#### 🧳 Module Voyage & Itinéraire
- Création d'un voyage (destination, dates, participants optionnels)
- Vue itinéraire jour par jour
- Détail d'une étape (lieu, horaire, notes, documents associés)
- Ajout et édition d'étapes (vol, hébergement, activité, transport, restaurant, autre)
- Tableau de bord avec statuts voyage (à venir / en cours / passé)

#### 📄 Module Documents
- Import de documents (PDF, JPG, PNG — 20 Mo max par fichier)
- Catégorisation automatique (vol / hébergement / activité / autre)
- Consultation offline des documents mis en cache
- Visionneuse inline

#### 🔧 Module Outils Utilitaires
- Traducteur avec pré-sélection de langue depuis la destination du voyage actif
- Lecture audio de la traduction
- Convertisseur de devises avec taux en temps réel et affichage du taux horodaté
- Horloge et affichage du fuseau horaire de la destination

#### 📸 Module Souvenirs & Partage
- Galerie de souvenirs par voyage (photo, note, lieu)
- Ajout de souvenirs
- Génération d'un lien d'album partagé accessible sans authentification

#### 🔔 Notifications & Rappels
- Rappels configurables avant les étapes (24h, 2h, personnalisé)
- Respect du fuseau horaire de l'étape
- Activation/désactivation depuis les paramètres

#### 🔐 Authentification
- Inscription et connexion email/mot de passe
- Connexion Google (iOS + Android)
- Connexion Apple (iOS uniquement)
- Session persistante avec refresh token silencieux

#### 📡 Mode Offline
- Consultation de l'itinéraire et des documents en cache
- File de synchronisation pour les créations effectuées hors connexion
- Indicateurs visuels de l'état de connexion

---

### Fonctionnalités explicitement hors périmètre (Phase 2)

| Fonctionnalité | Raison |
|---|---|
| Back-office agence de voyages | B2B — Phase 2 |
| White label / marque blanche | B2B — Phase 2 |
| Gestion portefeuille clients (travel planner) | B2B — Phase 2 |
| Import automatique depuis email | Complexité technique — Phase 2 |
| Traduction offline (pack hors ligne) | Should Have — Phase 2 |
| Notifications de partage social | Could Have — Phase 2 |
| Interface web companion complète | Couverture minimale en MVP |

---

### Dépendances tierces

| Service | Usage | Criticité |
|---|---|---|
| Supabase | BDD, Auth, Storage | Critique |
| DeepL API | Traduction | Haute |
| Fixer.io / Open Exchange Rates | Taux de change | Haute |
| Timezone API | Fuseaux horaires | Moyenne |
| FCM (Firebase) | Push Android | Haute |
| APNs (Apple) | Push iOS | Haute |

---

### Risques identifiés pour cette version

| Risque | Probabilité | Impact | Mitigation |
|---|---|---|---|
| Indisponibilité API tierce (DeepL, Fixer.io) | Moyenne | Moyen | Cache des derniers résultats + message informatif offline |
| Dépassement quota Supabase Free Tier | Faible (MVP) | Moyen | Monitoring consommation dès Bloc 0, upgrade préventif |
| Fragmentation offline/online mal gérée | Moyenne | Élevé | Bloc 7 dédié, tests Detox avec Network mocking |
| Refus App Store / Play Store | Faible | Élevé | Respect des guidelines dès Bloc 8 (UX Polish), review interne pré-soumission |
| Dérive vers features B2B en cours de sprint | Faible | Moyen | DEC-001 documentée, critères de sortie par Bloc comme garde-fou |

---

## 10. CHECKLIST DE DÉMARRAGE DE PROJET

> *À valider par le porteur de projet et le Tech Lead avant le démarrage du Bloc 0.*

### Administratif & Légal
- [ ] Nom de l'application défini et vérifié (disponibilité marque, domaine)
- [ ] Comptes Apple Developer Program créés et actifs (99$/an)
- [ ] Compte Google Play Console créé (25$ one-time)
- [ ] Politique de confidentialité rédigée (obligatoire pour Auth sociale et push)
- [ ] CGU rédigées

### Technique
- [ ] Compte Supabase créé, projet initialisé (environnements dev / staging / prod)
- [ ] Compte DeepL API créé, clé générée
- [ ] Compte Fixer.io (ou Open Exchange Rates) créé, clé générée
- [ ] Compte Firebase créé (FCM), projet configuré
- [ ] Repository GitHub créé, accès équipe configuré
- [ ] Secrets manager configuré (GitHub Secrets ou équivalent)

### Design
- [ ] Maquettes haute-fidélité validées par le porteur de projet
- [ ] Design System figé (tokens, couleurs, typographie)
- [ ] Ressources graphiques fournies (icône app, splashscreen, assets marketing stores)

### Produit
- [ ] Backlog product owner priorisé et importé dans l'outil de suivi (Jira, Linear, Notion…)
- [ ] Définition of Done validée par l'équipe
- [ ] Critères de sortie du Bloc 0 approuvés par le Tech Lead

---

*Livrable produit par le Documentation Manager — App Builder IA*
*Version : Cycle 1 — Complet*
*Statut : Prêt pour archivage et diffusion équipe*