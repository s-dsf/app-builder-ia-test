# tech_lead

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— TECH LEAD

# Plan d'Exécution — Travel Companion App
## MVP B2C · React Native + Node.js + PostgreSQL

---

## 0. SYNTHÈSE EXÉCUTIVE

### Périmètre du MVP
Ce plan couvre exclusivement le **MVP B2C** tel que défini par le Product Owner. Les fonctionnalités B2B (back-office agence, white label, marque blanche) sont **hors périmètre** et ne figurent dans aucune estimation.

### Stack technique retenue (décisions architecte)
| Couche | Technologie |
|---|---|
| Mobile | React Native (iOS + Android) |
| Backend | Node.js (Express) |
| Base de données | PostgreSQL + Redis |
| Stockage fichiers | S3-compatible (ex. AWS S3 ou Supabase Storage) |
| Auth | JWT + OAuth2 (Google, Apple) |
| Push Notifications | FCM (Android) + APNs (iOS) |
| APIs tierces | DeepL (traduction), Fixer.io (devises), Timezone API |
| CI/CD | GitHub Actions |
| Environnements | Dev / Staging / Production |

### Durée totale estimée
**12 semaines** en rythme soutenu avec une équipe de 2 développeurs (1 fullstack orienté backend, 1 fullstack orienté frontend/mobile).

---

## 1. STRUCTURE EN BLOCS DE DÉVELOPPEMENT

```
BLOC 0 — Infrastructure & Fondations          (Semaine 1)
BLOC 1 — Authentification                     (Semaine 2)
BLOC 2 — Gestion des voyages & Itinéraire     (Semaines 3-4)
BLOC 3 — Gestion des documents                (Semaine 5)
BLOC 4 — Outils utilitaires                   (Semaine 6)
BLOC 5 — Module Souvenirs & Partage           (Semaines 7-8)
BLOC 6 — Notifications & Rappels              (Semaine 9)
BLOC 7 — Offline & Synchronisation            (Semaine 10)
BLOC 8 — UX Polish & États vides              (Semaine 11)
BLOC 9 — Tests, QA & Préparation Stores       (Semaine 12)
```

---

## 2. DÉTAIL PAR BLOC

---

### BLOC 0 — Infrastructure & Fondations
**Durée : 1 semaine**
**Dépendances : aucune — point de départ absolu**

#### Objectif
Poser l'infrastructure qui permettra à tous les blocs suivants de fonctionner. Rien ne peut démarrer en parallèle avant la fin de ce bloc.

#### Tâches Backend
| # | Tâche | Détail |
|---|---|---|
| B0-BE-01 | Setup projet Node.js | Structure Express, ESLint, Prettier, variables d'environnement |
| B0-BE-02 | Base de données PostgreSQL | Initialisation, connexion, système de migrations (ex. Knex ou Prisma) |
| B0-BE-03 | Redis | Setup pour cache sessions et taux de change |
| B0-BE-04 | API Gateway | Nginx ou configuration Express avec routing `/v1/`, rate limiting de base |
| B0-BE-05 | Object Storage | Connexion S3, bucket dev/staging/prod, politique d'accès |
| B0-BE-06 | Pipelines CI/CD | GitHub Actions : lint, test, deploy vers staging automatique |
| B0-BE-07 | Environnements | Variables `.env` par environnement, secrets manager |
| B0-BE-08 | Schéma de données initial | Migration des tables : `users`, `trips`, `steps`, `documents`, `participants` |

#### Tâches Frontend / Mobile
| # | Tâche | Détail |
|---|---|---|
| B0-FE-01 | Setup React Native | Initialisation projet (Expo ou RN CLI), structure dossiers, ESLint |
| B0-FE-02 | Navigation | React Navigation — stack principal, bottom tabs, structure d'arborescence |
| B0-FE-03 | Design System | Tokens (couleurs, typographie, espacement), composants atomiques (Button, Input, Card) |
| B0-FE-04 | API Client | Axios configuré, intercepteurs pour JWT, gestion erreurs globale |
| B0-FE-05 | State Management | Setup Redux Toolkit ou Zustand — stores initiaux vides |
| B0-FE-06 | Local Storage | Setup AsyncStorage + SQLite (WatermelonDB ou expo-sqlite) pour le cache offline |

#### Critères de sortie du Bloc 0
- [ ] L'API répond sur `/v1/health` en staging
- [ ] La connexion PostgreSQL et Redis est opérationnelle
- [ ] Le bucket S3 est créé et accessible depuis le backend
- [ ] L'app React Native démarre sur simulateur iOS ET Android sans erreur
- [ ] La navigation de base est fonctionnelle (écrans vides mais routables)
- [ ] Le pipeline CI déploie automatiquement sur staging à chaque push sur `main`

