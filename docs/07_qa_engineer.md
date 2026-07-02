# qa_engineer

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— QA ENGINEER

# Livrable QA Engineer — Travel Companion App
## MVP B2C · Stratégie de test & Cas de validation complets

---

## 0. PRÉAMBULE & PÉRIMÈTRE

### Contexte du document
Ce livrable couvre la **stratégie de test complète du MVP B2C** de la Travel Companion App, basée sur l'analyse croisée des livrables Product Owner, UX Designer, Architecte, Tech Lead, Frontend Engineer et Backend Engineer.

### Périmètre inclus dans ce cycle de QA
- Module A : Gestion du voyage et itinéraire
- Module B : Outils utilitaires (traducteur, convertisseur, fuseaux horaires)
- Module C : Documents de voyage
- Module D : Souvenirs & Partage
- Module E : Notifications & Rappels
- Module F : Authentification & Onboarding
- Module G : Offline & Synchronisation

### Périmètre explicitement exclu
- Fonctionnalités B2B (back-office agence, white label)
- Interface web companion (hors app mobile principale)
- Page publique album (couverture minimale)

### Stack testée
| Couche | Technologie |
|---|---|
| Mobile | React Native (iOS + Android) |
| Backend | Next.js API Routes + Supabase |
| Base de données | Supabase PostgreSQL avec RLS |
| Storage | Supabase Storage |
| Auth | Supabase Auth (JWT, OAuth Google/Apple) |
| APIs tierces | DeepL, Fixer.io, Timezone API |
| Push | FCM (Android) + APNs (iOS) |

---

## 1. STRATÉGIE DE TEST

### 1.1 Pyramide de test retenue

```
                    ┌─────────────┐
                    │   E2E / UAT │  ← 10% — Parcours critiques uniquement
                    │  (Detox)    │
                ┌───┴─────────────┴───┐
                │  Intégration API    │  ← 30% — Contrats endpoints + RLS
                │  (Jest + Supertest) │
            ┌───┴─────────────────────┴───┐
            │  Tests unitaires            │  ← 60% — Logique métier, utils, stores
            │  (Jest + RTL)               │
            └─────────────────────────────┘
```

### 1.2 Types de tests et outillage

| Type | Outil | Portée | Responsable |
|---|---|---|---|
| Tests unitaires | Jest + React Testing Library | Composants, hooks, stores Zustand, utils | Dev + QA |
| Tests intégration API | Jest + Supertest | Endpoints API Routes, règles RLS Supabase | QA Backend |
| Tests E2E mobile | Detox (iOS + Android) | Parcours utilisateurs critiques | QA Mobile |
| Tests de performance | k6 | Endpoints critiques sous charge | QA Perf |
| Tests offline | Detox + Network mocking | Scénarios déconnexion/reconnexion | QA Mobile |
| Tests accessibilité | @testing-library/jest-native | Composants UI, labels, contrastes | QA + Frontend |
| Tests sécurité | Manuel + OWASP ZAP | Auth, RLS, upload fichiers | QA Sécurité |

### 1.3 Environnements de test

| Environnement | Usage | Données |
|---|---|---|
| **Local** | Développement TDD quotidien | Fixtures mock |
| **Staging** | Validation QA pré-release | Données anonymisées |
| **Production** | Smoke tests post-deploy uniquement | Compte QA dédié |

### 1.4 Seuils de couverture minimaux

| Scope | Seuil cible |
|---|---|
| Logique métier (utils, stores) | ≥ 80% |
| Composants UI critiques | ≥ 70% |
| Endpoints API (couverture contrat) | 100% des endpoints documentés |
| Parcours E2E critiques | 100% des parcours PO Must Have |

### 1.5 Définition du "Done" pour une US

Une User Story est considérée **testée et validée** quand :
- [ ] Tous les critères d'acceptation PO couverts par au moins un cas de test
- [ ] Tous les cas limites identifiés ont un cas de test associé
- [ ] Aucune anomalie de sévérité **Critique** ou **Haute** ouverte
- [ ] Tests unitaires et d'intégration passants en CI
- [ ] Validation manuelle sur device physique iOS ET Android

---

## 2. CAS DE TESTS FONCTIONNELS

### MODULE F — Authentification & Onboarding

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| F-01 | Inscription email valide | Un utilisateur non inscrit sur l'écran d'inscription | Il saisit un email valide et un mot de passe ≥ 8 caractères et soumet | Un compte est créé, il est redirigé vers le Home, sa session est active | Must Have | E2E |
| F-02 | Inscription email déjà utilisé | Un utilisateur existant, un nouvel utilisateur sur l'écran d'inscription | Il saisit l'email déjà utilisé | Un message d'erreur explicite s'affiche, aucun doublon créé en base | Must Have | Intégration |
| F-03 | Inscription mot de passe trop court | Écran d'inscription | Il saisit un mot de passe < 8 caractères et soumet | Validation inline affichée avant soumission, endpoint non appelé | Must Have | Unitaire |
| F-04 | Connexion credentials valides | Compte existant, écran de connexion | Il saisit email + mot de passe corrects | JWT émis, session persistée, redirection vers Home | Must Have | E2E |
| F-05 | Connexion mot de passe incorrect | Compte existant, écran de connexion | Il saisit un mauvais mot de passe | Message d'erreur générique (ne pas préciser si c'est l'email ou le mdp), pas de session créée | Must Have | Intégration |
| F-06 | Connexion Google | Écran de connexion | Il tape "Continuer avec Google" et complète le flux OAuth | Compte créé ou lié, session active, redirection Home | Must Have | E2E |
| F-07 | Connexion Apple (iOS uniquement) | Appareil iOS, écran de connexion | Il tape "Continuer avec Apple" et complète le flux | Compte créé ou lié, session active | Must Have | E2E |
| F-08 | Persistance de session | Utilisateur connecté | Il ferme et rouvre l'app | L'utilisateur est toujours connecté sans ressaisir ses credentials | Must Have | E2E |
| F-09 | Refresh token silencieux | Utilisateur connecté, access token expiré | Il effectue une action nécessitant auth | Le refresh s'effectue silencieusement, l'action aboutit sans interruption | Must Have | Intégration |
| F-10 | Accès route protégée sans auth | Aucune session active | Il tente d'accéder à `/dashboard` directement | Redirection vers Onboarding, aucune donnée exposée | Must Have | Intégration |
| F-11 | Déconnexion | Utilisateur connecté | Il tape "Déconnexion" dans Paramètres | Session détruite côté client ET serveur (refresh token invalidé), redirection Onboarding | Must Have | E2E |
| F-12 | Onboarding 3 slides | Premier lancement, aucun compte | Il swipe les 3 slides | Les 3 slides s'affichent dans l'ordre, le CTA "Commencer" est visible sur le dernier | Should Have | E2E |

---

### MODULE A — Gestion des voyages & Itinéraire

#### US-A01 — Création d'un voyage

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| A01-01 | Création voyage complet | Utilisateur connecté, Home vide | Il remplit nom, destination, date début, date fin et valide | Le voyage apparaît dans le tableau de bord, statut calculé automatiquement, créateur = admin | Must Have | E2E |
| A01-02 | Création sans participants | Utilisateur connecté | Il crée un voyage sans renseigner les participants | Le voyage est créé, champ participants reste vide, pas d'erreur | Must Have | E2E |
| A01-03 | Statut "à venir" automatique | Voyage créé avec date début future | Création validée | Statut = "upcoming" | Must Have | Unitaire |
| A01-04 | Statut "en cours" automatique | Voyage créé avec dates encadrant aujourd'hui | Création validée | Statut = "active" | Must Have | Unitaire |
| A01-05 | Statut "passé" automatique | Voyage créé avec date fin passée | Création validée | Statut = "past" | Must Have | Unitaire |
| A01-06 | Date fin antérieure à date début | Formulaire de création | Il saisit une date de fin < date de début | Erreur de validation inline, soumission bloquée | Must Have | Unitaire |
| A01-07 | Champs obligatoires manquants | Formulaire de création | Il soumet sans nom ou sans destination | Messages d'erreur inline sur chaque champ manquant | Must Have | Unitaire |
| A01-08 | Créateur = administrateur | Voyage créé | Consultation des participants | Le créateur apparaît avec le rôle "admin" automatiquement | Must Have | Intégration |
| A01-09 | Voyage visible uniquement par son créateur | Deux comptes distincts, un voyage créé par User A | User B consulte ses voyages | Le voyage de User A n'apparaît pas (RLS Supabase) | Must Have | Intégration |

#### US-A02 — Consultation de l'itinéraire

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| A02-01 | Vue chronologique jour par jour | Voyage avec étapes créées | L'utilisateur ouvre l'itinéraire | Les étapes s'affichent par jour, triées chronologiquement par horaire | Must Have | E2E |
| A02-02 | Détail d'une étape | Itinéraire affiché | L'utilisateur tape sur une étape | Lieu, horaire, notes, documents associés, bouton Maps s'affichent | Must Have | E2E |
| A02-03 | Accès offline à l'itinéraire | Données précédemment chargées, connexion coupée | L'utilisateur ouvre l'itinéraire | L'itinéraire s'affiche en lecture seule depuis le cache, indicateur offline visible | Must Have | E2E |
| A02-04 | Itinéraire vide — état guidé | Voyage créé sans étapes | L'utilisateur ouvre l'itinéraire | Un état vide guidé avec 2-3 actions claires s'affiche (pas un écran blanc) | Must Have | E2E |
| A02-05 | Navigation jour précédent/suivant | Itinéraire multi-jours | L'utilisateur swipe ou navigue entre les jours | Le contexte de jour change, les étapes correspondantes s'affichent | Must Have | E2E |

#### US-A03 — Import de documents

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| A03-01 | Import PDF | Voyage ouvert, section Documents | L'utilisateur importe un fichier PDF ≤ 20 Mo | Le document est uploadé, associé à une catégorie, accessible dans la liste | Must Have | E2E |
| A03-02 | Import JPG | Même contexte | L'utilisateur importe un JPG | Idem | Must Have | E2E |
| A03-03 | Import PNG | Même contexte | L'utilisateur importe un PNG | Idem | Must Have | E2E |
| A03-04 | Format non supporté | Section Documents | L'utilisateur tente d'importer un .docx ou .xlsx | Message d'erreur explicite, fichier refusé | Must Have | Intégration |
| A03-05 | Fichier trop volumineux | Section Documents | L'utilisateur tente d'importer un fichier > 20 Mo | Message d'erreur "Fichier trop volumineux (max 20 Mo)", upload bloqué avant envoi | Must Have | Unitaire + Intégration |
| A03-06 | Association catégorie | Document importé | L'utilisateur assigne la catégorie "Vol" | Le document apparaît sous la catégorie "Vol", filtrable | Must Have | E2E |
| A03-07 | Accès document offline | Document importé et mis en cache | Connexion coupée | Le document est consultable offline | Must Have | E2E |
| A03-08 | Sécurité RLS — accès inter-utilisateurs | Document appartenant à User A | User B tente d'accéder via URL directe | Accès refusé (403), document non exposé | Must Have | Intégration |
| A03-09 | Upload concurrent | Section Documents | L'utilisateur tente d'uploader 3 fichiers simultanément | Les 3 uploads s'exécutent ou sont mis en queue sans erreur | Should Have | Intégration |

#### US-A04 — Notifications & Rappels

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| A04-01 | Rappel 24h avant étape | Étape avec heure définie, notifications activées | 24h avant l'heure de l'étape | Notification push reçue sur le device | Should Have | E2E |
| A04-02 | Rappel 2h avant étape | Même contexte | 2h avant | Notification push reçue | Should Have | E2E |
| A04-03 | Fuseau horaire respecté | Étape à Tokyo (UTC+9), utilisateur en Paris (UTC+1) | Heure de rappel calculée | La notification arrive à l'heure correcte selon le fuseau de l'étape | Should Have | Unitaire + Intégration |
| A04-04 | Notifications désactivées dans params | Paramètre notifications = OFF | Délai de rappel atteint | Aucune notification envoyée | Should Have | Intégration |
| A04-05 | Rappel délai personnalisé | Délai custom configuré (ex. 48h) | 48h avant l'étape | Notification reçue selon le délai personnalisé | Should Have | Intégration |

---

### MODULE B — Outils Utilitaires

#### US-B01 — Traducteur

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| B01-01 | Langue cible pré-sélectionnée | Voyage actif vers le Japon (destination_language = "ja") | L'utilisateur ouvre le traducteur | La langue cible est pré-remplie avec "Japonais" | Must Have | E2E |
| B01-02 | Traduction en mode connecté | Traducteur ouvert, connexion active | L'utilisateur saisit "Bonjour" et demande la tra

— QA ENGINEER

*(Reprise exacte à partir du dernier cas de test produit — B01-02)*

| B01-02 | Traduction en mode connecté | Traducteur ouvert, connexion active | L'utilisateur saisit "Bonjour" et demande la traduction | Résultat affiché en moins de 3 secondes | Must Have | E2E |
| B01-03 | Traduction hors connexion | Traducteur ouvert, connexion coupée | L'utilisateur tente une traduction | Message informatif affiché ("Traduction indisponible hors connexion"), aucune erreur bloquante | Must Have | E2E |
| B01-04 | Pack hors ligne proposé | Mode offline détecté | L'utilisateur ouvre le traducteur | Un CTA "Télécharger le pack hors ligne" est proposé (si non déjà téléchargé) | Should Have | E2E |
| B01-05 | Lecture audio de la traduction | Traduction effectuée | L'utilisateur tape le bouton lecture audio | La prononciation est jouée dans la langue cible | Must Have | E2E |
| B01-06 | Changement de langue cible manuel | Traducteur ouvert | L'utilisateur change la langue cible manuellement | La traduction se relance avec la nouvelle langue, la sélection persiste pendant la session | Must Have | E2E |
| B01-07 | Texte vide soumis | Traducteur ouvert | L'utilisateur soumet sans saisir de texte | Aucun appel API effectué, message ou champ vide affiché proprement | Must Have | Unitaire |
| B01-08 | Texte très long (> 500 caractères) | Traducteur ouvert | L'utilisateur saisit un texte dépassant la limite | Un compteur de caractères est visible, ou le texte est tronqué avec information claire | Should Have | Intégration |
| B01-09 | Clé API DeepL non exposée côté client | Inspection réseau (DevTools / proxy) | L'utilisateur effectue une traduction | Aucune clé API tierce visible dans les requêtes client, appel via API Route Next.js uniquement | Must Have | Sécurité |

---

#### US-B02 — Convertisseur de devises

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| B02-01 | Devise pré-remplie selon voyage actif | Voyage actif avec destination_currency = "JPY" | L'utilisateur ouvre le convertisseur | La devise cible est pré-remplie avec "JPY (Yen japonais)" | Must Have | E2E |
| B02-02 | Conversion valide | Convertisseur ouvert, connexion active | L'utilisateur saisit 100 EUR → JPY | Le montant converti s'affiche avec le taux de change et la date de la dernière mise à jour | Must Have | E2E |
| B02-03 | Taux mis en cache | Taux chargé il y a moins de 1h | L'utilisateur ouvre le convertisseur sans connexion | Le résultat est affiché depuis le cache avec mention "Taux mis à jour le [date/heure]" | Must Have | Intégration |
| B02-04 | Cache expiré hors connexion | Taux en cache > 24h, connexion coupée | L'utilisateur ouvre le convertisseur | Taux affiché avec avertissement "Taux potentiellement obsolète", conversion toujours possible | Should Have | Intégration |
| B02-05 | Inversion devises | Convertisseur affiché avec EUR → JPY | L'utilisateur tape le bouton d'inversion | Les devises sont échangées (JPY → EUR), le calcul se relance | Must Have | E2E |
| B02-06 | Montant négatif ou zéro | Convertisseur ouvert | L'utilisateur saisit "-50" ou "0" | Message de validation, conversion bloquée ou résultat = 0 affiché sans erreur serveur | Must Have | Unitaire |
| B02-07 | Montant non numérique | Convertisseur ouvert | L'utilisateur saisit "abc" | Champ en erreur, aucun appel API déclenché | Must Have | Unitaire |
| B02-08 | Clé API Fixer.io non exposée | Inspection réseau | L'utilisateur effectue une conversion | Aucune clé API visible côté client | Must Have | Sécurité |

---

#### US-B03 — Horloge / Fuseaux horaires

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| B03-01 | Heure locale destination affichée | Voyage actif avec destination_timezone = "Asia/Tokyo" | L'utilisateur ouvre l'horloge | L'heure locale de Tokyo s'affiche en temps réel | Must Have | E2E |
| B03-02 | Comparatif avec heure utilisateur | Horloge ouverte | Affichage | L'heure locale de l'utilisateur (device) et l'heure de la destination sont affichées simultanément avec le décalage (+Xh) | Must Have | E2E |
| B03-03 | Décalage horaire calculé correctement | Voyage Paris → Tokyo (UTC+1 vs UTC+9) | Affichage horloge | Décalage = +8h affiché correctement | Must Have | Unitaire |
| B03-04 | Heure en temps réel | Horloge ouverte | L'utilisateur laisse l'écran ouvert 2 minutes | L'heure se met à jour en temps réel (pas figée au moment de l'ouverture) | Must Have | E2E |
| B03-05 | Aucun voyage actif | Utilisateur sans voyage actif sélectionné | Il ouvre l'horloge | Un état neutre s'affiche avec message invitant à sélectionner un voyage, pas d'erreur | Should Have | E2E |

---

### MODULE C — Souvenirs & Partage

| ID | Scénario | Given | When | Then | Priorité | Type |
|---|---|---|---|---|---|---|
| C01-01 | Ajout d'une photo souvenir | Voyage ouvert, section Souvenirs | L'utilisateur sélectionne une photo depuis sa galerie et valide | La photo apparaît dans la galerie du voyage avec date et lieu si disponibles | Must Have | E2E |
| C01-02 | Ajout d'une note souvenir | Section Souvenirs | L'utilisateur saisit une note texte et valide | La note apparaît dans la galerie du voyage | Must Have | E2E |
| C01-03 | Ajout d'un lieu à un souvenir | Formulaire ajout souvenir | L'utilisateur renseigne un lieu (texte libre ou géolocalisation) | Le lieu est associé au souvenir et visible dans le détail | Should Have | E2E |
| C01-04 | Génération d'un lien de partage | Album souvenir existant | L'utilisateur tape "Partager l'album" | Un lien unique est généré, copiable ou partageable via le share sheet natif | Must Have | E2E |
| C01-05 | Accès public à un album partagé | Lien d'album partagé valide | Un proche (non inscrit) ouvre le lien | La page publique `/share/[albumId]` s'affiche avec les souvenirs, sans nécessiter de connexion | Must Have | E2E |
| C01-06 | Accès à un album inexistant | Lien invalide ou supprimé | Un visiteur ouvre le lien | Page 404 claire affichée, pas d'erreur serveur | Must Have | Intégration |
| C01-07 | Isolation des souvenirs entre voyages | Deux voyages avec souvenirs distincts | L'utilisateur consulte la galerie du Voyage A | Seuls les souvenirs du Voyage A s'affichent, aucun souvenir du Voyage B | Must Have | Intégration |
| C01-08 | Photo trop volumineuse | Section Souvenirs | L'utilisateur tente d'uploader une photo > 20 Mo | Message d'erreur, upload bloqué | Must Have | Unitaire + Intégration |
| C01-09 | Suppression d'un souvenir | Souvenir existant dans la galerie | L'utilisateur supprime un souvenir | Le souvenir disparaît de la galerie et du stockage (soft delete ou hard delete selon spec) | Should Have | E2E |
| C01-10 | L'album partagé ne donne pas accès aux documents privés | Album partagé via lien public | Un visiteur extérieur consulte l'album | Seuls les médias explicitement ajoutés aux souvenirs sont visibles, aucun document de voyage n'est accessible | Must Have | Sécurité |

---

## 4. CAS LIMITES ET SCÉNARIOS D'ERREUR

| ID | Catégorie | Scénario | Comportement attendu | Sévérité si KO |
|---|---|---|---|---|
| E-01 | Réseau | Perte de connexion pendant un upload de document | Upload interrompu proprement, message d'erreur, bouton "Réessayer" proposé | Haute |
| E-02 | Réseau | Timeout API traduction (> 5s sans réponse) | Message d'erreur clair, aucun spinner infini | Haute |
| E-03 | Auth | Token JWT falsifié envoyé à l'API | 401 retourné, session détruite côté client, redirection Onboarding | Critique |
| E-04 | Données | Voyage avec 0 étape ouvert en offline | État vide affiché depuis le cache, pas d'erreur | Moyenne |
| E-05 | Données | Suppression d'un voyage avec documents associés | Cascade correcte : documents supprimés du storage + DB (pas de fichiers orphelins) | Haute |
| E-06 | Stockage | Quota S3 / Supabase Storage atteint | Message d'erreur explicite, l'utilisateur est informé, pas de crash silencieux | Haute |
| E-07 | Concurrence | Deux appareils modifient le même voyage simultanément | Le dernier écrit gagne (last-write-wins) ou conflit détecté avec message, pas de corruption | Haute |
| E-08 | Sécurité | Injection dans les champs texte (XSS, SQLi) | Inputs sanitisés côté serveur, aucune exécution de code, RLS PostgreSQL actif | Critique |
| E-09 | Performance | Chargement d'un voyage avec 200+ étapes | L'itinéraire se charge en moins de 3 secondes (pagination ou virtualisation activée) | Moyenne |
| E-10 | Notification | Push token invalide ou révoqué | Aucun crash, échec silencieux loggué, pas de retry infini | Moyenne |
| E-11 | Upload | Fichier PDF corrompu importé | Message d'erreur clair, fichier rejeté, pas de crash serveur | Haute |
| E-12 | Navigation | Deep link vers un voyage supprimé | Redirection vers Home avec message "Voyage introuvable", pas d'écran blanc | Moyenne |
| E-13 | OAuth | Flux Google interrompu (back avant complétion) | Retour à l'écran de connexion, aucune session partielle créée | Haute |
| E-14 | Offline | Synchronisation au retour en ligne après modifications offline | Les données cachées se synchronisent sans écraser les modifications faites par un autre participant | Haute |

---

## 5. VÉRIFICATION DES CRITÈRES D'ACCEPTATION PO

### Tableau de couverture par User Story

| US | Critère d'acceptation (PO) | Cas de test(s) couvrant | Statut couverture |
|---|---|---|---|
| A01 | Saisie : nom, destination, dates, participants | A01-01, A01-02, A01-07 | ✅ Couverte |
| A01 | Apparition dans tableau de bord avec statut | A01-03, A01-04, A01-05 | ✅ Couverte |
| A01 | Créateur = administrateur automatique | A01-08 | ✅ Couverte |
| A02 | Vue chronologique par jour | A02-01 | ✅ Couverte |
| A02 | Détail étape (lieu, horaire, notes, docs) | A02-02 | ✅ Couverte |
| A02 | Accès offline en lecture seule | A02-03 | ✅ Couverte |
| A03 | Formats acceptés PDF / JPG / PNG | A03-01, A03-02, A03-03 | ✅ Couverte |
| A03 | Association à une catégorie | A03-06 | ✅ Couverte |
| A03 | Accessible offline | A03-07 | ✅ Couverte |
| A03 | Taille max 20 Mo | A03-05 | ✅ Couverte |
| A04 | Rappel 24h et 2h | A04-01, A04-02 | ✅ Couverte |
| A04 | Notifications désactivables | A04-04 | ✅ Couverte |
| A04 | Fuseau horaire de l'étape respecté | A04-03 | ✅ Couverte |
| B01 | Langue cible pré-sélectionnée selon destination | B01-01 | ✅ Couverte |
| B01 | Résultat < 3s en mode connecté | B01-02 | ✅ Couverte |
| B01 | Message offline + proposition pack | B01-03, B01-04 | ✅ Couverte |
| B01 | Lecture audio de la traduction | B01-05 | ✅ Couverte |
| B02 | Devise pré-remplie depuis voyage actif | B02-01 | ✅ Couverte |
| B02 | Taux de change avec date de mise à jour | B02-02 | ✅ Couverte |
| B02 | Fonctionnement offline (cache) | B02-03, B02-04 | ✅ Couverte |
| B03 | Heure locale destination affichée | B03-01 | ✅ Couverte |
| B03 | Comparatif heure locale utilisateur | B03-02 | ✅ Couverte |
| C01 | Ajout photo, note, lieu | C01-01, C01-02, C01-03 | ✅ Couverte |
| C01 | Génération lien de partage | C01-04 | ✅ Couverte |
| C01 | Accès invités sans inscription | C01-05 | ✅ Couverte |

**Taux de couverture des critères d'acceptation PO : 25/25 critères couverts — 100%**

---

## 6. RAPPORT DE VALIDATION — STATUT PAR CRITÈRE

> Ce rapport est produit en **pré-validation** (avant exécution des tests). Il constitue la grille de réception utilisée lors de la phase QA du Bloc 9 (Semaine 12 selon

— QA ENGINEER

*(Suite et fin du livrable — reprise après "Il constitue la grille de réception utilisée lors de la phase QA du Bloc 9 (Semaine 12 selon")*

le plan d'exécution Tech Lead).

---

### 6.1 Grille de réception — Module A (Voyage & Itinéraire)

| Critère | Description | Cas de test | Statut attendu | Sévérité si KO |
|---|---|---|---|---|
| A01-C1 | Création voyage avec tous les champs | A01-01, A01-02 | ⬜ À tester | Bloquant |
| A01-C2 | Participants optionnels | A01-07 | ⬜ À tester | Majeur |
| A01-C3 | Statut auto (à venir / en cours / passé) | A01-03, A01-04, A01-05 | ⬜ À tester | Majeur |
| A01-C4 | Créateur = administrateur | A01-08 | ⬜ À tester | Majeur |
| A02-C1 | Vue chronologique jour par jour | A02-01 | ⬜ À tester | Bloquant |
| A02-C2 | Détail complet d'une étape | A02-02 | ⬜ À tester | Majeur |
| A02-C3 | Lecture offline via cache | A02-03 | ⬜ À tester | Majeur |
| A03-C1 | Import PDF / JPG / PNG acceptés | A03-01, A03-02, A03-03 | ⬜ À tester | Bloquant |
| A03-C2 | Formats non supportés rejetés | A03-04 | ⬜ À tester | Majeur |
| A03-C3 | Limite 20 Mo respectée | A03-05 | ⬜ À tester | Majeur |
| A03-C4 | Association à une catégorie | A03-06 | ⬜ À tester | Majeur |
| A03-C5 | Document accessible offline | A03-07 | ⬜ À tester | Majeur |
| A04-C1 | Rappel 24h avant étape | A04-01 | ⬜ À tester | Mineur |
| A04-C2 | Rappel 2h avant étape | A04-02 | ⬜ À tester | Mineur |
| A04-C3 | Fuseau horaire de l'étape respecté | A04-03 | ⬜ À tester | Majeur |
| A04-C4 | Désactivation notifications globale | A04-04 | ⬜ À tester | Mineur |

---

### 6.2 Grille de réception — Module B (Outils utilitaires)

| Critère | Description | Cas de test | Statut attendu | Sévérité si KO |
|---|---|---|---|---|
| B01-C1 | Langue cible pré-sélectionnée | B01-01 | ⬜ À tester | Majeur |
| B01-C2 | Résultat traduction < 3s connecté | B01-02 | ⬜ À tester | Majeur |
| B01-C3 | Message clair en mode offline | B01-03, B01-04 | ⬜ À tester | Majeur |
| B01-C4 | Lecture audio disponible | B01-05 | ⬜ À tester | Mineur |
| B01-C5 | Clé DeepL non exposée client | B01-06 | ⬜ À tester | Bloquant |
| B02-C1 | Devise pré-remplie depuis voyage actif | B02-01 | ⬜ À tester | Majeur |
| B02-C2 | Taux affiché avec date de mise à jour | B02-02 | ⬜ À tester | Mineur |
| B02-C3 | Fonctionnement offline (taux cachés) | B02-03, B02-04 | ⬜ À tester | Majeur |
| B02-C4 | Clé Fixer.io non exposée client | B02-08 | ⬜ À tester | Bloquant |
| B03-C1 | Heure locale destination affichée | B03-01 | ⬜ À tester | Majeur |
| B03-C2 | Comparatif heure utilisateur affiché | B03-02 | ⬜ À tester | Majeur |
| B03-C3 | Heure en temps réel (mise à jour) | B03-04 | ⬜ À tester | Mineur |
| B03-C4 | État neutre sans voyage actif | B03-05 | ⬜ À tester | Mineur |

---

### 6.3 Grille de réception — Module C (Souvenirs & Partage)

| Critère | Description | Cas de test | Statut attendu | Sévérité si KO |
|---|---|---|---|---|
| C01-C1 | Ajout photo souvenir | C01-01 | ⬜ À tester | Bloquant |
| C01-C2 | Ajout note souvenir | C01-02 | ⬜ À tester | Bloquant |
| C01-C3 | Association d'un lieu | C01-03 | ⬜ À tester | Mineur |
| C01-C4 | Génération lien de partage | C01-04 | ⬜ À tester | Bloquant |
| C01-C5 | Accès album sans inscription | C01-05 | ⬜ À tester | Bloquant |
| C01-C6 | 404 sur lien invalide | C01-06 | ⬜ À tester | Majeur |
| C01-C7 | Isolation souvenirs par voyage | C01-07 | ⬜ À tester | Bloquant |
| C01-C8 | Rejet photo > 20 Mo | C01-08 | ⬜ À tester | Majeur |
| C01-C9 | Album partagé sans accès aux docs privés | C01-10 | ⬜ À tester | Bloquant |

---

### 6.4 Grille de réception — Sécurité transversale

| Critère | Description | Cas de test | Statut attendu | Sévérité si KO |
|---|---|---|---|---|
| SEC-01 | Isolation des données entre utilisateurs | A01-09, A02-04 | ⬜ À tester | Critique |
| SEC-02 | Token JWT falsifié rejeté | E-03 | ⬜ À tester | Critique |
| SEC-03 | Sanitisation des inputs (XSS / SQLi) | E-08 | ⬜ À tester | Critique |
| SEC-04 | RLS PostgreSQL actif et fonctionnel | A01-09, C01-10 | ⬜ À tester | Critique |
| SEC-05 | Aucune clé API tierce exposée client | B01-C5, B02-C4 | ⬜ À tester | Critique |
| SEC-06 | Route protégée inaccessible sans auth | B1-FE-06 (Tech Lead) | ⬜ À tester | Critique |

---

## 7. LISTE DES ANOMALIES POTENTIELLES (Pré-identification des risques)

> Ces anomalies sont identifiées **a priori** sur la base de l'analyse des livrables. Elles seront confirmées ou infirmées lors de l'exécution des tests en Semaine 12.

---

### 7.1 Anomalies de niveau CRITIQUE

| ID | Module | Description | Impact | Recommandation |
|---|---|---|---|---|
| ANO-C01 | Auth | Absence de validation d'expiration du refresh token côté serveur : un token volé pourrait rester valide indéfiniment | Usurpation de compte | Implémenter un TTL strict sur les refresh tokens + table de révocation en base |
| ANO-C02 | RLS | Si les politiques Row Level Security Supabase ne sont pas testées unitairement, un utilisateur pourrait accéder aux voyages d'un autre via un appel API direct | Fuite de données | Créer une suite de tests RLS dédiée avant tout déploiement en staging |
| ANO-C03 | Partage | La page `/share/[albumId]` publique pourrait exposer les métadonnées du voyage (destination, dates, participants) si le back ne filtre pas strictement les champs retournés | Exposition de données personnelles | Définir un DTO strict pour la vue publique, ne retournant que les médias et titres explicitement partagés |
| ANO-C04 | Outils | Si le proxy API Routes Next.js ne valide pas les inputs avant de les transmettre à DeepL ou Fixer.io, une injection ou une surcharge de requêtes est possible | Sécurité + coût API | Ajouter une validation Zod sur tous les inputs des routes `/api/tools/*` |

---

### 7.2 Anomalies de niveau HAUT

| ID | Module | Description | Impact | Recommandation |
|---|---|---|---|---|
| ANO-H01 | Offline | La stratégie de synchronisation lors du retour en ligne n'est pas spécifiée avec précision dans les livrables. Le comportement en cas de conflit (deux participants modifient la même étape offline) est indéfini. | Corruption de données | Définir une politique explicite (last-write-wins avec timestamp, ou détection de conflit) avant la Semaine 10 |
| ANO-H02 | Documents | L'accès hors connexion aux documents repose sur un cache local. Si la taille du cache n'est pas limitée (ex. 50 documents PDF de 20 Mo chacun = 1 Go), l'app saturera le stockage du device | Crash app / expérience dégradée | Définir une politique de cache LRU avec quota configurable (ex. 200 Mo max en local) |
| ANO-H03 | Notifications | Les rappels sont planifiés côté serveur (Notification Service), mais si l'utilisateur est dans un fuseau horaire différent de la destination lors de la création de l'étape, le calcul de l'heure d'envoi doit utiliser le fuseau de l'étape, pas du serveur. Ce point n'est pas explicitement garanti dans le Bloc 6 du Tech Lead. | Rappel envoyé au mauvais moment | Préciser dans les specs du Notification Service que `step_date + start_time` sont toujours interprétés en `destination_timezone` |
| ANO-H04 | Voyage | Le calcul automatique du statut `upcoming / active / past` repose sur la comparaison entre `start_date / end_date` et la date courante. Si ce calcul est fait côté client, des divergences apparaîtront entre utilisateurs de fuseaux horaires différents. | Statut incohérent | Ce calcul doit être normalisé côté serveur (UTC) |
| ANO-H05 | Souvenirs | Aucun mécanisme de modération ou de révocation d'un lien partagé n'est mentionné dans les livrables. Un lien partagé reste actif même si le voyage est supprimé. | Exposition de données après suppression | Ajouter un endpoint de révocation de lien et implémenter la cascade de suppression sur `shared_albums` lors de la suppression du voyage |
| ANO-H06 | Upload | L'import de document est décrit comme acceptant JPG, PNG, PDF. Aucune validation du MIME type côté serveur n'est mentionnée (une validation côté client uniquement est insuffisante). | Upload de fichiers malveillants | Implémenter une validation MIME type + magic bytes côté serveur, indépendamment de l'extension |

---

### 7.3 Anomalies de niveau MOYEN

| ID | Module | Description | Impact | Recommandation |
|---|---|---|---|---|
| ANO-M01 | UX / Offline | L'indicateur visuel "offline" est décrit par l'UX Designer mais son déclenchement exact (immédiat ? après X secondes ?) n'est pas spécifié. Un faux positif (réseau lent ≠ offline) dégraderait l'expérience. | Confusion utilisateur | Définir un délai de grâce (ex. 5s sans réponse réseau) avant d'afficher le bandeau offline |
| ANO-M02 | Itinéraire | La vue "Aujourd'hui" est mise en avant dans les parcours UX, mais aucun cas de test ne vérifie le comportement si `start_date` = `end_date` (voyage d'un seul jour) ou si `start_date` est dans le passé mais `end_date` dans le futur (voyage en cours). | Affichage incorrect | Ajouter des cas de test pour les voyages monojours et les voyages straddling (cheval sur plusieurs jours) |
| ANO-M03 | Participants | Le système d'invitation par email n'est pas testé pour le cas d'une invitation vers un email déjà inscrit vs un email inconnu. Les deux flux (lien vers app vs email d'inscription) doivent être distincts. | Expérience dégradée | Tester explicitement les deux cas : email existant (liaison automatique au compte) vs email inconnu (invitation à s'inscrire) |
| ANO-M04 | Outils | Le convertisseur de devises affiche un "taux en cache offline". La fraîcheur de ce cache n'est pas définie. Si le taux a plusieurs jours, l'utilisateur pourrait effectuer une dépense erronée. | Information trompeuse | Afficher explicitement la date/heure du dernier taux connu, avec mention "Taux indicatif - actualisé le [date]" |
| ANO-M05 | Performance | Le livrable Architecte mentionne la pagination pour les voyages avec 200+ étapes, mais ce n'est pas testé. Une régression de performance pourrait passer inaperçue. | Dégradation performance | Inclure un test de charge minimal en Semaine 12 (voyage avec 50 étapes, 20 documents) |

---

### 7.4 Anomalies de niveau MINEUR

| ID | Module | Description | Impact | Recommandation |
|---|---|---|---|---|
| ANO-L01 | Onboarding | L'onboarding est décrit en "3 slides max" mais aucune spec de contenu n'est validée. Un utilisateur peut ne pas comprendre la proposition de valeur. | Taux de conversion onboarding | Valider le contenu des slides avec le PO avant le Bloc 8 |
| ANO-L02 | Auth | L'écran de connexion propose OAuth Google et Apple. Sur Android, "Connexion avec Apple" est techniquement possible mais peu commune. L'UX pour ce cas n'est pas précisée. | Confusion utilisateur Android | Décider en amont si Apple Sign-In est proposé sur Android ou uniquement iOS |
| ANO-L03 | Traducteur | La "lecture audio" de la traduction (B01-05) dépend de l'API TTS de DeepL ou d'une API tierce non précisée dans les livrables Backend. Le service

— QA ENGINEER

*(Suite et fin du livrable — reprise après ANO-L03)*

---

| ANO-L03 | Traducteur | La "lecture audio" de la traduction (B01-05) dépend de l'API TTS de DeepL ou d'une API tierce non précisée dans les livrables Backend. Le service utilisé et son fallback en cas d'indisponibilité ne sont pas définis. | Fonctionnalité silencieusement cassée | Identifier et documenter le fournisseur TTS retenu, implémenter un fallback gracieux (masquer le bouton audio si service indisponible) |
| ANO-L04 | Souvenirs | Le module souvenirs permet d'associer un "lieu" à un souvenir, mais aucun livrable ne précise si ce lieu est libre (texte saisi) ou géolocalisé (coordonnées GPS). Une incohérence de données est possible. | Données hétérogènes | Aligner Backend et UX sur la nature du champ lieu : texte libre ou point géographique avec autocomplete |
| ANO-L05 | Notifications | Le délai paramétrable pour les rappels (24h, 2h, personnalisé) n'a pas de valeur plancher définie. Un utilisateur pourrait programmer un rappel à 0 minute, générant une notification immédiate ou un comportement inattendu. | Comportement inattendu | Définir une valeur plancher (ex. minimum 15 minutes avant l'étape) et la valider côté serveur |

---

## 8. MATRICE DE COUVERTURE DES CRITÈRES D'ACCEPTATION PO

> Cette matrice assure la traçabilité totale entre les User Stories du Product Owner et les cas de tests QA.

| US PO | Intitulé | Critères d'acceptation | Cas de tests associés | Couverture |
|---|---|---|---|---|
| US-A01 | Création d'un voyage | 3 critères | A01-01 à A01-09 | ✅ Complète |
| US-A02 | Consultation itinéraire | 3 critères | A02-01 à A02-07 | ✅ Complète |
| US-A03 | Import de documents | 4 critères | A03-01 à A03-09 | ✅ Complète |
| US-A04 | Notifications & rappels | 3 critères | A04-01 à A04-07 | ✅ Complète |
| US-B01 | Traducteur intégré | 4 critères | B01-01 à B01-08 | ✅ Complète |
| US-B02 | Convertisseur de devises | À préciser (livrable tronqué) | B02-01 à B02-07 | ✅ Complète |
| US-C01 | Souvenirs & partage | 7 critères | C01-01 à C01-10 | ✅ Complète |
| SEC | Sécurité transversale | 6 critères | SEC-01 à SEC-06 | ✅ Complète |
| ERR | Cas d'erreur globaux | 8 critères | E-01 à E-10 | ✅ Complète |

---

## 9. STRATÉGIE DE TEST COMPLÈTE

### 9.1 Niveaux de test

| Niveau | Responsable | Outillage | Moment |
|---|---|---|---|
| **Tests unitaires** | Développeurs | Jest, React Testing Library | En continu (chaque PR) |
| **Tests d'intégration API** | Développeurs + QA | Supertest, Postman/Newman | À chaque fin de Bloc |
| **Tests end-to-end (E2E)** | QA | Detox (mobile), Playwright (web) | Semaine 12 |
| **Tests de régression** | QA | Suite E2E automatisée | Avant chaque release |
| **Tests de sécurité** | QA + Architecte | Tests manuels ciblés, OWASP checklist | Semaine 12 |
| **Tests de performance** | QA | k6 ou Artillery (API) | Semaine 12 |
| **Tests d'accessibilité** | QA + Frontend | axe-core, audit manuel | Semaine 11-12 |
| **Tests offline** | QA | Simulation réseau (Airplane mode + Charles Proxy) | Semaine 10-12 |

---

### 9.2 Environnements de test

| Environnement | Usage | Données |
|---|---|---|
| **Dev local** | Tests unitaires développeur | Données de mock |
| **Staging** | Tests d'intégration + E2E QA | Jeu de données de test isolé |
| **Production** | Smoke tests post-déploiement uniquement | Données réelles — lecture seule |

> ⚠️ **Règle absolue** : Aucun test destructif (création/suppression de compte, upload massif) ne s'exécute en production.

---

### 9.3 Jeu de données de test

Pour garantir la reproductibilité des tests, les fixtures suivantes doivent être disponibles en staging :

| Fixture | Description | Utilisée par |
|---|---|---|
| `user_alice` | Compte actif, voyage actif "Bali 2025", 3 étapes, 2 documents | A02, A03, B01, B02, C01 |
| `user_bob` | Compte actif, aucun voyage créé (état vide) | A01, onboarding |
| `user_carol` | Participant invité au voyage de `user_alice`, rôle viewer | A01 (participants), SEC-01 |
| `trip_past` | Voyage terminé il y a 30 jours, lié à `user_alice` | A01 (statut "passé") |
| `trip_active` | Voyage en cours (start_date = hier, end_date = dans 5 jours) | A02 (vue "Aujourd'hui") |
| `trip_singleday` | Voyage d'un seul jour (start_date = end_date) | ANO-M02 |
| `document_large` | Fichier PDF de 21 Mo (dépasse la limite) | A03-07 |
| `document_valid` | PDF de 5 Mo, catégorie "vol" | A03-01 à A03-05 |
| `album_public` | Album partagé avec lien valide, lié à `trip_active` | C01-04 à C01-06 |
| `album_deleted_trip` | Album partagé dont le voyage parent est supprimé | ANO-H05 |

---

### 9.4 Critères de sortie de la phase QA (Definition of Done — QA)

Avant toute mise en production, les critères suivants doivent être tous verts :

- [ ] **100%** des cas de test de niveau BLOQUANT exécutés et passants
- [ ] **100%** des cas de test de niveau CRITIQUE (sécurité) exécutés et passants
- [ ] **0** anomalie de sévérité CRITIQUE ou BLOQUANTE ouverte
- [ ] **< 3** anomalies de sévérité MAJEURE ouvertes (avec plan de résolution documenté)
- [ ] Les tests RLS Supabase sont exécutés et documentés
- [ ] Le parcours offline est validé sur device physique iOS ET Android
- [ ] La page publique `/share/[albumId]` est validée sans accès aux données privées
- [ ] Le smoke test de production est préparé et documenté

---

## 10. CAS DE TESTS COMPLÉMENTAIRES — BLOCS NON ENCORE DOCUMENTÉS

> Les sections précédentes (produites lors du premier tour) couvraient les modules A01, A02, A03, A04, B01, B02, C01 et la sécurité transversale. La présente section ajoute les cas de tests manquants pour les modules Authentification, Onboarding, Participants et Performance.

---

### 10.1 Module Authentification (US référence : Bloc 1 Tech Lead)

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| AUTH-01 | Inscription email valide | Utilisateur non inscrit, formulaire affiché | Saisie email valide + mot de passe ≥ 8 caractères, clic "S'inscrire" | Compte créé, redirection vers home, session active | Bloquant |
| AUTH-02 | Inscription email déjà utilisé | Email déjà enregistré en base | Tentative d'inscription avec le même email | Message d'erreur explicite "Email déjà utilisé", aucun doublon créé | Bloquant |
| AUTH-03 | Connexion credentials valides | Compte existant actif | Saisie email + mot de passe corrects | Session créée, JWT émis, redirection home | Bloquant |
| AUTH-04 | Connexion mot de passe incorrect | Compte existant | Saisie mot de passe erroné | Message d'erreur générique (ne pas révéler si l'email existe), aucun token émis | Bloquant |
| AUTH-05 | Session persistante au redémarrage | Utilisateur connecté, app fermée | Réouverture de l'app | Session restaurée, accès direct au home sans re-connexion | Bloquant |
| AUTH-06 | Refresh token silencieux | Access token expiré, refresh token valide | Navigation dans l'app après expiration | Nouveau access token émis silencieusement, utilisateur ne voit aucune interruption | Majeur |
| AUTH-07 | Refresh token expiré | Refresh token expiré | Tentative de navigation | Redirection vers l'écran de connexion, message "Session expirée" | Majeur |
| AUTH-08 | Connexion OAuth Google | Écran de connexion | Clic "Connexion avec Google", autorisation accordée | Session créée via OAuth, profil pré-rempli avec nom/avatar Google | Bloquant |
| AUTH-09 | Connexion Apple (iOS) | iPhone, écran de connexion | Clic "Connexion avec Apple", autorisation accordée | Session créée, email masqué Apple géré correctement | Bloquant |
| AUTH-10 | Déconnexion | Utilisateur connecté | Clic "Déconnexion" dans les paramètres | Session locale détruite, refresh token révoqué côté serveur, redirection onboarding | Majeur |
| AUTH-11 | Accès route protégée sans token | Utilisateur non authentifié | Appel direct à `GET /v1/trips` sans header Authorization | HTTP 401, aucune donnée retournée | Critique |
| AUTH-12 | Token JWT falsifié | Requête avec JWT modifié manuellement | Appel à n'importe quelle route protégée | HTTP 401, rejet immédiat | Critique |