---

### BLOC 1 — Authentification
**Durée : 1 semaine**
**Dépendances : Bloc 0 terminé**

#### Objectif
Permettre à un utilisateur de créer un compte, se connecter et maintenir sa session. Porte d'entrée obligatoire avant tout autre module.

#### Tâches Backend
| # | Tâche | US couverte |
|---|---|---|
| B1-BE-01 | Endpoint `POST /v1/auth/register` | Email + password, hash bcrypt, création user |
| B1-BE-02 | Endpoint `POST /v1/auth/login` | Vérification credentials, émission JWT access + refresh token |
| B1-BE-03 | Endpoint `POST /v1/auth/refresh` | Rotation des tokens, invalidation ancien refresh |
| B1-BE-04 | Endpoint `POST /v1/auth/logout` | Invalidation refresh token côté serveur |
| B1-BE-05 | OAuth Google + Apple | Intégration passport.js ou équivalent, mapping vers user interne |
| B1-BE-06 | Middleware auth | Validation JWT sur toutes les routes protégées |
| B1-BE-07 | Endpoint `GET /v1/auth/me` | Retourne le profil utilisateur connecté |

#### Tâches Frontend / Mobile
| # | Tâche | Écran UX |
|---|---|---|
| B1-FE-01 | Écran Onboarding (3 slides) | Valeur proposition, CTA "Commencer" |
| B1-FE-02 | Écran Inscription | Email + password, validation inline, feedback erreur |
| B1-FE-03 | Écran Connexion | Email + password + boutons OAuth Google/Apple |
| B1-FE-04 | Gestion tokens côté client | Stockage sécurisé (Keychain/Keystore), refresh automatique silencieux |
| B1-FE-05 | Store Auth | État connecté/déconnecté, persistance de session au redémarrage |
| B1-FE-06 | Flux de redirection | Connecté → Home, Déconnecté → Onboarding, gestion deep links futurs |

#### Critères de sortie du Bloc 1
- [ ] Inscription email/password fonctionnelle end-to-end
- [ ] Connexion email/password fonctionnelle
- [ ] Connexion Google fonctionnelle sur iOS et Android
- [ ] Connexion Apple fonctionnelle sur iOS
- [ ] Session persistante au redémarrage de l'app
- [ ] Token expiré déclenche un refresh silencieux sans déconnecter l'utilisateur
- [ ] Un utilisateur non connecté ne peut accéder à aucune route protégée

---

### BLOC 2 — Gestion des Voyages & Itinéraire
**Durée : 2 semaines**
**Dépendances : Bloc 1 terminé**

> Bloc critique — c'est le cœur du produit. Toutes les autres fonctionnalités gravitent autour du voyage.

#### Semaine 1 du bloc — CRUD Voyages + Tableau de bord

##### Tâches Backend
| # | Tâche | US couverte |
|---|---|---|
| B2-BE-01 | `POST /v1/trips` | Création voyage (nom, destination, dates, participants) — US-A01 |
| B2-BE-02 | `GET /v1/trips` | Liste des voyages de l'utilisateur avec statuts (à venir / en cours / passé) |
| B2-BE-03 | `GET /v1/trips/:id` | Détail d'un voyage |
| B2-BE-04 | `PUT /v1/trips/:id` | Édition d'un voyage |
| B2-BE-05 | `DELETE /v1/trips/:id` | Suppression (soft delete) |
| B2-BE-06 | Calcul statut automatique | Job ou trigger PostgreSQL calculant `upcoming/ongoing/past` selon dates |
| B2-BE-07 | `POST /v1/trips/:id/participants` | Invitation participant par email |
| B2-BE-08 | Gestion rôles | Créateur = admin, participant = lecteur |

##### Tâches Frontend / Mobile
| # | Tâche | Écran UX |
|---|---|---|
| B2-FE-01 | Tableau de bord | Carte voyage actif en avant, liste voyages, état vide motivant |
| B2-FE-02 | Formulaire création voyage | Nom, destination (autocomplete), dates, participants (optionnel) |
| B2-FE-03 | Récapitulatif avant validation | Review avant création |
| B2-FE-04 | Store Voyages | Liste voyages, voyage actif, CRUD actions |
| B2-FE-05 | Contexte voyage actif | Mécanisme global : le voyage actif informe tous les modules (outils, etc.) |