---

### 10.2 Module Participants (US-A01 — critère participants)

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| PART-01 | Invitation email existant | Admin du voyage, email d'un utilisateur inscrit | Saisie email + envoi invitation | Invitation créée statut "pending", notification envoyée à l'invité | Majeur |
| PART-02 | Invitation email inconnu | Admin du voyage, email non inscrit | Saisie email + envoi invitation | Invitation créée, email envoyé avec lien d'inscription + accès au voyage | Majeur |
| PART-03 | Acceptation invitation | Utilisateur invité, notification reçue | Clic "Accepter" | Statut passe à "accepted", utilisateur accède au voyage en tant que viewer | Majeur |
| PART-04 | Refus invitation | Utilisateur invité | Clic "Refuser" | Statut passe à "declined", utilisateur n'apparaît plus dans la liste active | Mineur |
| PART-05 | Viewer ne peut pas modifier | Participant rôle "viewer" | Tentative de modification d'une étape | Action bloquée côté client (bouton grisé) ET côté serveur (HTTP 403) | Bloquant |
| PART-06 | Admin peut supprimer un participant | Admin du voyage | Suppression d'un participant depuis la liste | Participant retiré, plus d'accès au voyage | Majeur |
| PART-07 | Invitation email dupliqué | Email déjà invité au même voyage | Deuxième tentative d'invitation | Message "Cet email a déjà été invité", aucun doublon en base (contrainte UNIQUE) | Majeur |
| PART-08 | Isolation — viewer ne voit pas les voyages des autres | Utilisateur A et utilisateur B avec voyages distincts | Utilisateur A appelle `GET /v1/trips` | Seuls les voyages où A est créateur ou participant apparaissent | Critique |

---

### 10.3 Tests de performance — Seuils acceptables

| ID | Scénario | Condition | Seuil acceptable | Priorité |
|---|---|---|---|---|
| PERF-01 | Chargement itinéraire — voyage standard | Voyage avec 15 étapes, réseau 4G | Temps de réponse API < 500ms (p95) | Majeur |
| PERF-02 | Chargement itinéraire — voyage dense | Voyage avec 50 étapes, 20 documents | Temps de réponse API < 1000ms (p95) | Majeur |
| PERF-03 | Traduction connectée | Texte de 200 caractères, bonne connexion | Résultat affiché < 3 secondes (critère PO) | Bloquant |
| PERF-04 | Conversion devises | Appel avec taux en cache | Résultat affiché < 500ms | Majeur |
| PERF-05 | Upload document 10 Mo | Fichier PDF 10 Mo, réseau 4G | Upload complété < 30 secondes | Majeur |
| PERF-06 | Démarrage de l'app (cold start) | iPhone 12 / Android mid-range | App interactive < 3 secondes | Majeur |
| PERF-07 | Charge API concurrente | 50 utilisateurs simultanés sur `GET /v1/trips` | Aucune erreur 5xx, latence < 800ms (p99) | Majeur |

---

### 10.4 Tests d'accessibilité — Vérifications minimales MVP

| ID | Scénario | Critère vérifié | Priorité |
|---|---|---|---|
| ACC-01 | Contraste texte — écrans principaux | Ratio de contraste ≥ 4.5:1 (WCAG AA) sur home, itinéraire, outils | Majeur |
| ACC-02 | Navigation clavier (web) | Tous les éléments interactifs sont atteignables au clavier sans souris | Majeur |
| ACC-03 | Labels accessibles | Tous les champs de formulaire ont un label associé (non seulement placeholder) | Majeur |
| ACC-04 | Messages d'erreur accessibles | Les messages d'erreur sont annoncés aux lecteurs d'écran (aria-live) | Majeur |
| ACC-05 | Boutons avec texte alternatif | Les icônes cliquables sans libellé visible ont un aria-label | Mineur |

---

## 11. RAPPORT DE VALIDATION GLOBAL — SYNTHÈSE EXÉCUTIVE

> Ce rapport sera mis à jour à l'issue de la

— QA ENGINEER

## Suite et fin du livrable — Travel Companion App

---

## 11. RAPPORT DE VALIDATION GLOBAL — SYNTHÈSE EXÉCUTIVE

> Ce rapport sera mis à jour à l'issue de chaque cycle de test. La version ci-dessous constitue le **template de référence** à compléter par l'équipe QA lors de l'exécution.

### 11.1 Tableau de bord des critères d'acceptation PO

| Module | US | Critère d'acceptation | Statut | Cas de test associés | Observations |
|---|---|---|---|---|---|
| **A — Voyage** | US-A01 | Création avec destination, dates, participants | ⬜ À tester | A01-01 à A01-06, PART-01 à PART-08 | |
| | US-A01 | Apparition dans tableau de bord avec statut | ⬜ À tester | A01-03 | |
| | US-A01 | Créateur = admin automatique | ⬜ À tester | A01-04 | |
| | US-A02 | Vue chronologique par jour avec étapes | ⬜ À tester | A02-01 à A02-07 | |
| | US-A02 | Détail étape accessible | ⬜ À tester | A02-03, A02-04 | |
| | US-A02 | Accès offline en lecture seule | ⬜ À tester | A02-05, A02-06 | Nécessite test device physique |
| | US-A03 | Import PDF, JPG, PNG accepté | ⬜ À tester | A03-01, A03-02 | |
| | US-A03 | Association à une catégorie | ⬜ À tester | A03-03 | |
| | US-A03 | Accessible hors connexion | ⬜ À tester | A03-05 | |
| | US-A03 | Limite 20 Mo respectée | ⬜ À tester | A03-07 | |
| | US-A04 | Notification selon délai paramétrable | ⬜ À tester | A04-01 à A04-06 | |
| | US-A04 | Respect du fuseau horaire de l'étape | ⬜ À tester | A04-05 | |
| | US-A04 | Désactivation notifications respectée | ⬜ À tester | A04-03 | |
| **B — Outils** | US-B01 | Langue cible pré-sélectionnée depuis destination | ⬜ À tester | B01-01 | |
| | US-B01 | Traduction < 3 secondes en connecté | ⬜ À tester | B01-02, PERF-03 | Seuil PO contractuel |
| | US-B01 | Message informatif hors connexion | ⬜ À tester | B01-04 | |
| | US-B01 | Lecture audio disponible | ⬜ À tester | B01-03 | |
| | US-B02 | Devise cible pré-remplie | ⬜ À tester | B02-01 | |
| | US-B02 | Taux mis à jour toutes les heures | ⬜ À tester | B02-04 | |
| | US-B02 | Valeur négative ou nulle gérée | ⬜ À tester | B02-03 | |
| **C — Souvenirs** | US-C01 | Ajout photo/note/lieu | ⬜ À tester | C01-01 à C01-03 | |
| | US-C01 | Génération lien partage public | ⬜ À tester | C01-04 | |
| | US-C01 | Lien public sans accès données privées | ⬜ À tester | C01-05, SEC-04 | Critique sécurité |
| **Auth** | Bloc 1 | Inscription / connexion fonctionnelle | ⬜ À tester | AUTH-01 à AUTH-12 | |
| | Bloc 1 | Session persistante et refresh silencieux | ⬜ À tester | AUTH-05, AUTH-06 | |
| | Bloc 1 | OAuth Google + Apple | ⬜ À tester | AUTH-08, AUTH-09 | |
| **Sécurité** | Transverse | Isolation données entre utilisateurs | ⬜ À tester | SEC-01, PART-08 | RLS Supabase |
| | Transverse | Aucune clé API tierce exposée client | ⬜ À tester | SEC-05 | Revue de code obligatoire |
| **Perf** | Transverse | Temps de réponse API standards | ⬜ À tester | PERF-01 à PERF-07 | |
| **Accessibilité** | Transverse | Contraste et navigation clavier | ⬜ À tester | ACC-01 à ACC-05 | |

**Légende statut** : ⬜ À tester · ✅ Passant · ❌ Échoué · ⚠️ Partiel · 🔵 Bloqué (dépendance)

---

### 11.2 Récapitulatif volumétrique des tests

| Catégorie | Nombre de cas | Bloquant | Critique | Majeur | Mineur |
|---|---|---|---|---|---|
| Authentification | 12 | 6 | 2 | 4 | 0 |
| Module A — Voyage & Itinéraire | 30 | 12 | 0 | 14 | 4 |
| Module A — Documents | 10 | 4 | 0 | 5 | 1 |
| Module A — Notifications | 8 | 2 | 0 | 5 | 1 |
| Module B — Traducteur | 8 | 3 | 0 | 4 | 1 |
| Module B — Convertisseur | 7 | 2 | 0 | 4 | 1 |
| Module C — Souvenirs & Partage | 8 | 3 | 2 | 3 | 0 |
| Participants | 8 | 3 | 2 | 3 | 0 |
| Sécurité transversale | 8 | 0 | 8 | 0 | 0 |
| Performance | 7 | 1 | 0 | 6 | 0 |
| Accessibilité | 5 | 0 | 0 | 4 | 1 |
| Anomalies structurelles (ANO) | 10 | 4 | 0 | 5 | 1 |
| **TOTAL** | **121** | **40** | **14** | **57** | **10** |

---

## 12. REGISTRE DES ANOMALIES — TEMPLATE DE SUIVI

> Les anomalies ci-dessous sont des **anomalies identifiées par analyse statique des livrables** (avant exécution de tests). Elles représentent des risques détectés lors de la lecture croisée PO / Architecture / Tech Lead / Frontend / Backend.

### 12.1 Anomalies détectées par analyse des livrables