#### Semaine 2 du bloc — Itinéraire & Étapes

##### Tâches Backend
| # | Tâche | US couverte |
|---|---|---|
| B2-BE-09 | `POST /v1/trips/:id/steps` | Création d'une étape (lieu, horaire, notes, type) — US-A02 |
| B2-BE-10 | `GET /v1/trips/:id/steps` | Liste des étapes triées par date/heure |
| B2-BE-11 | `PUT /v1/trips/:id/steps/:stepId` | Édition d'une étape |
| B2-BE-12 | `DELETE /v1/trips/:id/steps/:stepId` | Suppression d'une étape |
| B2-BE-13 | Endpoint groupé par jour | `GET /v1/trips/:id/steps?groupBy=day` — optimisé pour la vue mobile |

##### Tâches Frontend / Mobile
| # | Tâche | Écran UX |
|---|---|---|
| B2-FE-06 | Vue Itinéraire | Affichage chronologique jour par jour, scroll vertical |
| B2-FE-07 | Détail d'une étape | Lieu, horaire, notes, placeholder documents |
| B2-FE-08 | Formulaire ajout/édition étape | Champs : titre, lieu, date/heure, type, notes |
| B2-FE-09 | État vide itinéraire | Guidage avec 2-3 actions suggérées (importer doc, ajouter étape) |

#### Critères de sortie du Bloc 2
- [ ] Création, édition, suppression d'un voyage fonctionnelles
- [ ] Statuts automatiques (à venir / en cours / passé) corrects
- [ ] Le voyage actif est correctement identifié et accessible depuis n'importe quel écran
- [ ] Vue itinéraire jour par jour fonctionnelle
- [ ] Ajout, édition, suppression d'étapes fonctionnels
- [ ] Invitation de participant par email (envoi email + accès en lecture)
- [ ] État vide soigné visible sur tableau de bord ET itinéraire

---

### BLOC 3 — Gestion des Documents
**Durée : 1 semaine**
**Dépendances : Bloc 2 terminé**

#### Objectif
Permettre l'import, la catégorisation, la visualisation et l'accès offline aux documents de voyage (US-A03).

#### Tâches Backend
| # | Tâche | Détail |
|---|---|---|
| B3-BE-01 | `POST /v1/trips/:id/documents` | Upload fichier vers S3, association catégorie, limite 20 Mo |
| B3-BE-02 | `GET /v1/trips/:id/documents` | Liste documents par catégorie (vol / hébergement / activité / autre) |
| B3-BE-03 | `GET /v1/documents/:id/url` | Génération URL signée temporaire (accès sécurisé S3) |
| B3-BE-04 | `DELETE /v1/documents/:id` | Suppression fichier S3 + enregistrement BDD |
| B3-BE-05 | Validation types | Vérification MIME type : PDF, JPG, PNG uniquement |
| B3-BE-06 | Association étape | `PATCH /v1/documents/:id/step` — lier un document à une étape |

#### Tâches Frontend / Mobile
| # | Tâche | Écran UX |
|---|---|---|
| B3-FE-01 | Document Picker | Sélection fichier depuis galerie ou système de fichiers (expo-document-picker) |
| B3-FE-02 | Écran upload | Progress bar, feedback succès/erreur, sélection catégorie |
| B3-FE-03 | Liste documents catégorisée | Onglets ou sections par catégorie, icône par type |
| B3-FE-04 | Viewer documents | PDF viewer (react-native-pdf), image viewer |
| B3-FE-05 | Cache offline documents | Téléchargement local après import, accessible sans connexion |
| B3-FE-06 | Indicateur offline | Badge visuel sur les documents disponibles offline |

#### Critères de sortie du Bloc 3
- [ ] Upload PDF, JPG, PNG fonctionnel (limite 20 Mo respectée)
- [ ] Catégorisation correcte à l'import
- [ ] Visualisation in-app des PDF et images
- [ ] Documents accessibles hors connexion après premier chargement
- [ ] Association d'un document à une étape fonctionnelle
- [ ] Suppression propre (S3 + BDD)

---

### BLOC 4 — Outils Utilitaires
**Durée : 1 semaine**
**Dépendances : Bloc 2 terminé (contexte voyage actif requis)**

#### Objectif
Implémenter le traducteur, le convertisseur de devises et l'horloge des fuseaux horaires, tous contextualisés via le voyage actif (US-B01, US-B02, US-B03).

#### Tâches Backend
| # | Tâche | API tierce |
|---|---|---|
| B4-BE-01 | `POST /v1/tools/translate` | Proxy DeepL, langue cible depuis voyage actif |
| B4-BE-02 | `GET /v1/tools/exchange-rates` | Proxy Fixer.io, résultat mis en cache Redis 1h |
|

— TECH LEAD

*(Reprise exacte après le tableau du Bloc 4 — Tâches Backend)*

| B4-BE-03 | `GET /v1/tools/timezones` | Calcul fuseaux horaires via bibliothèque serveur (luxon / date-fns-tz), pas d'API tierce requise |
| B4-BE-04 | Cache Redis outils | Exchange rates : TTL 1h — Langues disponibles : TTL 24h |

#### Tâches Frontend / Mobile

| # | Tâche | Écran UX |
|---|---|---|
| B4-FE-01 | Hub Outils | Écran d'accès regroupant les 3 outils, accessible depuis la barre persistante |
| B4-FE-02 | Traducteur | Saisie texte, langue cible pré-remplie depuis voyage actif, résultat en < 3s |
| B4-FE-03 | Lecture audio traduction | Intégration TTS (expo-speech ou API tierce audio) |
| B4-FE-04 | Convertisseur de devises | Saisie montant, devise source/cible pré-remplies depuis voyage actif, affichage taux |
| B4-FE-05 | Horloge / Fuseaux horaires | Heure locale destination en temps réel, comparatif heure utilisateur |
| B4-FE-06 | États offline outils | Traducteur et convertisseur affichent un état désactivé clair (non cassé) hors connexion |