| ID | Titre | Description | Sévérité | Module impacté | Source détection | Statut |
|---|---|---|---|---|---|---|
| ANO-STATIC-01 | Incohérence stack : Express vs Next.js API Routes | Le Tech Lead préconise Node.js/Express, le Backend Engineer implémente Next.js API Routes. Le livrable Tech Lead (estimations, structure CI/CD) ne reflète pas la stack réelle. | Majeur | Architecture globale | Analyse croisée livrables | 🔴 Ouvert |
| ANO-STATIC-02 | Absence de stratégie de gestion d'erreur unifiée | Aucun livrable ne définit un format d'erreur API standardisé (code, message, details). Le Frontend devra gérer des formats hétérogènes. | Majeur | API / Frontend | Analyse Backend + Frontend | 🔴 Ouvert |
| ANO-STATIC-03 | Quota de stockage documents non défini | La limite est 20 Mo par fichier (US-A03), mais aucun quota global par utilisateur ou par voyage n'est spécifié dans l'architecture ou le backend. | Majeur | Documents / Backend | Analyse PO + Backend | 🔴 Ouvert |
| ANO-STATIC-04 | Gestion des timezones côté notifications — ambiguïté | US-A04 précise que les rappels tiennent compte du fuseau horaire de l'étape. Le schéma SQL stocke `destination_timezone` au niveau du voyage, pas de l'étape. Une étape à une destination intermédiaire différente serait mal gérée. | Majeur | Notifications / BDD | Analyse PO + Backend SQL | 🔴 Ouvert |
| ANO-STATIC-05 | Comportement du lien de partage si voyage supprimé — non spécifié | Le PO ne définit pas ce qui doit se passer si un album partagé est consulté après suppression du voyage parent. Le cas `album_deleted_trip` (fixture ANO-H05) ne dispose d'aucune spec de comportement attendu. | Majeur | Souvenirs / Partage | Analyse PO + Backend | 🔴 Ouvert |
| ANO-STATIC-06 | Pas de spécification de la durée de vie des liens de partage | Les liens d'album partagé (`/share/[albumId]`) semblent permanents. Aucune mention d'expiration, révocation ou contrôle d'accès post-partage dans les livrables. | Mineur | Souvenirs | Analyse PO + Frontend | 🔴 Ouvert |
| ANO-STATIC-07 | Rôle "editor" défini en base mais absent des US | Le schéma SQL définit trois rôles : `admin`, `editor`, `viewer`. Les User Stories ne mentionnent que admin et viewer. Le comportement de l'editor n'est pas spécifié. | Mineur | Participants / BDD | Analyse PO + Backend SQL | 🔴 Ouvert |
| ANO-STATIC-08 | Offline : périmètre exact non borné | Le PO indique "itinéraire en lecture seule" et "documents accessibles" hors connexion. Le livrable UX mentionne un cache offline sans préciser la profondeur (combien de voyages ? tous les documents ?) | Majeur | Offline / Architecture | Analyse UX + Architecture | 🔴 Ouvert |
| ANO-STATIC-09 | Pack de traduction hors ligne — non implémenté | L'US-B01 mentionne "propose un pack hors ligne téléchargeable" comme alternative offline. Aucun livrable technique ne couvre cette fonctionnalité. | Mineur | Traducteur | Analyse PO + Backend | 🔴 Ouvert |
| ANO-STATIC-10 | Validation format email côté backend non documentée | L'inscription accepte un email comme identifiant. Aucun livrable backend ne documente les règles de validation (format RFC, domaines bloqués, normalisation de casse). | Mineur | Auth | Analyse Backend | 🔴 Ouvert |

---

### 12.2 Template d'anomalie (à utiliser lors des tests d'exécution)

```
ID Anomalie    : ANO-EXEC-[numéro séquentiel]
Titre          : [Description courte]
Date détection : [JJ/MM/AAAA]
Détecté par    : [Nom du testeur]
Environnement  : [Staging / Production]
Device         : [iPhone 15 / Pixel 7 / Navigateur Chrome 120]

Sévérité       : [ ] Critique  [ ] Bloquant  [ ] Majeur  [ ] Mineur

Cas de test    : [ID du cas de test ayant déclenché l'anomalie]
US impactée    : [US-A01, etc.]

Description    :
[Description détaillée du comportement observé]

Étapes de reproduction :
1. [Étape 1]
2. [Étape 2]
3. [Étape 3]

Comportement attendu :
[Ce qui devrait se passer selon les critères d'acceptation PO]

Comportement observé :
[Ce qui se passe réellement]

Pièces jointes : [Screenshot / vidéo / logs]

Assigné à      : [Développeur responsable]
Statut         : [ ] Ouvert  [ ] En cours  [ ] Corrigé  [ ] Fermé  [ ] Won't fix
Date résolution: [JJ/MM/AAAA]
Version fix    : [numéro de build]
```

---

## 13. STRATÉGIE DE RÉGRESSION

### 13.1 Périmètre de régression par livraison

À chaque nouvelle livraison (fin de bloc Tech Lead), les tests de régression suivants sont systématiquement re-exécutés, indépendamment des modifications :

| Périmètre | Cas de test à re-exécuter | Fréquence |
|---|---|---|
| **Smoke test — chemins critiques** | AUTH-01, AUTH-03, A01-01, A02-01, A03-01, B01-02, C01-04 | Chaque build staging |
| **Régression sécurité** | SEC-01 à SEC-08, AUTH-11, AUTH-12, PART-05, PART-08 | Chaque livraison de bloc |
| **Régression offline** | A02-05, A02-06, A03-05 | Chaque livraison de bloc |
| **Régression performance** | PERF-01, PERF-03, PERF-06 | Chaque livraison de bloc |
| **Régression complète** | 121 cas complets | Avant release candidate |

### 13.2 Règle de blocage de livraison

> **Une livraison est bloquée si :**
> - 1 ou plusieurs anomalies CRITIQUE ouvertes
> - 1 ou plusieurs anomalies BLOQUANTE ouvertes sur les chemins critiques (auth, création voyage, consultation itinéraire)
> - Le smoke test échoue sur plus de 2 cas

---

## 14. PLAN D'EXÉCUTION DES TESTS — PHASAGE

### Phase 1 — Tests d'intégration par bloc (semaines 2 à 9)

Alignés sur le plan d'exécution Tech Lead, les tests QA interviennent **en fin de chaque bloc** :

| Semaine | Bloc terminé | Tests QA à exécuter | Durée estimée QA |
|---|---|---|---|
| S2 | Bloc 1 — Auth | AUTH-01 à AUTH-12 + SEC transverse auth | 1 jour |
| S3-S4 | Bloc 2 — Voyage & Itinéraire | A01-01 à A02-07, PART-01 à PART-08 | 2 jours |
| S5 | Bloc 3 — Documents | A03-01 à A03-10 | 1 jour |
| S6 | Bloc 4 — Outils utilitaires | B01-01 à B02-07 | 1 jour |
| S7-S8 | Bloc 5 — Souvenirs & Partage | C01-01 à C01-

— QA ENGINEER

*(Suite et fin du livrable — reprise après "C01-01 à C01-")*

---

| S7-S8 | Bloc 5 — Souvenirs & Partage | C01-01 à C01-08 | 1 jour |
| S9 | Bloc 6 — Notifications & Rappels | A04-01 à A04-08 | 1 jour |
| S10 | Bloc 7 — Offline & Synchronisation | A02-05, A02-06, A03-05, OFF-01 à OFF-06 | 1,5 jours |
| S11 | Bloc 8 — UX Polish | ACC-01 à ACC-05, états vides, feedback UI | 0,5 jour |
| S12 | Bloc 9 — QA final & Stores | Régression complète 121 cas + PERF + SEC | 3 jours |

---

### Phase 2 — Campagne de régression finale (semaine 12)

Exécution complète des 121 cas de tests sur environnement **staging gelé** (code freeze).

**Critères d'entrée en phase 2 :**
- Tous les blocs 0 à 8 déclarés terminés par le Tech Lead
- Zéro anomalie CRITIQUE ou BLOQUANTE ouverte sur les modules cœur
- L'environnement staging est stabilisé (aucun déploiement pendant la campagne)

**Critères de sortie (Definition of Done QA) :**
- 100% des cas Must Have exécutés
- 0 anomalie Critique ouverte
- 0 anomalie Bloquante sur chemins critiques (Auth, Voyage, Itinéraire)
- ≤ 3 anomalies Majeures ouvertes avec plan de correction documenté
- Rapport de validation final signé

---

## 15. PLAN DE TEST OFFLINE — DÉTAIL SPÉCIFIQUE

> Le comportement offline est une **feature différenciante** identifiée par le PO et l'UX Designer. Il mérite un plan de test dédié.

### 15.1 Matrice de couverture offline

| Fonctionnalité | Connecté | Offline attendu | Offline à tester |
|---|---|---|---|
| Consultation itinéraire | ✅ Lecture + édition | ✅ Lecture seule (cache) | OFF-01 |
| Consultation documents | ✅ Lecture + upload | ✅ Lecture seule (fichiers cachés) | OFF-02 |
| Import document | ✅ Upload immédiat | ⏳ File d'attente sync | OFF-03 |
| Traducteur | ✅ Résultat API | ❌ Message informatif clair | OFF-04 |
| Convertisseur devises | ✅ Taux temps réel | ⚠️ Dernier taux connu (horodaté) | OFF-05 |
| Création étape itinéraire | ✅ Sauvegarde immédiate | ⏳ File d'attente sync | OFF-06 |
| Notifications | ✅ Push temps réel | ✅ Planifiées localement | OFF-07 |
| Partage album souvenirs | ✅ Génération lien | ❌ Non disponible | OFF-08 |

### 15.2 Cas de tests Offline

| ID | Scénario | Given | When | Then | Sévérité si échoue |
|---|---|---|---|---|---|
| OFF-01 | Accès itinéraire sans réseau | App précédemment chargée avec voyage actif, réseau coupé | L'utilisateur ouvre la vue Itinéraire | Les données du dernier chargement s'affichent, badge "Hors connexion" visible, boutons d'édition désactivés | Bloquant |
| OFF-02 | Consultation document hors connexion | Document déjà consulté ou téléchargé, réseau coupé | L'utilisateur ouvre le document | Le document s'ouvre depuis le cache local sans erreur | Bloquant |
| OFF-03 | Tentative d'import hors connexion | Réseau coupé | L'utilisateur tente d'importer un PDF | Message "Upload en attente de connexion", document mis en file, upload déclenché à la reconnexion | Majeur |
| OFF-04 | Traducteur sans réseau | Réseau coupé | L'utilisateur ouvre le traducteur et saisit du texte | Message clair "Traduction nécessite une connexion Internet", pas de spinner infini | Majeur |
| OFF-05 | Convertisseur avec taux en cache | Taux chargés lors de la dernière session connectée, réseau coupé | L'utilisateur utilise le convertisseur | Le dernier taux connu est utilisé avec mention explicite "Taux du [date/heure]" | Majeur |
| OFF-06 | Création étape offline | Réseau coupé, voyage ouvert | L'utilisateur crée une nouvelle étape | Étape sauvegardée localement, indicateur de sync pending visible, synchronisée à la reconnexion | Majeur |
| OFF-07 | Reconnexion après actions offline | 3 actions effectuées hors connexion (étape créée, document uploadé en attente) | Réseau rétabli | Toutes les actions en file d'attente se synchronisent automatiquement, indicateur de progression visible | Bloquant |
| OFF-08 | App jamais ouverte en mode connecté | Première ouverture de l'app, réseau coupé | L'utilisateur ouvre l'app | État vide informatif, pas de crash, invitation à se connecter pour accéder au contenu | Critique |

---

## 16. PLAN DE TEST SÉCURITÉ — DÉTAIL SPÉCIFIQUE

> Les tests de sécurité sont **non négociables avant toute mise en production**. Ils ne peuvent être exécutés que par une personne ayant accès aux outils de test (Postman, proxy réseau, compte test multiple).

### 16.1 Cas de tests sécurité détaillés

| ID | Scénario | Given | When | Then | Sévérité |
|---|---|---|---|---|---|
| SEC-01 | Isolation des voyages entre utilisateurs | Deux comptes distincts : UserA (propriétaire voyage Trip-A), UserB | UserB tente d'appeler `GET /api/trips/[tripId-de-A]` avec son propre token JWT valide | HTTP 403 Forbidden ou 404, aucune donnée du voyage de A retournée | Critique |
| SEC-02 | Accès document d'un autre utilisateur | UserA possède Document-A dans son voyage | UserB appelle `GET /api/documents/[docId-de-A]` | HTTP 403 ou 404, document non retourné | Critique |
| SEC-03 | Manipulation de rôle participant | UserB est "viewer" dans Trip-A | UserB envoie `PATCH /api/trips/[tripId]/steps/[stepId]` avec payload de modification | HTTP 403, modification refusée | Critique |
| SEC-04 | Accès album partagé — données privées | Album partagé public `/share/[albumId]` créé depuis Trip-A | Visiteur anonyme (sans token) consulte la page | Seules les données de l'album (médias, titre) sont exposées — aucune donnée privée du voyage (itinéraire, documents, participants) | Critique |
| SEC-05 | Vérification exposition clés API tierces | App en production | Inspection réseau depuis le client mobile (Charles Proxy / mitmproxy) | Aucune requête directe vers DeepL, Fixer.io ou Timezone API depuis le client — toutes les requêtes passent par `/api/` | Critique |
| SEC-06 | Injection SQL via champs texte | Formulaire création voyage (champ "destination") | Saisie de `'; DROP TABLE trips; --` dans le champ | Valeur traitée comme texte ordinaire, aucune erreur SQL, aucune donnée supprimée | Critique |
| SEC-07 | Upload de fichier malveillant | Module import documents | Upload d'un fichier `.exe` renommé en `.pdf`, puis d'un fichier SVG avec script XSS embarqué | Fichier refusé avec message d'erreur explicite — ou, si accepté, neutralisé sans exécution de code | Critique |
| SEC-08 | Token JWT expiré accepté | Token JWT dont la date d'expiration est dépassée | Appel à `GET /api/trips` avec token expiré (modifié manuellement) | HTTP 401 Unauthorized, accès refusé | Critique |
| SEC-09 | Énumération d'IDs | UUIDs utilisés pour tous les identifiants | Tentative d'itération séquentielle sur `/api/trips/0001`, `/api/trips/0002`... | UUIDs non prédictibles — les tentatives retournent 403/404 sans révéler l'existence des ressources | Majeur |
| SEC-10 | Rate limiting sur auth | Endpoint `POST /api/auth/login` | 20 tentatives de connexion en moins d'une minute avec credentials incorrects | HTTP 429 Too Many Requests après le seuil défini, tentatives bloquées temporairement | Majeur |

---

## 17. ENVIRONNEMENTS ET DONNÉES DE TEST

### 17.1 Comptes de test à provisionner

| Compte | Rôle | Usage |
|---|---|---|
| `qa-admin@test.com` | Admin voyage | Tests création, gestion, invitations |
| `qa-viewer@test.com` | Viewer invité | Tests restrictions de rôle |
| `qa-newuser@test.com` | Nouveau compte vierge | Tests onboarding, état vide |
| `qa-security@test.com` | Compte attaquant (tests SEC) | Tests d'isolation et accès non autorisé |
| `qa-offlineuser@test.com` | Compte avec voyage pré-chargé | Tests offline |

### 17.2 Jeux de données de test

| Dataset | Description | Utilisé par |
|---|---|---|
| `trip-upcoming` | Voyage à venir dans 7 jours, 3 étapes, 2 documents | A01, A02, A04 |
| `trip-active` | Voyage en cours aujourd'hui, étapes aujourd'hui et demain | A02, B01, B02, Offline |
| `trip-past` | Voyage terminé il y a 30 jours, avec souvenirs | A01, C01 |
| `trip-multi-participant` | Voyage avec 3 participants (admin + editor + viewer) | PART-01 à PART-08, SEC-03 |
| `document-all-types` | 1 PDF vol, 1 JPG hébergement, 1 PNG activité, 1 PDF autre | A03 |
| `document-oversized` | Fichier PDF de 21 Mo | A03-07 |
| `album-shared-public` | Album partagé avec lien actif | C01, SEC-04 |
| `album-deleted-trip` | Album dont le voyage parent est archivé | ANO-STATIC-05 |

### 17.3 Devices et environnements cibles

| Type | Device | OS | Priorité |
|---|---|---|---|
| iOS physique | iPhone 14 ou 15 | iOS 17 | P1 |
| Android physique | Google Pixel 7 ou Samsung Galaxy S23 | Android 13/14 | P1 |
| iOS simulateur | Xcode Simulator iPhone 15 | iOS 17 | P2 |
| Android émulateur | Android Studio AVD | Android 13 | P2 |
| Web (companion) | Chrome 120+ | macOS / Windows | P1 |
| Web | Safari 17 | macOS / iOS | P2 |

---

## 18. CRITÈRES DE QUALITÉ — DÉFINITION OF DONE QA

### 18.1 Definition of Done — Feature

Une User Story est considérée **Done** du point de vue QA si et seulement si :

- [ ] Tous les cas de tests associés (Must Have) sont exécutés
- [ ] 0 anomalie Critique ou Bloquante ouverte sur cette US
- [ ] Les critères d'acceptation PO sont tous couverts par au moins un cas de test passant
- [ ] Les cas limites identifiés sont exécutés (même si non bloquants)
- [ ] Le comportement offline est validé si la US est concernée
- [ ] Les tests de sécurité transversaux passent pour cette surface

### 18.2 Definition of Done — Release Candidate

| Critère | Seuil | Statut |
|---|---|---|
| Cas de tests Must Have exécutés | 100% | ⬜ |
| Cas de tests Should Have exécutés | ≥ 80% | ⬜ |
| Anomalies Critiques ouvertes | 0 | ⬜ |
| Anomalies Bloquantes ouvertes (chemins critiques) | 0 | ⬜ |
| Anomalies Majeures ouvertes | ≤ 3 avec plan de fix documenté | ⬜ |
| Tests sécurité SEC-01 à SEC-08 | 100% passants | ⬜ |
| Tests performance PERF-01, PERF-03, PERF-06 | 100% dans les seuils | ⬜ |
| Tests offline OFF-01, OFF-02, OFF-07 | 100% passants | ⬜ |
| Smoke test complet | 7/7 passants | ⬜ |
| Rapport de validation signé QA | Oui | ⬜ |

---

## 19. RAPPORT DE VALIDATION FINAL — TEMPLATE

```
═══════════════════════════════════════════════════════════
RAPPORT DE VALIDATION QA — TRAVEL COMPANION APP
Version : [X.Y.Z]       Build : [numéro]       Date : [JJ/MM/AAAA]
Environnement : Staging  Campagne : Régression finale S12
═══════════════════════════════════════════════════════════

RÉSUMÉ EXÉCUTIF
───────────────────────────────────────────────────────────
Total cas exécutés    : [N] / 121
Passants (✅)         : [N]
Échoués (❌)          : [N]
Partiels (⚠️)         : [N]
Bloqués (🔵)          : [N]

Taux de succès        : [%]
Anomalies ouvertes    : Critiques [N] | Bloquantes [N] | Majeures [N] | Mineures [N]

DÉCISION DE LIVRAISON : [ ] GO  [ ] NO-GO  [ ] GO conditionnel

Conditions (si GO conditionnel) :
- [Condition 1]
- [Condition 2]

DÉTAIL PAR MODULE
───────────────────────────────────────────────────────────
Module          | Exécutés | ✅ | ❌ | ⚠️ | Anomalies ouvertes
────────────────|──────────|────|────|────|──────────────────
Authentification|          |    |    |    |
Module A Voyage |          |    |    |    |
Module A Docs   |          |    |    |    |
Module A Notifs |          |    |    |    |
Module B Outils |          |    |    |    |
Module C Souvs  |          |    |    |    |
Participants    |          |    |

— QA ENGINEER

*(Suite et fin du livrable — reprise après la troncature du Rapport de Validation)*

---

```
DÉTAIL PAR MODULE (suite)
───────────────────────────────────────────────────────────
Module          | Exécutés | ✅ | ❌ | ⚠️ | Anomalies ouvertes
────────────────|──────────|────|────|────|──────────────────
Authentification|          |    |    |    |
Module A Voyage |          |    |    |    |
Module A Docs   |          |    |    |    |
Module A Notifs |          |    |    |    |
Module B Outils |          |    |    |    |
Module C Souvs  |          |    |    |    |
Participants    |          |    |    |    |
Offline         |          |    |    |    |
Sécurité        |          |    |    |    |
Performance     |          |    |    |    |
────────────────|──────────|────|────|────|──────────────────
TOTAL           |          |    |    |    |

ANOMALIES BLOQUANTES RESTANTES
───────────────────────────────────────────────────────────
ID       | Module    | Description courte          | Statut fix
─────────|───────────|─────────────────────────────|──────────
         |           |                             |
         |           |                             |

RISQUES ET OBSERVATIONS
───────────────────────────────────────────────────────────
[Zone de texte libre : alertes, régressions détectées,
 zones de fragilité identifiées, recommandations]

VALIDATION
───────────────────────────────────────────────────────────
QA Engineer        : ___________________   Date : ___/___/______
Tech Lead          : ___________________   Date : ___/___/______
Product Owner      : ___________________   Date : ___/___/______

═══════════════════════════════════════════════════════════
```

---

## 20. REGISTRE COMPLET DES ANOMALIES — TEMPLATE

> Ce registre est tenu à jour tout au long de la campagne. Chaque anomalie détectée y est consignée avec sa sévérité, son statut et son affectation.

### Échelle de sévérité

| Niveau | Définition | Délai de résolution attendu |
|---|---|---|
| **Critique** | Perte de données, faille de sécurité, crash général, accès non autorisé | Immédiat — bloque la livraison |
| **Bloquant** | Fonctionnalité Must Have inopérante, chemin utilisateur impossible | Avant release |
| **Majeur** | Fonctionnalité dégradée, contournement possible mais expérience altérée | Avant release ou plan documenté |
| **Mineur** | Problème cosmétique, wording, comportement légèrement inattendu | Sprint suivant acceptable |

### Template de fiche anomalie

```
────────────────────────────────────────────────────────────
ANO-[MODULE]-[NNN]
────────────────────────────────────────────────────────────
Titre         : [Description courte et précise]
Sévérité      : [ ] Critique  [ ] Bloquant  [ ] Majeur  [ ] Mineur
Module        : [Authentification / Voyage / Documents / Outils /
                 Souvenirs / Participants / Offline / Sécurité]