#### Critères de sortie du Bloc 4
- [ ] Traducteur fonctionnel avec pré-sélection langue depuis voyage actif
- [ ] Lecture audio du résultat de traduction opérationnelle
- [ ] Convertisseur fonctionnel avec taux mis à jour (cache Redis vérifié)
- [ ] Horloge affichant correctement la destination et le fuseau utilisateur
- [ ] Comportement offline correct : état désactivé visible, aucun crash
- [ ] Aucun appel direct depuis le mobile vers DeepL ou Fixer.io (tout passe par l'API Gateway)

---

### BLOC 5 — Module Souvenirs & Partage
**Durée : 1,5 semaine**
**Dépendances : Bloc 2 terminé (voyage actif), Bloc 3 terminé (stockage S3 partagé)**

#### Objectif
Permettre au voyageur d'ajouter des souvenirs (photos, notes, lieux), de constituer un album par voyage et de le partager avec ses proches via un lien public (US-C01, US-C02, US-C03).

#### Tâches Backend

| # | Tâche | Détail |
|---|---|---|
| B5-BE-01 | `POST /v1/trips/:id/memories` | Création souvenir : photo (S3), note texte, localisation GPS optionnelle |
| B5-BE-02 | `GET /v1/trips/:id/memories` | Liste des souvenirs du voyage, triés par date de création |
| B5-BE-03 | `PUT /v1/memories/:id` | Édition note ou métadonnées d'un souvenir |
| B5-BE-04 | `DELETE /v1/memories/:id` | Suppression souvenir (S3 + BDD) |
| B5-BE-05 | `POST /v1/trips/:id/memories/share` | Génération token de partage unique + URL publique avec TTL paramétrable |
| B5-BE-06 | `GET /v1/shared/:shareToken` | Endpoint public (sans auth) — retourne l'album avec ses souvenirs |
| B5-BE-07 | Gestion TTL partage | Token expirable (7j / 30j / permanent) selon choix utilisateur |
| B5-BE-08 | Quota média | Limite par voyage (ex. 500 Mo) vérifiée à l'upload |

#### Tâches Frontend / Mobile

| # | Tâche | Écran UX |
|---|---|---|
| B5-FE-01 | Galerie souvenirs | Vue grille par voyage, scroll infini, date superposée sur vignette |
| B5-FE-02 | Ajout souvenir | Picker photo (galerie ou appareil photo), champ note, géolocalisation optionnelle |
| B5-FE-03 | Détail souvenir | Photo plein écran, note, lieu, date — swipe entre souvenirs |
| B5-FE-04 | Édition / Suppression souvenir | Accès depuis le détail, confirmation avant suppression |
| B5-FE-05 | Génération lien de partage | Choix TTL, copie lien, partage natif (Share API) |
| B5-FE-06 | Page album public (WebView ou web) | Vue dégradée accessible sans compte pour les destinataires |
| B5-FE-07 | Feedback émotionnel | Animations légères à l'ajout d'un souvenir (principe P5 UX) |

#### Critères de sortie du Bloc 5
- [ ] Ajout photo depuis galerie ET appareil photo fonctionnel
- [ ] Note texte et localisation optionnelle associées au souvenir
- [ ] Galerie affichée correctement (grille, dates, scroll)
- [ ] Génération de lien de partage fonctionnelle avec TTL respecté
- [ ] Album public accessible sans compte via le lien partagé
- [ ] Suppression propre (S3 + BDD)
- [ ] Aucun accès à l'album partagé possible après expiration du token

---

### BLOC 6 — Notifications & Rappels
**Durée : 1 semaine**
**Dépendances : Bloc 2 terminé (étapes avec horaires), Bloc 1 terminé (device token)**

#### Objectif
Envoyer des notifications push paramétrables avant les étapes importantes du voyage (US-A04).

#### Tâches Backend

| # | Tâche | Détail |
|---|---|---|
| B6-BE-01 | `POST /v1/devices` | Enregistrement device token FCM/APNs par utilisateur |
| B6-BE-02 | `PUT /v1/notifications/preferences` | Paramétrage délais (24h, 2h, personnalisé) et activation/désactivation |
| B6-BE-03 | Job de planification | Cron ou queue (BullMQ) : calcul des notifications à J-1 de chaque étape |
| B6-BE-04 | Envoi push FCM | Intégration Firebase Admin SDK pour Android |
| B6-BE-05 | Envoi push APNs | Intégration APNs pour iOS |
| B6-BE-06 | Gestion fuseaux horaires | Les rappels sont calculés dans le fuseau de l'étape, pas de l'utilisateur |
| B6-BE-07 | Nettoyage tokens | Suppression des device tokens invalides (feedback FCM/APNs) |

#### Tâches Frontend / Mobile

| # | Tâche | Écran UX |
|---|---|---|
| B6-FE-01 | Permission notifications | Demande de permission au bon moment (après création premier voyage) |
| B6-FE-02 | Enregistrement device token | Envoi automatique du token FCM/APNs au backend après permission accordée |
| B6-FE-03 | Centre de notifications | Liste des rappels reçus, tap → navigation vers l'étape concernée |
| B6-FE-04 | Préférences notifications | Écran paramètres : délais, activation/désactivation globale |
| B6-FE-05 | Deep link depuis notification | Tap sur notification → ouverture directe du détail de l'étape |

#### Critères de sortie du Bloc 6
- [ ] Notification reçue sur iOS et Android pour une étape avec horaire défini
- [ ] Délais 24h et 2h fonctionnels
- [ ] Fuseau horaire de l'étape respecté (test avec étape en décalage horaire)
- [ ] Désactivation globale des notifications respectée
- [ ] Deep link depuis notification atterrit sur le bon écran
- [ ] Device token invalide nettoyé automatiquement

---

### BLOC 7 — Mode Offline & Synchronisation
**Durée : 1 semaine**
**Dépendances : Blocs 2, 3, 4 terminés**

#### Objectif
Consolider le comportement offline de l'application : accès à l'itinéraire, aux documents et aux outils en cache, synchronisation silencieuse au retour de la connexion (principe UX P3).

#### Tâches Backend

| # | Tâche | Détail |
|---|---|---|
| B7-BE-01 | Endpoint sync delta | `GET /v1/trips/:id/sync?since=timestamp` — retourne uniquement les changements depuis la dernière sync |
| B7-BE-02 | Versioning des données | Ajout `updated_at` sur toutes les entités principales (trips, steps, documents) |

#### Tâches Frontend / Mobile

| # | Tâche | Détail |
|---|---|---|
| B7-FE-01 | Stratégie de cache complète | Mise en cache systématique : itinéraire, étapes, liste documents après chargement |
| B7-FE-02 | SQLite / WatermelonDB | Stockage local structuré pour itinéraire et métadonnées documents |
| B7-FE-03 | Détection connexion | NetInfo : bascule automatique entre mode online et offline |
| B7-FE-04 | Indicateurs visuels offline | Bandeau discret "Mode hors connexion", icônes sur contenus non disponibles |
| B7-FE-05 | Synchronisation au retour | Dès reconnexion : appel endpoint delta, merge des données sans perte |
| B7-FE-06 | File d'attente actions offline | Les actions effectuées offline (ajout note, etc.) sont queued et rejouées à la reconnexion |
| B7-FE-07 | Tests offline | Scénarios de test : mode avion, reconnexion partielle, conflit de données |

#### Critères de sortie du Bloc 7
- [ ] Itinéraire lisible en mode avion après un premier chargement connecté
- [ ] Documents importés accessibles hors connexion
- [ ] Aucun message d'erreur "red" visible en mode offline (état dégradé explicite uniquement)
- [ ] Synchronisation transparente au retour de connexion
- [ ] Les actions queued offline sont rejouées sans doublon

---

### BLOC 8 — Qualité, Tests & Préparation Production
**Durée : 1 semaine**
**Dépendances : Tous les blocs fonctionnels terminés (1 à 7)**

#### Objectif
Stabiliser l'ensemble, corriger les régressions inter-blocs, tester les parcours critiques de bout en bout et préparer le déploiement en production.

#### Tâches Transverses Backend

| # | Tâche |
|---|---|
| B8-BE-01 | Tests d'intégration sur tous les endpoints critiques (auth, trips, documents, souvenirs) |
| B8-BE-02 | Tests de charge (k6 ou Artillery) : seuil 200 utilisateurs simultanés |
| B8-BE-03 | Audit sécurité : rate limiting, validation inputs, headers HTTP, gestion erreurs |
| B8-BE-04 | Variables d'environnement production séparées de staging |
| B8-BE-05 | Mise en place monitoring (Sentry backend, alertes Uptime) |
| B8-BE-06 | Revue des logs : suppression logs verbeux, structuration JSON |

#### Tâches Transverses Frontend / Mobile

| # | Tâche |
|---|---|
| B8-FE-01 | Tests E2E Detox sur les 3 parcours critiques (création voyage, import document, partage souvenir) |
| B8-FE-02 | Tests sur devices physiques : iPhone SE, iPhone 15, Samsung Galaxy A53, Pixel 7 |
| B8-FE-03 | Audit accessibilité : contrastes, tailles tactiles minimales (44x44pt), labels ARIA |
| B8-FE-04 | Sentry Mobile : intégration crash reporting |
| B8-FE-05 | Optimisation performances : temps de démarrage < 3s, images lazy loading |
| B8-FE-06 | Build de production iOS (TestFlight) et Android (Play Store Internal Testing) |
| B8-FE-07 | Revue des permissions déclarées (camera, notifications, storage) dans Info.plist et AndroidManifest |

#### Critères de sortie du Bloc 8 — **Conditions de livraison**
- [ ] 0 crash bloquant sur les 3 parcours critiques
- [ ] Temps de démarrage < 3 secondes sur les devices cibles
- [ ] Tous les tests E2E Detox passent en CI
- [ ] Score accessibilité conforme aux standards WCAG AA (contrastes et tailles tactiles)
- [ ] Build disponible sur TestFlight et Play Store Internal Testing
- [ ] Sentry opérationnel en production (première alerte reçue de test)
- [ ] Documentation API Swagger à jour pour tous les endpoints `/v1/`
- [ ] Revue de code finale validée par au moins un pair sur les modules critiques (Auth, Documents, Souvenirs)

---

## 4. RÉCAPITULATIF DES ESTIMATIONS

| Bloc | Contenu | Durée estimée | Criticité |
|---|---|---|---|
| Bloc 0 | Infrastructure & socle technique | 1 semaine | 🔴 Bloquant |
| Bloc 1 | Authentification | 1 semaine | 🔴 Bloquant |
| Bloc 2 | Gestion voyages & itinéraire | 2 semaines | 🔴 Bloquant |
| Bloc 3 | Gestion des documents | 1 semaine | 🟠 Important |
| Bloc 4 | Outils utilitaires | 1 semaine | 🟠 Important |
| Bloc 5 | Souvenirs & Partage | 1,5 semaine | 🟡 Standard |
| Bloc 6 | Notifications & Rappels | 1 semaine | 🟡 Standard |
| Bloc 7 | Mode Offline & Synchronisation | 1 semaine | 🟠 Important |
| Bloc 8 | Qualité, Tests & Prod | 1 semaine | 🔴 Bloquant |
| **TOTAL** | | **~11,5 semaines** | |

> **⚠️ Note de cadrage** : Ces estimations supposent une équipe de 2 développeurs (1 Backend / 1 Frontend-Mobile) à temps plein. En solo developer, multiplier par 1,6 à 2. En équipe de 3+, certains blocs peuvent être partiellement parallélisés (Blocs 3, 4 et 6 peuvent démarrer simultanément après le Bloc 2).

---

## 5. GRAPHE DE DÉPENDANCES

```
BLOC 0 — Infrastructure
    └──► BLOC 1 — Auth
              └──► BLOC 2 — Voyages & Itinéraire
                        ├──► BLOC 3 — Documents
                        │         └──► BLOC 5 — Sou

— TECH LEAD

# Plan d'Exécution — Travel Companion App *(suite et fin)*

---

## 5. GRAPHE DE DÉPENDANCES *(complet)*

```
BLOC 0 — Infrastructure
    └──► BLOC 1 — Auth
              └──► BLOC 2 — Voyages & Itinéraire
                        ├──► BLOC 3 — Documents
                        │         └──► BLOC 5 — Souvenirs & Partage
                        │                   └──► BLOC 8 — Qualité & Prod
                        ├──► BLOC 4 — Outils utilitaires
                        │         └──► BLOC 8 — Qualité & Prod
                        ├──► BLOC 6 — Notifications & Rappels
                        │         └──► BLOC 8 — Qualité & Prod
                        └──► BLOC 7 — Mode Offline & Sync
                                  └──► BLOC 8 — Qualité & Prod
```

**Lecture du graphe :**
- Aucun bloc ne peut démarrer sans que son prédécesseur soit en critères de sortie validés
- Les Blocs 3, 4 et 6 peuvent être développés en parallèle après validation du Bloc 2
- Le Bloc 5 attend obligatoirement le Bloc 3 (stockage S3 partagé)
- Le Bloc 7 consolide tous les blocs fonctionnels — il ne peut démarrer qu'après 2, 3 et 4

---

## 6. RÉPARTITION DES RESPONSABILITÉS PAR BLOC

| Bloc | Backend | Frontend / Mobile | Partagé |
|---|---|---|---|
| Bloc 0 | Infra cloud, BDD, CI/CD, API Gateway | Projet RN, navigation, design system | Variables d'env, conventions Git |
| Bloc 1 | Auth Service, JWT, OAuth, refresh | Écrans login/register, stockage token | Schéma User, règles sécurité |
| Bloc 2 | CRUD voyages, étapes, participants | Dashboard, itinéraire J/J, formulaires | Modèle de données Trip/Step |
| Bloc 3 | Upload S3, URLs signées, quotas | Picker, upload progress, viewer offline | Contrat API documents |
| Bloc 4 | Proxy APIs tierces, cache Redis | Traducteur, convertisseur, horloge | Clés API tierces, gestion erreurs |
| Bloc 5 | Souvenirs Service, liens partage TTL | Galerie, ajout média, vue album public | Politique d'accès public/privé |
| Bloc 6 | Job queue BullMQ, FCM/APNs | Permissions, device token, deep links | Format payload notification |
| Bloc 7 | Endpoint delta sync, versioning | SQLite, NetInfo, queue offline | Stratégie de merge, résolution conflits |
| Bloc 8 | Tests intégration, audit sécu, monitoring | Tests E2E Detox, builds stores, Sentry | Revue de code croisée |

---

## 7. JALONS ET POINTS DE CONTRÔLE

| Jalon | Après le bloc | Livrable attendu | Validateur |
|---|---|---|---|
| **J1 — Socle prêt** | Bloc 1 | App deployable avec auth fonctionnelle sur staging | Tech Lead |
| **J2 — MVP navigable** | Bloc 2 | Parcours complet création voyage → itinéraire consultable | Product Owner + UX |
| **J3 — MVP complet** | Blocs 3+4+5+6 | Tous les modules fonctionnels assemblés | Product Owner |
| **J4 — Offline validé** | Bloc 7 | App fonctionnelle en mode avion sur device physique | QA + Tech Lead |
| **J5 — Go Production** | Bloc 8 | Build stable sur TestFlight + Play Store Internal | Toute l'équipe |

> Chaque jalon donne lieu à une **démo de 30 minutes** avec le porteur de projet avant de passer au bloc suivant. Aucun bloc suivant ne démarre sans sign-off explicite sur le jalon précédent.

---

## 8. RISQUES IDENTIFIÉS ET MESURES D'ATTÉNUATION

| # | Risque | Probabilité | Impact | Mesure |
|---|---|---|---|---|
| R1 | API de traduction (DeepL/Google) : quotas dépassés ou latence élevée | Moyen | Moyen | Cache Redis côté Outils Service, fallback message clair, monitoring quota |
| R2 | Taux de change : API Fixer.io indisponible | Faible | Faible | Cache Redis 24h, affichage "taux du JJ/MM/AAAA" pour transparence |
| R3 | Upload documents : fichiers corrompus ou formats non supportés | Moyen | Moyen | Validation MIME type côté backend, message d'erreur explicite côté mobile |
| R4 | Offline : conflits de données à la resynchronisation | Moyen | Élevé | Stratégie "server wins" pour le MVP, log des conflits, résolution manuelle si nécessaire |
| R5 | FCM/APNs : tokens expirés ou révoqués en masse | Faible | Moyen | Nettoyage automatique tokens invalides (B6-BE-07), logs erreurs d'envoi |
| R6 | Dérive de périmètre : ajout de features B2B pendant le MVP | Élevé | Élevé | Décision PO documentée dans le journal : B2B hors périmètre MVP — tout ajout = nouveau cycle |
| R7 | Performance mobile : app lente sur devices d'entrée de gamme | Moyen | Moyen | Tests obligatoires sur Samsung Galaxy A53 (Bloc 8), lazy loading images dès Bloc 5 |

---

## 9. CONVENTIONS TECHNIQUES À RESPECTER DÈS LE BLOC 0

Ces conventions doivent être décidées et documentées **avant** le premier commit de code.

### Conventions API
- Toutes les routes préfixées `/v1/`
- Réponses JSON standardisées : `{ data: {}, error: null, meta: {} }`
- Codes HTTP sémantiques : 200 / 201 / 400 / 401 / 403 / 404 / 422 / 500
- Pagination systématique sur les listes : `?page=1&limit=20`
- Timestamps en ISO 8601 UTC dans toutes les réponses

### Conventions Mobile
- Aucun appel API direct depuis un composant UI — passer par le Service Layer
- Aucune clé API en dur dans le code — variables d'environnement `.env` + exclusion `.gitignore`
- Les écrans ne contiennent pas de logique métier — séparation stricte UI / logique
- TypeScript strict mode activé dès le Bloc 0

### Conventions Git
- Branches : `feature/bloc-X-nom-tache`, `fix/description`, `release/vX.X`
- Commit message : `[BLOC-X] type: description` (ex: `[BLOC-2] feat: add trip creation endpoint`)
- Pull Request obligatoire pour tout merge sur `main`
- Revue de code minimale : 1 approbation requise

---

## 10. CONDITIONS DE LIVRAISON FINALE

L'application est considérée **prête pour une mise en production** lorsque **toutes** les conditions suivantes sont réunies :

### Fonctionnelles
- [ ] Les 3 parcours critiques (création voyage, import document, partage souvenir) passent de bout en bout sans intervention manuelle
- [ ] Le mode offline fonctionne après premier chargement connecté sur les 4 devices de test
- [ ] Les notifications sont reçues dans les délais configurés (tolérance : ±2 minutes)
- [ ] Les documents importés sont accessibles depuis n'importe quel appareil connecté au même compte

### Techniques
- [ ] 0 crash bloquant enregistré sur les 3 parcours critiques en Sentry
- [ ] Temps de démarrage < 3 secondes sur device d'entrée de gamme (Samsung Galaxy A53)
- [ ] Couverture de tests E2E sur les 3 parcours critiques validée en CI
- [ ] Documentation Swagger complète et à jour pour tous les endpoints `/v1/`
- [ ] Aucune clé API ou secret exposé dans le code source (audit `.env` + historique Git)

### Déploiement
- [ ] Build iOS disponible sur TestFlight avec au moins 5 testeurs externes
- [ ] Build Android disponible sur Play Store Internal Testing
- [ ] Environnements staging et production séparés (BDD, variables, URLs)
- [ ] Runbook de déploiement rédigé (procédure de rollback incluse)
- [ ] Monitoring Sentry + alertes uptime actifs en production

### Produit
- [ ] Sign-off Product Owner sur chaque jalon J1 à J4 documenté
- [ ] Aucune user story "Must Have" (US-A01, A02, A03, B01, B02, B03, C01) en statut ouvert
- [ ] Démo finale réalisée sur device physique (pas simulateur) devant le porteur de projet

---

## SYNTHÈSE EXÉCUTIVE

| Dimension | Décision |
|---|---|
| **Durée totale estimée** | ~11,5 semaines (équipe 2 devs full-time) |
| **Nombre de blocs** | 9 blocs séquencés (0 → 8) |
| **Parallélisations possibles** | Blocs 3, 4, 6 après validation Bloc 2 |
| **Jalons de validation** | 5 jalons avec sign-off PO obligatoire |
| **Périmètre MVP** | B2C uniquement — B2B / white label = cycle 2 |
| **Stack validée** | React Native · Node.js · PostgreSQL · Redis · S3 |
| **Premier livrable testable** | Fin Bloc 1 (~2 semaines) : auth fonctionnelle sur staging |
| **Risque principal** | Dérive de périmètre B2B — décision PO à maintenir fermement |