US liée       : [US-XXX]
Cas de test   : [ID du cas ayant détecté l'anomalie]
Statut        : [ ] Ouvert  [ ] En cours  [ ] Résolu  [ ] Fermé
Assigné à     : [Développeur / équipe]

ENVIRONNEMENT
Device        : [iPhone 15 / Pixel 7 / Chrome 120 / ...]
OS            : [iOS 17 / Android 14 / macOS 14 / ...]
Build         : [numéro de build]
Environnement : [Dev / Staging / Production]

REPRODUCTION
Préconditions : [État du système avant la reproduction]
Étapes        :
  1. [Étape 1]
  2. [Étape 2]
  3. [Étape 3]
Fréquence     : [ ] Systématique  [ ] Intermittent (X/N fois)

COMPORTEMENT OBSERVÉ
[Description précise de ce qui se passe]

COMPORTEMENT ATTENDU
[Description de ce qui devrait se passer — référence au critère d'acceptation PO]

CAPTURES / LOGS
[Liens vers screenshots, vidéo de reproduction, logs console]

HISTORIQUE
[JJ/MM/AAAA] Ouvert par QA
[JJ/MM/AAAA] Assigné à [dev]
[JJ/MM/AAAA] Fix soumis — PR #[numéro]
[JJ/MM/AAAA] Validé en re-test par QA → Fermé
────────────────────────────────────────────────────────────
```

---

## 21. ANOMALIES STATIQUES IDENTIFIÉES À LA LECTURE DES LIVRABLES

> Ces anomalies sont détectées **avant tout test d'exécution**, par analyse croisée des livrables PO, UX, Architecture, Tech Lead, Frontend et Backend. Elles doivent être levées ou arbitrées avant le début de la campagne de test.

| ID | Source de détection | Description | Modules impactés | Sévérité | Action requise |
|---|---|---|---|---|---|
| ANO-STATIC-01 | Livrable PO tronqué (US-B02 incomplète) | La User Story US-B02 (convertisseur de devises) est tronquée dans les livrables — les critères d'acceptation sont partiels. Les cas de tests B02 sont construits sur hypothèse. | Module B — Outils | Majeur | PO doit fournir les critères d'acceptation complets de US-B02 avant la campagne de test |
| ANO-STATIC-02 | Divergence Stack Architecture vs Backend | L'Architecte préconise Node.js/Express + Redis ; le Backend Engineer implémente Next.js API Routes + Supabase. La stratégie de cache (Redis → Supabase Edge Cache + in-memory) n'est pas équivalente pour les taux de change et les sessions. | Backend — Outils Service, Auth | Majeur | Tech Lead doit arbitrer et documenter la décision — vérifier que le comportement offline du convertisseur reste identique |
| ANO-STATIC-03 | Absence de US dédiée à la gestion des participants | Le PO mentionne les participants dans US-A01 (création voyage) mais aucune US dédiée ne couvre l'invitation, l'acceptation/refus, la gestion des rôles (admin/editor/viewer). Ces flux existent dans le schéma SQL Backend mais sont orphelins côté PO. | Module A — Participants | Majeur | PO doit créer les US manquantes ou confirmer que ces flux sont hors MVP |
| ANO-STATIC-04 | Comportement offline traducteur non arbitré | La US-B01 propose deux comportements mutuellement exclusifs hors connexion : "message d'information" OU "pack hors ligne téléchargeable". Le livrable UX ne tranche pas. Le Backend ne mentionne pas de mécanisme de téléchargement de packs. | Module B — Traducteur, Offline | Majeur | PO + Tech Lead doivent trancher le comportement offline du traducteur pour le MVP |
| ANO-STATIC-05 | Durée de vie des liens de partage d'album non définie | Le module Souvenirs génère des liens de partage (`/share/[albumId]`) mais aucun livrable ne définit la durée de validité, la révocation, ni le comportement si le voyage parent est supprimé/archivé. | Module C — Souvenirs, Sécurité | Majeur | PO + Architecte doivent définir la politique de partage (durée, révocation, suppression) |
| ANO-STATIC-06 | Fuseaux horaires dans les notifications — implémentation non spécifiée | US-A04 stipule que les rappels "tiennent compte du fuseau horaire de l'étape". Le Backend ne décrit pas comment le Notification Service récupère et applique ce fuseau au moment de la planification des jobs. | Module A — Notifications | Majeur | Backend Engineer doit documenter l'implémentation du calcul de fuseau horaire pour la planification des rappels |
| ANO-STATIC-07 | Quota de stockage documents non implémenté | US-A03 définit une taille maximale de 20 Mo **par fichier** mais aucun livrable ne définit de quota global par utilisateur ou par voyage. Le schéma SQL Backend ne comporte pas de champ quota. | Module A — Documents | Mineur | PO doit trancher sur l'existence ou non d'un quota global pour le MVP |
| ANO-STATIC-08 | Gestion de la "destination active" lors d'un voyage multi-destinations | Le principe UX P1 ("le voyage est le contexte permanent") et les outils (traducteur, convertisseur) s'appuient sur la destination du voyage actif. Le schéma SQL prévoit un champ `destination` unique par voyage. Les voyages multi-destinations (étapes dans plusieurs pays) ne sont pas gérés. | Module B — Outils, UX | Mineur | PO doit clarifier si les voyages multi-destinations sont dans le MVP et, si oui, quelle destination est "active" pour les outils |

---

## 22. MATRICE DE COUVERTURE — USER STORIES × CAS DE TESTS

> Cette matrice garantit qu'aucune User Story n'est orpheline de test et qu'aucun critère d'acceptation PO n'est oublié.

| User Story | Critères d'acceptation PO | Cas de tests couvrants | Statut couverture |
|---|---|---|---|
| **US-A01** — Création voyage | 3 critères | AUTH-03, TRIP-01 à TRIP-07 | ✅ Complet |
| **US-A02** — Itinéraire | 3 critères | ITIN-01 à ITIN-06, OFF-01, OFF-02 | ✅ Complet |
| **US-A03** — Import documents | 4 critères | DOC-01 à DOC-10 | ✅ Complet |
| **US-A04** — Notifications | 3 critères | NOTIF-01 à NOTIF-08 | ✅ Complet |
| **US-B01** — Traducteur | 4 critères | TOOL-01 à TOOL-06 | ⚠️ Partiel — comportement offline non arbitré (ANO-STATIC-04) |
| **US-B02** — Convertisseur devises | Critères incomplets (livrable tronqué) | TOOL-07 à TOOL-12 | ⚠️ Partiel — critères d'acceptation incomplets (ANO-STATIC-01) |
| **US-B03** — Fuseaux horaires | Non visible (livrable tronqué) | TOOL-13 à TOOL-16 | ⚠️ Partiel — US non consultable |
| **US-C01** — Souvenirs & partage | Non visible (livrable tronqué) | MEM-01 à MEM-12 | ⚠️ Partiel — US non consultable |
| **US Participants** | Absente du livrable PO | PART-01 à PART-08 | ❌ US manquante (ANO-STATIC-03) |
| **Onboarding** | Défini via parcours UX | AUTH-01 à AUTH-06 | ✅ Complet |

---

## 23. STRATÉGIE DE RÉGRESSION

### 23.1 Suite de régression minimale (à exécuter à chaque build)

> Ces 25 cas constituent le **smoke test + régression core** à exécuter en moins de 45 minutes avant toute validation de build staging.

| # | ID | Description | Durée estimée |
|---|---|---|---|
| 1 | AUTH-01 | Inscription email/password | 3 min |
| 2 | AUTH-03 | Connexion email/password | 2 min |
| 3 | TRIP-01 | Création voyage complet | 3 min |
| 4 | TRIP-03 | Affichage tableau de bord avec voyages | 2 min |
| 5 | TRIP-04 | Statut voyage calculé automatiquement | 2 min |
| 6 | ITIN-01 | Affichage itinéraire jour par jour | 2 min |
| 7 | ITIN-02 | Détail d'une étape | 2 min |
| 8 | DOC-01 | Import PDF | 3 min |
| 9 | DOC-04 | Catégorisation automatique | 2 min |
| 10 | DOC-07 | Taille fichier > 20 Mo refusée | 2 min |
| 11 | TOOL-01 | Langue cible pré-remplie traducteur | 2 min |
| 12 | TOOL-02 | Traduction texte en mode connecté | 2 min |
| 13 | TOOL-07 | Devise cible pré-remplie convertisseur | 2 min |
| 14 | TOOL-08 | Conversion devise en temps réel | 2 min |
| 15 | MEM-01 | Ajout souvenir photo | 3 min |
| 16 | MEM-07 | Génération lien partage | 2 min |
| 17 | MEM-08 | Accès album public sans connexion | 2 min |
| 18 | PART-01 | Invitation participant par email | 3 min |
| 19 | PART-05 | Restriction viewer (lecture seule) | 3 min |
| 20 | OFF-01 | Consultation itinéraire hors connexion | 3 min |
| 21 | OFF-03 | Documents accessibles hors connexion | 3 min |
| 22 | SEC-01 | Isolation voyages entre utilisateurs | 3 min |
| 23 | SEC-05 | Aucun appel API tierce depuis le client | 3 min |
| 24 | NOTIF-01 | Envoi notification 24h avant étape | 3 min |
| 25 | AUTH-07 | Token expiré → refresh silencieux | 2 min |
| | | **Total estimé** | **~62 min** |

### 23.2 Déclencheurs de régression complète

Une régression complète (121 cas) est obligatoire dans les cas suivants :
- Modification du schéma de base de données
- Changement d'un fournisseur d'API tierce
- Modification de la logique d'authentification ou des tokens
- Modification de la stratégie de cache offline
- Toute PR touchant les modules Auth, Voyage Service, ou Offline Module
- Avant chaque Release Candidate formelle

---

## 24. INDICATEURS DE SUIVI QA

> Ces métriques sont produites à chaque fin de sprint et intégrées au rapport hebdomadaire.

| Indicateur | Formule | Cible MVP |
|---|---|---|
| **Taux de couverture US** | US avec ≥1 cas de test passant / Total US | 100% |
| **Taux d'exécution** | Cas exécutés

— QA ENGINEER

*(Continuation exacte à partir du dernier mot produit — "Cas exécutés")*

---

| **Taux d'exécution** | Cas exécutés / Total cas planifiés | ≥ 95% |
| **Taux de succès** | Cas passants / Cas exécutés | ≥ 90% avant release |
| **Densité d'anomalies** | Anomalies trouvées / Cas exécutés | Référence à établir au Sprint 1 |
| **Taux de régression** | Anomalies régressées / Total anomalies fermées | 0% toléré en P1/P2 |
| **MTTR QA** | Délai moyen ouverture anomalie → fermeture validée | ≤ 5 jours ouvrés (P1/P2) |
| **Blockers open en fin de sprint** | Nombre d'anomalies P1 ouvertes au day de la démo | 0 toléré |

---

## 25. RAPPORT DE VALIDATION PAR CRITÈRE D'ACCEPTATION

> Ce tableau constitue le **livrable de validation formel** à présenter au PO à chaque fin de sprint. Statut initial : tous les critères sont en attente d'exécution (🔲 Non testé).

### MODULE A — Gestion du voyage et itinéraire

| ID Critère | User Story | Critère d'acceptation (résumé) | Cas de test(s) | Statut | Observations |
|---|---|---|---|---|---|
| CA-A01-1 | US-A01 | Création voyage : saisie nom, destination, dates, participants | TRIP-01, TRIP-02 | 🔲 Non testé | |
| CA-A01-2 | US-A01 | Voyage affiché en dashboard avec statut | TRIP-03, TRIP-04 | 🔲 Non testé | |
| CA-A01-3 | US-A01 | Créateur = administrateur automatiquement | TRIP-05 | 🔲 Non testé | |
| CA-A02-1 | US-A02 | Vue chronologique par jour avec étapes, horaires, lieux | ITIN-01 | 🔲 Non testé | |
| CA-A02-2 | US-A02 | Détail d'une étape accessible | ITIN-02, ITIN-03 | 🔲 Non testé | |
| CA-A02-3 | US-A02 | Accès itinéraire hors connexion en lecture seule | OFF-01, OFF-02 | 🔲 Non testé | |
| CA-A03-1 | US-A03 | Formats acceptés : PDF, JPG, PNG uniquement | DOC-01, DOC-02, DOC-08 | 🔲 Non testé | |
| CA-A03-2 | US-A03 | Document associé à une catégorie | DOC-04, DOC-05 | 🔲 Non testé | |
| CA-A03-3 | US-A03 | Document accessible hors connexion | OFF-03 | 🔲 Non testé | |
| CA-A03-4 | US-A03 | Taille maximale 20 Mo par fichier | DOC-07, DOC-09 | 🔲 Non testé | |
| CA-A04-1 | US-A04 | Notification selon délai paramétrable (24h, 2h, custom) | NOTIF-01, NOTIF-02, NOTIF-03 | 🔲 Non testé | |
| CA-A04-2 | US-A04 | Aucune notification si désactivé dans paramètres | NOTIF-06 | 🔲 Non testé | |
| CA-A04-3 | US-A04 | Rappels en tenant compte du fuseau horaire de l'étape | NOTIF-07, NOTIF-08 | 🔲 Non testé | ⚠️ Implémentation non documentée (ANO-STATIC-06) |

### MODULE B — Outils utilitaires

| ID Critère | User Story | Critère d'acceptation (résumé) | Cas de test(s) | Statut | Observations |
|---|---|---|---|---|---|
| CA-B01-1 | US-B01 | Langue cible pré-sélectionnée selon destination | TOOL-01 | 🔲 Non testé | |
| CA-B01-2 | US-B01 | Traduction affichée < 3 secondes en connecté | TOOL-02, TOOL-03 | 🔲 Non testé | |
| CA-B01-3 | US-B01 | Message ou pack hors ligne si déconnecté | TOOL-04 | 🔲 Non testé | ⚠️ Comportement non arbitré (ANO-STATIC-04) |
| CA-B01-4 | US-B01 | Lecture audio de la traduction | TOOL-05, TOOL-06 | 🔲 Non testé | |
| CA-B02-1 | US-B02 | Devise cible pré-remplie selon destination | TOOL-07 | 🔲 Non testé | ⚠️ Critères incomplets (ANO-STATIC-01) |
| CA-B02-2 | US-B02 | Conversion en temps réel | TOOL-08 | 🔲 Non testé | ⚠️ Critères incomplets (ANO-STATIC-01) |
| CA-B02-3 | US-B02 | Critères manquants — à compléter par PO | TOOL-09 à TOOL-12 | ❌ Bloqué | Attente US-B02 complète |
| CA-B03-x | US-B03 | Non visible — livrable tronqué | TOOL-13 à TOOL-16 | ❌ Bloqué | Attente livrable PO complet |

### MODULE C — Souvenirs & Partage

| ID Critère | User Story | Critère d'acceptation (résumé) | Cas de test(s) | Statut | Observations |
|---|---|---|---|---|---|
| CA-C01-x | US-C01 | Non visible — livrable tronqué | MEM-01 à MEM-12 | ❌ Bloqué | Attente livrable PO complet |

### TRANSVERSES — Sécurité, Offline, Auth

| ID Critère | Domaine | Critère d'acceptation (résumé) | Cas de test(s) | Statut | Observations |
|---|---|---|---|---|---|
| CA-SEC-01 | Sécurité | Isolation données entre utilisateurs | SEC-01 | 🔲 Non testé | |
| CA-SEC-02 | Sécurité | Aucune clé API tierce exposée côté client | SEC-05 | 🔲 Non testé | |
| CA-OFF-01 | Offline | Données consultables hors connexion | OFF-01 à OFF-05 | 🔲 Non testé | |
| CA-AUTH-01 | Auth | Token expiré → refresh silencieux | AUTH-07 | 🔲 Non testé | |
| CA-AUTH-02 | Auth | Route protégée inaccessible sans auth | AUTH-06 | 🔲 Non testé | |

**Légende statuts :**
- 🔲 Non testé
- ✅ Passant
- ❌ Échouant — anomalie ouverte
- ⏭️ Ignoré (skip justifié)
- 🚫 Bloqué (dépendance non levée)

---

## 26. CATALOGUE COMPLET DES CAS DE TESTS

> Chaque cas de test est autonome : il peut être exécuté indépendamment avec les préconditions précisées.

---

### SECTION AUTH — Authentification & Session

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| AUTH-01 | Inscription email/password valide | Je suis sur l'écran d'inscription, aucun compte existant avec cet email | Je saisis un email valide + password ≥ 8 caractères et valide | Mon compte est créé, je suis redirigé vers le tableau de bord, un voyage vide m'est proposé | P1 |
| AUTH-02 | Inscription avec email déjà existant | Un compte existe avec l'email `test@example.com` | Je tente de m'inscrire avec le même email | Un message d'erreur explicite s'affiche ("email déjà utilisé"), aucun doublon créé en base | P1 |
| AUTH-03 | Connexion email/password correct | Un compte existe avec `user@test.com` / `Password123!` | Je saisis les identifiants corrects et valide | Je suis connecté, redirigé vers le tableau de bord, ma session est persistée | P1 |
| AUTH-04 | Connexion avec mot de passe incorrect | Un compte existe | Je saisis l'email correct mais un mauvais mot de passe | Message d'erreur générique affiché ("identifiants incorrects"), aucune info de sécurité divulguée | P1 |
| AUTH-05 | Connexion Google OAuth | Je suis sur l'écran de connexion | Je tape "Continuer avec Google" et j'autorise | Je suis connecté avec mon compte Google, profil créé ou récupéré, redirigé vers home | P1 |
| AUTH-06 | Accès route protégée sans connexion | Je ne suis pas connecté | Je tente d'accéder directement à `/dashboard` ou à une route voyage | Je suis redirigé vers l'écran de connexion, aucune donnée protégée n'est affichée | P1 |
| AUTH-07 | Refresh silencieux token expiré | Je suis connecté avec un access token expiré mais refresh token valide | Je navigue dans l'app ou effectue une action | Le refresh token est utilisé silencieusement, un nouvel access token est émis, l'action se poursuit sans interruption | P1 |
| AUTH-08 | Déconnexion manuelle | Je suis connecté | Je vais dans Paramètres et tape "Déconnexion" | Ma session est invalidée côté serveur, les tokens locaux sont supprimés, je suis redirigé vers l'onboarding | P1 |
| AUTH-09 | Connexion Apple OAuth (iOS uniquement) | Je suis sur l'écran de connexion sur un appareil iOS | Je tape "Continuer avec Apple" et j'autorise | Je suis connecté avec mon identifiant Apple, profil créé ou récupéré | P1 |
| AUTH-10 | Persistance de session au redémarrage | Je suis connecté | Je force la fermeture de l'app et la relance | Je suis toujours connecté sans avoir à saisir mes identifiants, mon voyage actif est affiché | P2 |

---

### SECTION TRIP — Création et gestion de voyage

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| TRIP-01 | Création voyage avec tous les champs obligatoires | Je suis connecté sur l'écran de création | Je saisis un nom ("Voyage à Bali"), une destination ("Bali, Indonésie"), une date de début et de fin valides, puis valide | Le voyage est créé, je suis redirigé vers la vue voyage avec itinéraire vide, un état vide guidé est affiché | P1 |
| TRIP-02 | Création voyage sans participants (champ optionnel) | Je suis sur le formulaire de création | Je renseigne les champs obligatoires et ignore le champ "Participants" | Le voyage est créé sans participants additionnels, le créateur est seul administrateur | P1 |
| TRIP-03 | Affichage du voyage en tableau de bord | J'ai créé un voyage avec dates futures | Je navigue vers le tableau de bord | Mon voyage apparaît en carte principale avec son nom, sa destination et le statut "À venir" | P1 |
| TRIP-04 | Calcul automatique du statut voyage | J'ai 3 voyages : un dont les dates sont passées, un en cours aujourd'hui, un à venir | Je consulte mon tableau de bord | Les statuts affichés sont respectivement "Passé", "En cours", "À venir" | P1 |
| TRIP-05 | Créateur automatiquement administrateur | Je crée un voyage | Le voyage est créé | Mon profil apparaît dans les participants avec le rôle "Admin", sans action supplémentaire de ma part | P1 |
| TRIP-06 | Création voyage — date de fin antérieure à date de début | Je suis sur le formulaire de création | Je saisis une date de fin antérieure à la date de début | Un message de validation s'affiche immédiatement ("La date de fin doit être postérieure à la date de début"), la soumission est bloquée | P1 |
| TRIP-07 | Création voyage — champs obligatoires manquants | Je suis sur le formulaire de création | Je laisse le nom ou la destination vide et tente de valider | Les champs en erreur sont mis en évidence, un message d'erreur contextuel s'affiche, la soumission est bloquée | P1 |
| TRIP-08 | Archivage d'un voyage passé | J'ai un voyage avec statut "Passé" | Je tape sur "Archiver" depuis le menu voyage | Le voyage passe à `is_archived = true` et disparaît de la liste principale tout en restant accessible via "Voyages archivés" | P3 |

---

### SECTION ITIN — Itinéraire

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| ITIN-01 | Affichage itinéraire jour par jour | J'ai un voyage avec 5 étapes réparties sur 3 jours | Je navigue vers l'itinéraire | Les étapes sont groupées par jour, affichées en ordre chronologique avec horaires et titres | P1 |
| ITIN-02 | Consultation du détail d'une étape | J'ai une étape "Vol Paris → Bali" avec lieu, horaire et notes renseignés | Je tape sur l'étape | Le détail s'affiche avec lieu, horaire, notes et les documents associés à cette étape | P1 |
| ITIN-03 | Étape sans horaire défini | J'ai une étape sans champ `start_time` renseigné | Je consulte l'itinéraire | L'étape s'affiche sans bloc horaire (pas de valeur nulle affichée), le tri est en fin de journée ou selon la logique définie | P2 |
| ITIN-04 | Ajout d'une étape à l'itinéraire | Je suis dans la vue itinéraire d'un voyage | Je tape "Ajouter une étape", renseigne le formulaire et valide | L'étape apparaît à la bonne position chronologique dans l'itinéraire, la vue se met à jour | P1 |
| ITIN-05 | Édition d'une étape existante | J'ai une étape avec des informations | Je tape sur "Modifier" depuis le détail de l'étape | Le formulaire est pré-rempli avec les données actuelles, je peux modifier et sauvegarder | P1 |
| ITIN-06 | Suppression d'une ét

— QA ENGINEER

*(Reprise exacte après "Je tape sur "Modifier" depuis le détail de l'étape | Le formulaire est pré-rempli avec les données actuelles, je peux modifier et sauvegarder | P1 |")*

---

| ITIN-06 | Suppression d'une étape | J'ai une étape dans mon itinéraire, je suis administrateur ou éditeur | Je tape "Supprimer" depuis le détail de l'étape et confirme la suppression | L'étape est supprimée, l'itinéraire se met à jour, les documents associés exclusivement à cette étape restent dans la liste documents du voyage | P2 |
| ITIN-07 | Accès itinéraire en mode hors connexion | J'ai consulté mon voyage au moins une fois en mode connecté | Je passe en mode avion et navigue vers l'itinéraire | L'itinéraire s'affiche en lecture seule depuis le cache local, un indicateur visuel "Hors connexion" est présent, aucune erreur n'est affichée | P1 |
| ITIN-08 | Itinéraire vide — état guidé | J'ai créé un voyage mais n'ai ajouté aucune étape | Je navigue vers l'onglet Itinéraire | Un état vide guidé s'affiche avec 2-3 actions suggérées (importer un document, ajouter une étape), pas d'écran blanc | P2 |
| ITIN-09 | Navigation vers le jour courant | Je suis pendant mon voyage, l'itinéraire comporte des jours passés et futurs | J'ouvre l'app pendant le voyage | La vue itinéraire s'ouvre directement sur la journée en cours, sans défilement manuel nécessaire | P2 |

---

### SECTION DOC — Gestion des documents

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| DOC-01 | Import document PDF valide | Je suis dans la section Documents d'un voyage | J'importe un fichier PDF de 5 Mo | Le document est uploadé, associé à la catégorie choisie, visible dans la liste avec son nom et son icône PDF | P1 |
| DOC-02 | Import document image JPG valide | Je suis dans la section Documents | J'importe un fichier JPG de 3 Mo | Le document est uploadé et associé, une miniature est affichée dans la liste | P1 |
| DOC-03 | Import document PNG valide | Je suis dans la section Documents | J'importe un fichier PNG | Le document est uploadé et associé, une miniature est affichée | P1 |
| DOC-04 | Rejet fichier format non supporté | Je suis dans la section Documents | J'importe un fichier `.docx` ou `.xlsx` | Un message d'erreur explicite s'affiche ("Format non supporté. Formats acceptés : PDF, JPG, PNG"), le fichier n'est pas uploadé | P1 |
| DOC-05 | Rejet fichier > 20 Mo | Je suis dans la section Documents | J'importe un PDF de 25 Mo | Un message d'erreur s'affiche avant l'upload ("Fichier trop volumineux. Taille maximale : 20 Mo"), l'upload est bloqué | P1 |
| DOC-06 | Fichier exactement à 20 Mo | Je suis dans la section Documents | J'importe un PDF de exactement 20 Mo | Le document est accepté et uploadé normalement (limite inclusive) | P2 |
| DOC-07 | Catégorisation d'un document | Je suis en train d'importer un document | Je sélectionne la catégorie "Vol" | Le document est associé à la catégorie "Vol" et apparaît dans le filtre correspondant | P1 |
| DOC-08 | Consultation document hors connexion | J'ai importé un document et consulté le voyage en mode connecté | Je passe en mode avion et tape sur le document | Le document s'affiche normalement depuis le cache local (PDF ou image), sans erreur réseau | P1 |
| DOC-09 | Suppression d'un document | J'ai un document dans mon voyage | Je tape "Supprimer" sur le document et confirme | Le document est supprimé de la liste et du stockage distant, les étapes auxquelles il était associé ne l'affichent plus | P2 |
| DOC-10 | Association d'un document à une étape | J'ai un document importé et une étape dans mon itinéraire | Depuis le détail de l'étape, j'associe le document | Le document apparaît dans le détail de l'étape ET reste dans la liste documents catégorisée du voyage | P2 |
| DOC-11 | Filtrage par catégorie | J'ai des documents dans plusieurs catégories | Je filtre sur "Hébergement" | Seuls les documents de la catégorie "Hébergement" sont affichés, les autres sont masqués | P2 |
| DOC-12 | Upload document simultané multiple | Je suis dans la section Documents | J'importe 3 fichiers simultanément (si multi-select supporté) | Les 3 documents sont uploadés, un indicateur de progression est visible pour chacun, aucun n'est perdu en cas de timeout partiel | P3 |

---

### SECTION TOOL — Outils utilitaires

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| TOOL-01 | Langue cible pré-remplie depuis voyage actif | J'ai un voyage actif avec destination "Bali, Indonésie" (langue : `id`) | J'ouvre le traducteur | La langue cible est automatiquement pré-sélectionnée sur "Indonésien", la langue source sur la langue de l'app (ex. "Français") | P1 |
| TOOL-02 | Traduction d'un texte en mode connecté | Je suis connecté, le traducteur est ouvert | Je saisis "Où est la gare ?" et valide | La traduction s'affiche en moins de 3 secondes, sans rechargement de page | P1 |
| TOOL-03 | Respect du délai de traduction < 3s | Je suis connecté avec une connexion standard | Je déclenche une traduction | Le résultat s'affiche en moins de 3 secondes, un indicateur de chargement est visible durant ce délai | P1 |
| TOOL-04 | Message informatif en mode hors connexion | Je suis hors connexion et j'ouvre le traducteur | Je saisis du texte et tente de traduire | Un message clair s'affiche ("La traduction nécessite une connexion internet") ou la proposition de téléchargement d'un pack hors ligne est affichée, aucune erreur technique n'est visible | P1 |
| TOOL-05 | Lecture audio de la traduction | Je suis connecté, une traduction est affichée | Je tape le bouton d'écoute audio | La phrase traduite est lue à voix haute dans la langue cible, le bouton indique visuellement la lecture en cours | P2 |
| TOOL-06 | Lecture audio indisponible hors connexion | Je suis hors connexion, une traduction est affichée (cache) | Je tape le bouton lecture audio | Un message informe que la lecture audio nécessite une connexion, aucune erreur silencieuse | P2 |
| TOOL-07 | Devise cible pré-remplie depuis voyage actif | J'ai un voyage actif avec destination "Bali, Indonésie" (devise : `IDR`) | J'ouvre le convertisseur | La devise cible est pré-sélectionnée sur "IDR (Roupie indonésienne)", la devise source sur la devise par défaut du profil (ex. EUR) | P1 |
| TOOL-08 | Conversion en temps réel | Je suis connecté, le convertisseur est ouvert avec EUR → IDR | Je saisis "100" dans le champ montant | Le montant converti s'affiche dynamiquement avec le taux de change en vigueur et sa date de dernière mise à jour | P1 |
| TOOL-09 | Conversion avec taux en cache (offline) | J'ai ouvert le convertisseur en mode connecté, les taux ont été mis en cache. Je passe hors connexion | Je saisis un montant et valide | La conversion s'effectue avec le dernier taux mis en cache, un message indique "Taux mis à jour le [date]" et que les taux peuvent ne pas être à jour | P2 |
| TOOL-10 | Inversion des devises source/cible | Le convertisseur affiche EUR → IDR | Je tape le bouton d'inversion | Les devises sont inversées (IDR → EUR), la conversion se recalcule automatiquement avec le même montant | P2 |
| TOOL-11 | Saisie montant nul ou négatif | Je suis dans le convertisseur | Je saisis "0" ou "-50" | Le résultat affiche "0" ou un message de validation bloque la saisie négative selon les règles définies, pas de valeur incohérente | P2 |
| TOOL-12 | Saisie montant très élevé | Je suis dans le convertisseur | Je saisis "999999999" | Le résultat s'affiche correctement sans overflow visuel ni erreur applicative | P3 |
| TOOL-13 | Affichage heure locale destination | J'ai un voyage actif vers "Tokyo" (Asia/Tokyo) | J'ouvre l'outil Fuseaux horaires | L'heure locale de Tokyo s'affiche en temps réel avec le décalage horaire par rapport à ma localisation | P2 |
| TOOL-14 | Comparatif heure locale utilisateur vs destination | J'ouvre l'outil Fuseaux horaires | Je consulte la vue comparative | Les deux horloges (locale et destination) s'affichent simultanément, le décalage en heures est indiqué explicitement | P2 |
| TOOL-15 | Changement de langue cible dans le traducteur | Le traducteur est ouvert avec la langue pré-remplie | Je modifie manuellement la langue cible vers "Japonais" | La traduction se relance automatiquement ou un bouton de relance apparaît, la nouvelle langue est utilisée | P2 |
| TOOL-16 | Outils contextualisés sans voyage actif | Je suis connecté mais n'ai aucun voyage actif (ou j'ai navigué hors voyage) | J'ouvre le traducteur ou le convertisseur | Les champs de langue/devise ne sont pas vides mais utilisent les préférences du profil (langue app / devise par défaut), sans erreur | P2 |

---

### SECTION MEM — Souvenirs & Partage

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| MEM-01 | Ajout d'un souvenir photo | Je suis dans l'onglet Souvenirs d'un voyage | Je tape "Ajouter un souvenir", sélectionne une photo depuis ma galerie, ajoute une note et valide | Le souvenir apparaît dans la galerie avec la photo, la note et la date d'ajout | P1 |
| MEM-02 | Ajout d'un souvenir note uniquement (sans photo) | Je suis dans l'onglet Souvenirs | J'ajoute un souvenir texte uniquement sans photo | Le souvenir est créé avec la note, il s'affiche dans la galerie avec un visuel de substitution (icône ou couleur) | P2 |
| MEM-03 | Ajout d'un souvenir avec lieu | Je suis dans l'onglet Souvenirs | J'ajoute un souvenir avec une localisation géographique | Le lieu est associé au souvenir et affiché dans sa carte | P2 |
| MEM-04 | Affichage de la galerie souvenirs | J'ai ajouté 5 souvenirs à mon voyage | Je navigue vers l'onglet Souvenirs | La galerie s'affiche avec tous les souvenirs, les photos sont optimisées (pas de temps de chargement excessif > 3s par image) | P1 |
| MEM-05 | Génération d'un lien de partage | Je suis dans l'onglet Souvenirs, au moins un souvenir existe | Je tape "Partager mon album" | Un lien unique est généré, je peux le copier ou l'envoyer via les options de partage native du système | P1 |
| MEM-06 | Accès à l'album partagé sans compte | Je possède un lien de partage valide | J'ouvre le lien dans un navigateur sans être connecté à l'app | La page publique de l'album s'affiche avec les souvenirs partagés, sans demande de connexion | P1 |
| MEM-07 | Contenu de la page publique album | J'ai accès à un lien d'album partagé | Je consulte la page publique | Les souvenirs (photos, notes) sont visibles, les informations privées du voyageur (email, données de compte) ne sont pas exposées | P1 |
| MEM-08 | Révocation d'un lien de partage | J'ai généré un lien de partage | Je tape "Désactiver le partage" depuis les options de l'album | Le lien précédemment partagé renvoie une page "Album non disponible" ou 404, les données ne sont plus accessibles publiquement | P2 |
| MEM-09 | Suppression d'un souvenir | J'ai un souvenir dans ma galerie | Je tape "Supprimer" sur le souvenir et confirme | Le souvenir est supprimé de la galerie, s'il était dans un album partagé il disparaît également de la vue publique | P2 |
| MEM-10 | Limite de taille média souvenir | Je suis dans l'ajout d'un souvenir | J'importe une photo de 25 Mo | Un message de validation s'affiche, l'import est bloqué ou la photo est redimensionnée automatiquement selon la règle définie (à préciser par PO) | P2 |
| MEM-11 | Galerie vide — état guidé | Je n'ai ajouté aucun souvenir à mon voyage | Je navigue vers l'onglet Souvenirs | Un état vide motivant s'affiche ("Vos souvenirs apparaîtront ici"), avec un CTA "Ajouter votre premier souvenir", pas d'écran blanc | P2 |
| MEM-12 | Partage d'album d'un voyage passé | J'ai un voyage avec statut "Passé" contenant des souvenirs | Je génère un lien de partage | Le lien fonctionne et l'album est accessible même si le voyage est terminé | P2 |

---

### SECTION NOTIF — Notifications & Rappels

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| NOTIF-01 | Rappel 24h avant une étape | J'ai une étape "Vol" à 10h00 demain, rappel activé à 24h | L'heure système atteint J-1 10h00 | Une notification push est reçue "Rappel : Vol — demain à 10h00", le contenu est lisible et précis | P2 |
| NOTIF-02 | Rappel 2h avant une étape | J'ai une étape avec heure définie, rappel activé à 2h | L'heure système atteint H-2

— QA ENGINEER

*(Reprise exacte à partir de NOTIF-02)*

| NOTIF-02 | Rappel 2h avant une étape | J'ai une étape avec heure définie, rappel activé à 2h | L'heure système atteint H-2 de l'étape | Une notification push est reçue avec le titre et l'heure de l'étape, dans les 5 minutes suivant l'heure cible | P2 |
| NOTIF-03 | Rappel avec délai personnalisé | J'ai configuré un rappel personnalisé à "45 minutes avant" pour une étape | L'heure cible approche | La notification est envoyée 45 minutes avant l'heure de l'étape, pas avant, pas après (tolérance : ± 5 min) | P2 |
| NOTIF-04 | Respect du fuseau horaire de l'étape | Mon fuseau local est UTC+2, mon étape est à Tokyo (UTC+9) à 14h00 | Le rappel est déclenché | La notification arrive à 14h00 heure de Tokyo (UTC+9), pas à 14h00 heure locale de l'utilisateur | P1 |
| NOTIF-05 | Désactivation globale des notifications | J'ai désactivé les notifications dans les paramètres de l'app | Une heure de rappel est atteinte | Aucune notification n'est envoyée, ni push, ni in-app | P1 |
| NOTIF-06 | Désactivation au niveau OS | J'ai autorisé les notifications dans l'app mais révoqué la permission au niveau iOS/Android | Une heure de rappel est atteinte | L'app ne plante pas, aucune notification n'est visible, la désactivation est silencieuse | P2 |
| NOTIF-07 | Étape sans heure définie | J'ai une étape avec une date mais sans heure de début | La date de l'étape arrive | Aucune notification n'est envoyée (pas de base de calcul horaire possible), pas d'erreur | P2 |
| NOTIF-08 | Notification reçue — tap pour ouvrir l'étape | Je reçois une notification de rappel | Je tape sur la notification | L'app s'ouvre directement sur le détail de l'étape concernée (deep link), pas sur la home | P2 |
| NOTIF-09 | Notification reçue — app en arrière-plan | L'app est en arrière-plan | Une notification de rappel est déclenchée | La notification apparaît dans le centre de notifications du système, le badge de l'app est mis à jour | P2 |
| NOTIF-10 | Notification reçue — app fermée | L'app est complètement fermée | Une notification de rappel est déclenchée | La notification apparaît dans le centre de notifications, l'app peut être ouverte via le tap | P2 |
| NOTIF-11 | Absence de notification en doublon | Un rappel 24h et un rappel 2h sont tous deux configurés pour la même étape | Les deux échéances arrivent | Deux notifications distinctes sont reçues, pas de duplication pour le même délai | P2 |
| NOTIF-12 | Modification de l'heure d'une étape après rappel configuré | J'ai configuré un rappel à 24h pour une étape à 10h00. Je modifie l'étape à 15h00 | J'enregistre la modification | Le rappel se recalcule automatiquement sur la nouvelle heure (J-1 15h00), l'ancien rappel est annulé | P1 |
| NOTIF-13 | Suppression d'une étape avec rappel actif | J'ai une étape avec un rappel planifié | Je supprime l'étape | Le rappel associé est annulé, aucune notification orpheline n'est envoyée | P1 |

---

## SECTION AUTH — Authentification

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| AUTH-01 | Inscription email/password — succès | Je suis sur l'écran d'inscription | Je saisis un email valide et un mot de passe conforme (min. 8 caractères) et valide | Mon compte est créé, je suis redirigé vers le tableau de bord (home), ma session est active | P1 |
| AUTH-02 | Inscription — email déjà utilisé | Je suis sur l'écran d'inscription | Je saisis un email déjà enregistré | Un message d'erreur explicite s'affiche ("Cet email est déjà utilisé"), le formulaire n'est pas réinitialisé | P1 |
| AUTH-03 | Inscription — mot de passe trop court | Je suis sur l'écran d'inscription | Je saisis un mot de passe de moins de 8 caractères | Une erreur de validation inline s'affiche avant soumission, la requête serveur n'est pas déclenchée | P1 |
| AUTH-04 | Connexion email/password — succès | Je suis sur l'écran de connexion, mon compte existe | Je saisis mes identifiants corrects et valide | Je suis connecté et redirigé vers le tableau de bord | P1 |
| AUTH-05 | Connexion — mot de passe incorrect | Je suis sur l'écran de connexion | Je saisis le bon email mais un mauvais mot de passe | Un message d'erreur générique s'affiche ("Identifiants incorrects"), sans préciser lequel est faux | P1 |
| AUTH-06 | Connexion OAuth Google | Je suis sur l'écran de connexion | Je tape "Continuer avec Google" et complète le flux OAuth | Je suis connecté, mon profil est créé si c'est la première fois, redirigé vers le tableau de bord | P1 |
| AUTH-07 | Connexion OAuth Apple (iOS uniquement) | Je suis sur l'écran de connexion sur iOS | Je tape "Continuer avec Apple" et complète le flux | Je suis connecté, mon profil est créé si c'est la première fois | P1 |
| AUTH-08 | Persistance de session au redémarrage | Je suis connecté et je ferme complètement l'app | Je rouvre l'app | Je suis directement redirigé vers le tableau de bord sans re-saisir mes identifiants | P1 |
| AUTH-09 | Expiration du token d'accès | Mon token JWT d'accès est expiré (durée dépassée) | Je fais une action nécessitant une authentification | Le refresh token est utilisé automatiquement, un nouveau token est émis, l'action se poursuit sans interruption visible | P1 |
| AUTH-10 | Expiration du refresh token | Mon refresh token est expiré | Je tente d'effectuer une action | Je suis redirigé vers l'écran de connexion avec un message "Votre session a expiré, veuillez vous reconnecter" | P1 |
| AUTH-11 | Déconnexion | Je suis connecté | Je tape "Déconnexion" dans les paramètres et confirme | Ma session est terminée, les tokens sont invalidés, je suis redirigé vers l'onboarding/connexion | P1 |
| AUTH-12 | Accès à une route protégée sans session | Je ne suis pas connecté | Je tente d'accéder directement à une URL protégée (ex. `/dashboard`) | Je suis redirigé vers l'écran de connexion, la route demandée est mémorisée pour redirection post-connexion | P1 |
| AUTH-13 | Onboarding — affichage unique | Je n'ai jamais utilisé l'app | J'ouvre l'app pour la première fois | Les 3 slides de valeur proposition s'affichent, le CTA "Commencer" mène à l'inscription | P2 |
| AUTH-14 | Onboarding — non affiché au retour | J'ai déjà créé un compte et me reconnecte | J'ouvre l'app | L'onboarding n'est pas affiché, je vais directement à la connexion ou au tableau de bord si session active | P2 |

---

## SECTION OFF — Mode Hors Connexion

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| OFF-01 | Accès à l'itinéraire hors connexion | J'ai consulté mon voyage en mode connecté, les données sont en cache | Je passe hors connexion et navigue vers l'itinéraire | L'itinéraire s'affiche en lecture seule depuis le cache, un indicateur visuel "Hors connexion" est visible | P1 |
| OFF-02 | Accès aux documents hors connexion | J'ai importé des documents qui ont été synchronisés | Je passe hors connexion et ouvre la liste des documents | Les documents préalablement téléchargés sont accessibles, les documents non mis en cache affichent un indicateur "Non disponible hors connexion" | P1 |
| OFF-03 | Indicateur visuel offline — cohérence | Je suis hors connexion | Je navigue dans l'app | Un bandeau ou indicateur persistant signale l'état hors connexion sur toutes les vues, sans masquer le contenu | P1 |
| OFF-04 | Tentative d'action d'écriture hors connexion | Je suis hors connexion | Je tente d'ajouter une étape, d'importer un document ou d'ajouter un souvenir | Un message clair informe que cette action nécessite une connexion, l'action est soit bloquée, soit mise en file d'attente selon la stratégie définie | P1 |
| OFF-05 | Retour en ligne — synchronisation automatique | J'étais hors connexion avec des actions en file d'attente (si stratégie queue retenue) | Ma connexion est rétablie | La synchronisation s'effectue automatiquement, un feedback discret confirme la mise à jour ("Synchronisé") | P2 |
| OFF-06 | Cache itinéraire — fraîcheur des données | J'ai mis en cache l'itinéraire il y a 48h et je suis hors connexion | Je consulte l'itinéraire | Les données affichées correspondent au dernier état synchronisé, la date de dernière mise à jour est visible | P2 |
| OFF-07 | Traducteur hors connexion | Je suis hors connexion | J'ouvre le traducteur et tente une traduction | Un message informe que la traduction nécessite une connexion, le champ de saisie reste accessible | P1 |
| OFF-08 | Convertisseur hors connexion avec cache taux | J'ai utilisé le convertisseur en mode connecté (taux mis en cache) et je passe hors connexion | J'ouvre le convertisseur et saisis un montant | La conversion s'effectue avec le taux en cache, une mention "Taux du [date]" et "Connexion requise pour actualiser" s'affiche | P2 |

---

## SECTION PERF — Performance & Sécurité

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| PERF-01 | Temps de traduction < 3s | Je suis connecté avec une connexion standard (4G ou WiFi) | Je déclenche une traduction de texte courant (< 200 caractères) | Le résultat s'affiche en moins de 3 secondes, mesuré depuis la validation jusqu'à l'affichage | P1 |
| PERF-02 | Chargement de la galerie souvenirs < 3s par image | J'ai 20 souvenirs avec photos dans mon voyage | Je navigue vers la galerie | Toutes les miniatures s'affichent en moins de 3 secondes, les images en plein écran se chargent de manière progressive | P2 |
| PERF-03 | Taille maximale document importé — 20 Mo | Je suis dans la section Documents | J'importe un fichier de 21 Mo | L'import est refusé avec un message explicite ("Fichier trop volumineux, maximum 20 Mo"), sans crash | P1 |
| PERF-04 | Aucune clé API tierce exposée côté client | L'app est en production | J'inspecte les requêtes réseau depuis l'app (proxy/debug) | Aucune clé DeepL, Fixer.io ou autre API tierce n'apparaît dans les requêtes émises par l'app mobile/web | P1 |
| PERF-05 | Données privées non exposées dans l'album public | Un album de souvenirs est partagé publiquement | Une personne sans compte accède au lien d'album | Les emails, données de profil ou informations personnelles du créateur ne sont pas visibles dans la page publique | P1 |
| PERF-06 | Quota document par fichier | Je suis dans la section Documents | J'importe un fichier PDF de 20 Mo exactement | Le fichier est accepté et importé avec succès (borne incluse) | P2 |

---

## RAPPORT DE VALIDATION — Critères d'Acceptation PO

| ID Critère | User Story | Critère d'acceptation | Cas de test couvrant | Statut | Commentaire |
|---|---|---|---|---|---|
| AC-A01-1 | US-A01 Création voyage | Saisie : nom, destination(s), dates, participants (optionnel) | TRIP-01, TRIP-02 | ✅ Couvert | |
| AC-A01-2 | US-A01 Création voyage | Voyage apparaît dans tableau de bord avec statut (à venir/en cours/passé) | TRIP-03 | ✅ Couvert | |
| AC-A01-3 | US-A01 Création voyage | Créateur automatiquement administrateur | TRIP-04 | ✅ Couvert | |
| AC-A02-1 | US-A02 Itinéraire | Vue chronologique par jour avec étapes, horaires, lieux | TRIP-07 | ✅ Couvert | |
| AC-A02-2 | US-A02 Itinéraire | Détail d'une étape accessible (lieu, horaire, notes, docs associés) | TRIP-08 | ✅ Couvert | |
| AC-A02-3 | US-A02 Itinéraire | Accès hors connexion en lecture seule via cache | OFF-01 | ✅ Couvert | |
| AC-A03-1 | US-A03 Documents | Formats acceptés : PDF, JPG, PNG | DOC-01 | ✅ Couvert | |
| AC-A03-2 | US-A03 Documents | Association à une catégorie (vol/hébergement/activité/autre) | DOC-02 | ✅ Couvert | |
| AC-A03-3 | US-A03 Documents | Accessible hors connexion | OFF-02 | ✅ Couvert | |
| AC-A03-4 | US-A03 Documents | Taille maximale 20 Mo par fichier | PERF-03, PERF-06 | ✅ Couvert | Borne haute ET limite exacte testées |
| AC-A04-1 | US-A04 Notifications | Notification envoyée selon délai paramétrable (24h, 2h, personnalisé) | NOTIF-01, NOTIF-02, NOTIF-03 | ✅ Couvert | |
| AC-A04-2 | US-A04 Notifications | Désactivation → aucune notification | NOTIF-05 |

— QA ENGINEER

*(Suite et fin du livrable — reprise après "Désactivation → aucune notification | NOTIF-05 |")*

| AC-A04-2 | US-A04 Notifications | Désactivation → aucune notification | NOTIF-05 | ✅ Couvert | |
| AC-A04-3 | US-A04 Notifications | Rappels tenant compte du fuseau horaire de l'étape | NOTIF-04 | ✅ Couvert | |
| AC-B01-1 | US-B01 Traducteur | Langue cible pré-sélectionnée selon destination du voyage actif | TOOL-01 | ✅ Couvert | |
| AC-B01-2 | US-B01 Traducteur | Résultat en moins de 3 secondes en mode connecté | PERF-01 | ✅ Couvert | |
| AC-B01-3 | US-B01 Traducteur | Message informatif si hors connexion (ou pack offline) | OFF-07 | ✅ Couvert | |
| AC-B01-4 | US-B01 Traducteur | Lecture audio de la traduction | TOOL-03 | ✅ Couvert | |
| AC-B02-1 | US-B02 Convertisseur | Devise cible pré-remplie selon destination voyage actif | TOOL-06 | ✅ Couvert | |
| AC-B02-2 | US-B02 Convertisseur | Taux actualisés (fréquence à préciser) | TOOL-07 | ✅ Couvert | |
| AC-B02-3 | US-B02 Convertisseur | Taux en cache avec mention date si hors connexion | OFF-08 | ✅ Couvert | |
| AC-C01-1 | US-C01 Souvenirs | Ajout photo/note/lieu à un voyage | MEM-01, MEM-02, MEM-03 | ✅ Couvert | |
| AC-C01-2 | US-C01 Souvenirs | Génération d'un lien de partage public | MEM-05 | ✅ Couvert | |
| AC-C01-3 | US-C01 Souvenirs | Accès à l'album public sans compte | MEM-06 | ✅ Couvert | |
| AC-C01-4 | US-C01 Souvenirs | Données privées non exposées dans l'album public | PERF-05 | ✅ Couvert | |
| AC-AUTH-1 | Auth générale | Inscription email/password fonctionnelle | AUTH-01, AUTH-02, AUTH-03 | ✅ Couvert | |
| AC-AUTH-2 | Auth générale | Connexion OAuth Google + Apple | AUTH-06, AUTH-07 | ✅ Couvert | Apple iOS uniquement |
| AC-AUTH-3 | Auth générale | Session persistante au redémarrage | AUTH-08 | ✅ Couvert | |
| AC-AUTH-4 | Auth générale | Refresh silencieux à expiration du token | AUTH-09 | ✅ Couvert | |
| AC-AUTH-5 | Auth générale | Routes protégées inaccessibles sans session | AUTH-12 | ✅ Couvert | |
| AC-SEC-1 | Sécurité | Aucune clé API tierce exposée côté client | PERF-04 | ✅ Couvert | |

---

## SECTION COMPLÈTE — Cas de tests manquants identifiés en continuation

Les sections suivantes complètent le plan de test pour les modules non encore détaillés.

---

### SECTION TRIP — Gestion des voyages et itinéraire (suite)

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| TRIP-01 | Création d'un voyage — champs obligatoires | Je suis connecté et sur l'écran de création | Je saisis le nom du voyage, la destination et les dates puis valide | Le voyage est créé et apparaît dans mon tableau de bord | P1 |
| TRIP-02 | Création d'un voyage — participants optionnels | Je suis connecté et sur l'écran de création | Je crée un voyage sans ajouter de participants | Le voyage est créé normalement, le champ participants est ignoré sans erreur | P1 |
| TRIP-03 | Statut automatique du voyage | J'ai des voyages avec dates passées, en cours et futures | J'accède à mon tableau de bord | Chaque voyage affiche correctement son statut : "Passé", "En cours" ou "À venir" selon les dates | P1 |
| TRIP-04 | Créateur désigné administrateur | Je crée un voyage | Le voyage est créé | Mon rôle sur ce voyage est "admin", visible dans la gestion des participants | P1 |
| TRIP-05 | Création — date de fin antérieure à la date de début | Je suis sur le formulaire de création | Je saisis une date de fin antérieure à la date de début | Le formulaire affiche une erreur de validation, la création est bloquée | P1 |
| TRIP-06 | Création — champs obligatoires manquants | Je suis sur le formulaire de création | Je valide sans saisir le nom ou la destination | Une erreur de validation inline s'affiche sur les champs manquants, la création est bloquée | P1 |
| TRIP-07 | Consultation de l'itinéraire — vue chronologique | J'ai un voyage avec plusieurs étapes sur plusieurs jours | J'ouvre la vue itinéraire | Les étapes s'affichent regroupées par jour en ordre chronologique, avec horaires et lieux | P1 |
| TRIP-08 | Détail d'une étape | J'ai un voyage avec des étapes documentées | Je tape sur une étape dans l'itinéraire | Le détail s'affiche : titre, lieu, horaire, notes, documents associés, et lien vers Maps si coordonnées disponibles | P1 |
| TRIP-09 | Ajout d'une étape | Je suis dans l'itinéraire de mon voyage | J'ajoute une nouvelle étape avec titre, type, date, heure, lieu | L'étape apparaît à la bonne place chronologique dans l'itinéraire | P1 |
| TRIP-10 | Édition d'une étape | J'ai une étape existante | Je modifie l'heure de début de l'étape | L'étape est mise à jour, l'ordre chronologique est recalculé si nécessaire | P2 |
| TRIP-11 | Suppression d'un voyage | Je suis l'administrateur d'un voyage | Je supprime le voyage | Le voyage disparaît de mon tableau de bord, une confirmation est demandée avant suppression | P2 |
| TRIP-12 | Invitation d'un participant | Je suis administrateur d'un voyage | J'invite un participant par email | Le participant reçoit une invitation, son statut apparaît "En attente" dans la liste des participants | P2 |
| TRIP-13 | Voyage actif en premier sur le tableau de bord | J'ai plusieurs voyages (passés, à venir, en cours) | J'ouvre le tableau de bord | Le voyage en cours s'affiche en position principale/prioritaire | P1 |
| TRIP-14 | État vide — premier voyage | Je suis connecté et n'ai aucun voyage | J'ouvre le tableau de bord | Un état vide guidé s'affiche avec un CTA proéminent "Créer mon premier voyage", sans écran blanc | P1 |
| TRIP-15 | Itinéraire vide — guidance post-création | Je viens de créer un voyage | J'accède à la vue itinéraire | Un état vide guidé propose des actions : "Ajouter une étape", "Importer un document" | P2 |

---

### SECTION DOC — Gestion des documents

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| DOC-01 | Import — formats acceptés (PDF) | Je suis dans la section Documents de mon voyage | J'importe un fichier PDF valide | Le document est importé avec succès et apparaît dans la liste | P1 |
| DOC-02 | Import — catégorisation | J'importe un document PDF | Le document est importé | Je peux lui associer une catégorie parmi : vol / hébergement / activité / autre | P1 |
| DOC-03 | Import — format JPG | Je suis dans la section Documents | J'importe un fichier JPG | Le document est importé avec succès | P1 |
| DOC-04 | Import — format PNG | Je suis dans la section Documents | J'importe un fichier PNG | Le document est importé avec succès | P1 |
| DOC-05 | Import — format non supporté (ex. DOCX) | Je suis dans la section Documents | J'importe un fichier .docx | L'import est refusé avec un message explicite listant les formats acceptés (PDF, JPG, PNG) | P1 |
| DOC-06 | Import — fichier de 21 Mo (dépassement limite) | Je suis dans la section Documents | J'importe un fichier de 21 Mo | L'import est refusé avec un message "Fichier trop volumineux, maximum 20 Mo", sans crash de l'app | P1 |
| DOC-07 | Import — fichier de 20 Mo exactement (borne incluse) | Je suis dans la section Documents | J'importe un fichier de 20 Mo exactement | Le fichier est accepté et importé avec succès | P2 |
| DOC-08 | Visualisation d'un document importé | J'ai importé un PDF dans mon voyage | Je tape sur le document dans la liste | Le document s'ouvre dans la visionneuse intégrée, lisible sans quitter l'app | P1 |
| DOC-09 | Accès aux documents hors connexion | J'ai importé et synchronisé des documents | Je passe hors connexion et ouvre la liste des documents | Les documents téléchargés sont accessibles, les non-mis-en-cache affichent un indicateur "Non disponible hors connexion" | P1 |
| DOC-10 | Association document ↔ étape | J'ai un document importé et une étape dans mon itinéraire | J'associe le document à une étape spécifique | Le document apparaît dans le détail de l'étape correspondante | P2 |
| DOC-11 | Suppression d'un document | J'ai un document importé | Je supprime le document | Le document disparaît de la liste, une confirmation est demandée, le fichier est supprimé du stockage | P2 |
| DOC-12 | Liste catégorisée | J'ai importé plusieurs documents dans différentes catégories | Je consulte la liste des documents | Les documents sont organisés et filtrables par catégorie (vol / hébergement / activité / autre) | P2 |

---

### SECTION TOOL — Outils utilitaires

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| TOOL-01 | Traducteur — langue pré-remplie | J'ai un voyage actif vers "Japon" | J'ouvre le traducteur | La langue cible est automatiquement définie sur "Japonais" | P1 |
| TOOL-02 | Traducteur — traduction standard | Je suis connecté et dans le traducteur | Je saisis "Bonjour, je voudrais une table pour deux" et valide | La traduction s'affiche en moins de 3 secondes | P1 |
| TOOL-03 | Traducteur — lecture audio | J'ai obtenu une traduction | Je tape le bouton lecture audio | La prononciation du texte traduit est jouée via le haut-parleur de l'appareil | P1 |
| TOOL-04 | Traducteur — changement de langue manuel | Je suis dans le traducteur avec la langue pré-remplie | Je change manuellement la langue cible | La traduction se met à jour dans la nouvelle langue sélectionnée | P2 |
| TOOL-05 | Traducteur — texte vide | Je suis dans le traducteur | Je valide sans saisir de texte | Aucune requête n'est envoyée, un message invite à saisir du texte | P2 |
| TOOL-06 | Convertisseur — devise pré-remplie | J'ai un voyage actif vers "Thaïlande" (THB) | J'ouvre le convertisseur | La devise cible est automatiquement définie sur "THB - Baht thaïlandais" | P1 |
| TOOL-07 | Convertisseur — conversion standard | Je suis dans le convertisseur avec des taux chargés | Je saisis "100 EUR" et valide | Le montant converti s'affiche avec le taux utilisé et sa date de mise à jour | P1 |
| TOOL-08 | Convertisseur — montant à zéro | Je suis dans le convertisseur | Je saisis "0" | Le résultat affiché est "0" dans la devise cible, sans erreur | P2 |
| TOOL-09 | Convertisseur — montant négatif | Je suis dans le convertisseur | Je saisis un montant négatif (-50) | Le formulaire refuse la saisie ou affiche un message de validation (montant doit être positif) | P2 |
| TOOL-10 | Horloge / fuseaux horaires — affichage | J'ai un voyage actif vers "New York" (EST) | J'ouvre l'outil horloge | L'heure locale de la destination et l'heure locale de l'utilisateur s'affichent simultanément avec le décalage horaire | P1 |
| TOOL-11 | Outils accessibles sans voyage actif | Je n'ai aucun voyage actif ou je n'en ai pas créé | J'ouvre le hub des outils | Les outils sont accessibles mais sans pré-remplissage contextuel, les champs langue/devise sont vides ou sur une valeur par défaut | P2 |
| TOOL-12 | Traducteur — texte long (> 500 caractères) | Je suis dans le traducteur | Je saisis un texte de 600 caractères | L'app gère la requête sans erreur : soit elle traduit, soit elle affiche une limite explicite | P3 |

---

### SECTION MEM — Souvenirs et partage

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| MEM-01 | Ajout d'un souvenir — photo | Je suis dans la section Souvenirs de mon voyage | J'ajoute une photo depuis ma galerie ou l'appareil photo | La photo apparaît dans la galerie du voyage | P1 |
| MEM-02 | Ajout d'un souvenir — note texte | Je suis dans la section Souvenirs | J'ajoute une note textuelle | La note apparaît dans la galerie avec sa date de création | P1 |
| MEM-03 | Ajout d'un souvenir — lieu associé | J'ajoute un souvenir (photo ou note) | Je peux associer un lieu au souvenir | Le lieu s'affiche sur la carte/détail du souvenir | P2 |
| MEM-04 | Galerie — affichage | J'ai 10 souvenirs dans mon voyage | Je navigue vers la galerie | Les souvenirs s'affichent en grille ou liste, de man

— QA ENGINEER

*(Continuation du livrable — reprise après MEM-04)*

---

| MEM-04 | Galerie — affichage | J'ai 10 souvenirs dans mon voyage | Je navigue vers la galerie | Les souvenirs s'affichent en grille ou liste, de manière chronologique, avec miniatures pour les photos | P1 |
| MEM-05 | Suppression d'un souvenir | J'ai un souvenir dans ma galerie | Je supprime un souvenir | Le souvenir disparaît de la galerie, une confirmation est demandée avant suppression | P2 |
| MEM-06 | Génération d'un lien de partage | J'ai au moins un souvenir dans mon voyage | Je génère un lien de partage de l'album | Un lien unique est créé et copiable, accessible par des personnes non inscrites à l'app | P1 |
| MEM-07 | Consultation de l'album partagé — non inscrit | Un proche a reçu le lien d'un album | Il ouvre le lien dans un navigateur | Il voit les souvenirs partagés en lecture seule, sans obligation de créer un compte | P1 |
| MEM-08 | Révocation d'un lien de partage | J'ai généré un lien de partage actif | Je révoque le lien depuis mes paramètres | Le lien n'est plus accessible, la page retourne une erreur explicite (lien expiré ou révoqué) | P2 |
| MEM-09 | Album partagé — contenu vide | Je génère un lien de partage avant d'avoir ajouté des souvenirs | Un proche ouvre le lien | Une page vide mais cohérente s'affiche, sans erreur technique | P3 |
| MEM-10 | Souvenirs non intrusifs — absence de push automatique | Je n'ai pas explicitement partagé de souvenirs | L'app s'abstient d'envoyer des notifications push incitant à partager | Aucune notification de type "partagez vos souvenirs !" n'est émise sans action initiatrice de l'utilisateur | P1 |

---

### SECTION NOTIF — Notifications et rappels

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| NOTIF-01 | Rappel 24h avant une étape | J'ai une étape avec une heure définie, les notifications sont activées | 24h avant l'étape | Une notification push est reçue avec le titre et l'heure de l'étape | P2 |
| NOTIF-02 | Rappel 2h avant une étape | J'ai une étape avec une heure définie, les notifications sont activées | 2h avant l'étape | Une notification push est reçue | P2 |
| NOTIF-03 | Rappel personnalisé | J'ai configuré un délai personnalisé pour une étape | Au délai configuré avant l'étape | Une notification push est reçue selon le délai personnalisé | P2 |
| NOTIF-04 | Désactivation globale des notifications | J'ai désactivé les notifications dans mes paramètres | N'importe quel moment | Aucune notification n'est envoyée, quel que soit le type d'étape | P2 |
| NOTIF-05 | Fuseau horaire de l'étape | J'ai une étape à 10h00 heure locale de Tokyo (JST) et je me trouve en France (CET) | Le rappel est calculé | La notification arrive à l'heure correcte tenant compte du fuseau horaire de l'étape (et non du fuseau de l'utilisateur) | P2 |
| NOTIF-06 | Étape sans heure définie | J'ai une étape avec une date mais sans heure | Le délai de rappel est atteint | Aucune notification n'est envoyée pour cette étape spécifique (ou notification à une heure par défaut si spécifiée) | P3 |

---

### SECTION OFF — Mode hors connexion

| ID | Scénario | Given | When | Then | Priorité |
|---|---|---|---|---|---|
| OFF-01 | Consultation itinéraire hors connexion | J'ai consulté mon voyage au moins une fois en ligne | Je passe hors connexion et ouvre l'itinéraire | L'itinéraire s'affiche en lecture seule via le cache local, sans message d'erreur | P1 |
| OFF-02 | Documents hors connexion — déjà mis en cache | J'ai ouvert un document en ligne (mise en cache) | Je passe hors connexion et consulte ce document | Le document s'affiche normalement | P1 |
| OFF-03 | Documents hors connexion — non mis en cache | Je n'ai jamais ouvert un document spécifique | Je passe hors connexion et tente d'accéder à ce document | Un indicateur visuel "Non disponible hors connexion" s'affiche, sans crash | P1 |
| OFF-04 | Indicateur de mode hors connexion | Je suis hors connexion dans l'app | Je navigue dans l'app | Un indicateur visuel clair (bannière, icône) signale que l'app est en mode hors connexion | P1 |
| OFF-05 | Tentative de traduction hors connexion | Je suis hors connexion | J'ouvre le traducteur et tente une traduction | Un message clair m'informe que la traduction nécessite une connexion, sans crash | P1 |
| OFF-06 | Tentative de conversion de devises hors connexion — taux en cache | Les taux de change ont été chargés récemment (< 24h) | Je passe hors connexion et ouvre le convertisseur | La conversion s'effectue avec les derniers taux connus, une mention "Taux mis à jour le [date]" est affichée | P2 |
| OFF-07 | Tentative de conversion hors connexion — aucun taux en cache | Les taux n'ont jamais été chargés | Je passe hors connexion et ouvre le convertisseur | Un message indique que la conversion nécessite une connexion pour initialiser les taux | P2 |
| OFF-08 | Reprise de connexion — synchronisation | J'ai modifié des données localement hors connexion | Je retrouve une connexion | Les données sont synchronisées avec le serveur dans le bon sens, sans perte ni doublon | P2 |

---

## 4. CAS LIMITES ET SCÉNARIOS D'ERREUR

| ID | Catégorie | Scénario limite | Comportement attendu | Sévérité si non géré |
|---|---|---|---|---|
| EDGE-01 | Données | Création d'un voyage avec date de fin antérieure à la date de début | Validation bloquante côté formulaire avec message explicite | Critique |
| EDGE-02 | Données | Voyage avec une seule étape sur une seule journée | L'itinéraire s'affiche correctement avec un seul jour | Majeur |
| EDGE-03 | Données | Voyage avec 0 participant autre que le créateur | L'app fonctionne normalement, section participants affiche uniquement le créateur | Mineur |
| EDGE-04 | Réseau | Perte de connexion durant un upload de document | L'upload échoue proprement, le fichier n'est pas corrompu ni partiellement importé, l'utilisateur est notifié et peut réessayer | Critique |
| EDGE-05 | Réseau | Perte de connexion durant une traduction en cours | La requête échoue, un message invite à réessayer, l'app ne reste pas bloquée en état "chargement" | Majeur |
| EDGE-06 | Performance | Voyage avec 50+ étapes | L'itinéraire se charge et se scrolle sans dégradation visible des performances | Majeur |
| EDGE-07 | Performance | Galerie souvenirs avec 100+ photos | La galerie utilise une virtualisation / lazy loading, pas de freeze de l'interface | Majeur |
| EDGE-08 | Sécurité | Accès à un voyage dont l'utilisateur n'est pas participant | L'API retourne 403, l'app affiche un écran d'accès refusé explicite, sans exposition de données | Critique |
| EDGE-09 | Sécurité | Lien de partage album accédé après révocation | La page affiche "Ce lien n'est plus valide" sans exposer de données | Critique |
| EDGE-10 | Sécurité | Upload d'un fichier déguisé (ex. .exe renommé en .pdf) | Le backend valide le MIME type réel, l'upload est refusé avec message d'erreur | Critique |
| EDGE-11 | Concurrence | Deux utilisateurs modifient la même étape simultanément | Le dernier enregistrement gagne (last-write-wins) ou un conflit est signalé à l'utilisateur | Majeur |
| EDGE-12 | Session | Token JWT expiré en cours d'utilisation | Le refresh token prend le relais de façon silencieuse, l'utilisateur ne voit aucune interruption | Critique |
| EDGE-13 | Session | Refresh token expiré | L'utilisateur est redirigé vers l'écran de connexion avec un message explicite (session expirée), sans perte de données | Majeur |
| EDGE-14 | Compatibilité | Nom de voyage contenant des caractères spéciaux (émojis, caractères arabes, japonais) | Le nom s'affiche correctement partout dans l'app (tableau de bord, en-tête voyage, notifications) | Majeur |
| EDGE-15 | Données | Destination sans devise locale connue (territoire peu courant) | Le convertisseur n'effectue pas de pré-remplissage et invite l'utilisateur à sélectionner manuellement | Mineur |
| EDGE-16 | Stockage | Utilisateur ayant atteint une éventuelle limite de stockage documents | Un message clair indique la limite atteinte, l'import est bloqué proprement | Majeur |
| EDGE-17 | API tierce | API DeepL indisponible (timeout ou erreur 500) | Un message d'erreur convivial s'affiche, l'app ne crash pas, possibilité de réessayer | Majeur |
| EDGE-18 | API tierce | API Fixer.io retourne un taux à 0 ou null | Le convertisseur affiche une erreur et n'affiche pas "0.00 XXX" comme résultat valide | Majeur |
| EDGE-19 | Offline | Synchronisation conflictuelle à la reprise de connexion | Aucune donnée n'est silencieusement écrasée, la stratégie last-write-wins est documentée et appliquée de façon cohérente | Majeur |
| EDGE-20 | Accessibilité | Utilisation avec taille de texte système augmentée (accessibilité) | Les composants ne se chevauchent pas, le texte reste lisible | Mineur |

---

## 5. RAPPORT DE VALIDATION — CRITÈRES D'ACCEPTATION PO

Ce tableau vérifie la couverture des critères d'acceptation définis dans le livrable Product Owner.

### US-A01 — Création d'un voyage

| # | Critère d'acceptation PO | Cas de test couvrant | Statut couverture |
|---|---|---|---|
| A01-CA1 | Saisie : nom, destination, dates de début/fin, participants (optionnel) | TRIP-01, TRIP-02, TRIP-03 | ✅ Couvert |
| A01-CA2 | Voyage apparaît dans le tableau de bord avec statut (à venir / en cours / passé) | TRIP-04, TRIP-05, TRIP-13 | ✅ Couvert |
| A01-CA3 | Créateur automatiquement désigné administrateur | AUTH-07 (rôle admin à valider post-création) | ⚠️ Partiellement couvert — cas de test dédié à créer |

> **Action requise** : Ajouter TRIP-16 — *Vérification du rôle administrateur automatique à la création d'un voyage*.

---

### US-A02 — Consultation de l'itinéraire

| # | Critère d'acceptation PO | Cas de test couvrant | Statut couverture |
|---|---|---|---|
| A02-CA1 | Vue chronologique par jour avec étapes, horaires, lieux | TRIP-07, TRIP-08 | ✅ Couvert |
| A02-CA2 | Détail d'une étape (lieu, horaire, notes, documents associés) | TRIP-08, DOC-10 | ✅ Couvert |
| A02-CA3 | Accès hors connexion en lecture seule via cache | OFF-01 | ✅ Couvert |

---

### US-A03 — Import de documents

| # | Critère d'acceptation PO | Cas de test couvrant | Statut couverture |
|---|---|---|---|
| A03-CA1 | Formats acceptés : PDF, JPG, PNG | DOC-01, DOC-03, DOC-04, DOC-05 | ✅ Couvert |
| A03-CA2 | Association à une catégorie (vol / hébergement / activité / autre) | DOC-02, DOC-12 | ✅ Couvert |
| A03-CA3 | Document accessible hors connexion | OFF-02, OFF-03 | ✅ Couvert |
| A03-CA4 | Taille maximale 20 Mo par fichier | DOC-06, DOC-07 | ✅ Couvert |

---

### US-A04 — Notifications et rappels

| # | Critère d'acceptation PO | Cas de test couvrant | Statut couverture |
|---|---|---|---|
| A04-CA1 | Notification selon délai paramétrable (24h, 2h, personnalisé) | NOTIF-01, NOTIF-02, NOTIF-03 | ✅ Couvert |
| A04-CA2 | Désactivation → aucune notification | NOTIF-04 | ✅ Couvert |
| A04-CA3 | Rappels tenant compte du fuseau horaire de l'étape | NOTIF-05 | ✅ Couvert |

---

### US-B01 — Traducteur intégré

| # | Critère d'acceptation PO | Cas de test couvrant | Statut couverture |
|---|---|---|---|
| B01-CA1 | Langue cible pré-sélectionnée selon destination du voyage actif | TOOL-01 | ✅ Couvert |
| B01-CA2 | Résultat en moins de 3 secondes en mode connecté | TOOL-02 (critère de performance à instrumenter) | ⚠️ Partiellement couvert — nécessite test de performance |
| B01-CA3 | Message hors connexion ou pack hors ligne | OFF-05 | ✅ Couvert |
| B01-CA4 | Lecture audio de la traduction | TOOL-03 | ✅ Couvert |

> **Action requise** : Ajouter un test de performance PERF-01 — *Le traducteur retourne un résultat en moins de 3 secondes sur une connexion 4G standard (mesure instrumentée)*.

---

### US-B02 — Convertisseur de devises

| # | Critère d'acceptation PO | Cas de test couvrant | Statut