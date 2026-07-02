# ux_designer

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— UX DESIGNER

# Livrable UX Designer — Travel Companion App

---

## 0. LECTURE DU CONTEXTE

**Persona principal retenu pour le MVP** : Le Voyageur autonome (B2C) — tech-friendly, veut tout en un endroit, supporte mal la fragmentation.

**Persona secondaire pris en compte** : Le Client d'agence — peu habitué aux outils numériques, reçoit un voyage clé en main, doit trouver sans chercher.

**Contrainte structurante** : L'app doit fonctionner partiellement hors connexion. Ce paramètre impacte l'architecture autant que le design.

**Tension centrale identifiée** : Une app "tout-en-un" peut vite devenir une app "rien de bien". La discipline de simplification est l'enjeu UX n°1.

---

## 1. PRINCIPES UX DIRECTEURS

**P1 — Le voyage est le contexte permanent**
Toute navigation, tout outil, toute action s'ancre dans le voyage actif. L'utilisateur ne doit jamais avoir à re-préciser où il est, quelle devise il utilise, quelle langue parler. L'app le sait.

**P2 — Zéro effort pour consulter, effort minimal pour créer**
La consultation (itinéraire, documents, outils) est accessible en 1 tap depuis n'importe où. La création et l'import peuvent demander plus d'étapes, mais jamais plus de 4.

**P3 — L'offline est une feature, pas une dégradation**
Ce qui est disponible hors connexion doit être identifiable visuellement et fonctionner sans message d'erreur visible. Ce qui ne l'est pas s'affiche comme désactivé, pas comme cassé.

**P4 — Les outils utilitaires sont des invités, pas des résidents**
Le traducteur, le convertisseur, l'horloge : présents, accessibles, mais jamais au premier plan. Ils ne doivent pas encombrer la vue voyage principale.

**P5 — Le partage de souvenirs est une récompense, pas une obligation**
Le module souvenirs est émotionnel. Il s'active quand le voyageur choisit de partager, jamais en mode push intrusif. L'expérience doit être plaisante, pas forcée.

---

## 2. ARBORESCENCE DE L'APPLICATION

```
APP TRAVEL COMPANION
│
├── ONBOARDING
│   ├── Écran d'accueil / valeur proposition
│   ├── Création de compte / Connexion
│   └── Configuration initiale (optionnelle : devise, langue, notifications)
│
├── TABLEAU DE BORD (home)
│   ├── Voyage actif (carte principale)
│   ├── Mes voyages (liste : à venir / en cours / passé)
│   └── Accès rapide outils (barre contextuelle)
│
├── VOYAGE [vue centrale]
│   ├── Itinéraire (vue jour par jour)
│   │   ├── Détail d'une étape
│   │   └── Ajout / édition d'une étape
│   ├── Documents
│   │   ├── Liste par catégorie (vol / hébergement / activité / autre)
│   │   ├── Visualisation d'un document
│   │   └── Import d'un document
│   ├── Participants
│   │   ├── Liste des participants
│   │   └── Invitation d'un participant
│   └── Souvenirs
│       ├── Galerie du voyage
│       ├── Ajout de souvenir (photo, note, lieu)
│       └── Partage (lien, album)
│
├── OUTILS (barre persistante ou hub dédié)
│   ├── Traducteur
│   │   ├── Saisie texte
│   │   ├── Détection / sélection de langue
│   │   └── Résultat + lecture audio
│   ├── Convertisseur de devises
│   │   ├── Montant source
│   │   ├── Devise source / cible (pré-remplie)
│   │   └── Résultat + taux affiché
│   └── Horloge / Fuseaux horaires
│       ├── Heure locale destination
│       └── Comparatif avec heure locale utilisateur
│
├── NOTIFICATIONS
│   └── Centre de notifications (rappels, alertes voyage)
│
└── PARAMÈTRES
    ├── Profil utilisateur
    ├── Préférences (notifications, devise par défaut, langue)
    ├── Gestion des voyages
    └── Déconnexion
```

---

## 3. PARCOURS UTILISATEURS CRITIQUES

### Parcours 1 — Première utilisation : créer son premier voyage

**Contexte** : Nouveau voyageur B2C, vient de télécharger l'app, veut organiser un voyage à venir.

| # | Étape | Écran | Action utilisateur | Ce que l'app fait |
|---|---|---|---|---|
| 1 | Découverte | Onboarding | Ouvre l'app | Affiche la valeur proposition en 3 slides max, CTA "Commencer" |
| 2 | Compte | Inscription | Saisit email + mot de passe OU connexion sociale | Création compte, redirection vers home vide |
| 3 | Home vide | Tableau de bord | Voit l'état vide avec CTA proéminent "Créer mon premier voyage" | Affiche un état motivant, pas un écran blanc |
| 4 | Création | Formulaire voyage | Saisit nom, destination, dates | Validation inline, suggestion de destination auto-complétée |
| 5 | Participants | Formulaire voyage | Ignore ou ajoute des participants | Champ optionnel clairement marqué, skip possible |
| 6 | Confirmation | Récapitulatif | Valide la création | Voyage créé, redirection vers la vue Voyage avec itinéraire vide |
| 7 | Premier contenu | Itinéraire vide | Voit suggestions d'actions (importer doc, ajouter étape) | État vide guidé avec 2-3 actions claires |

**Décision UX** : L'onboarding ne demande aucune configuration initiale obligatoire (devise, langue). Ces préférences sont inférées depuis la destination du voyage créé.

---

### Parcours 2 — Usage quotidien pendant le voyage

**Contexte** : Voyageur en déplacement, veut consulter le programme du jour et utiliser le traducteur au restaurant.

| # | Étape | Écran | Action utilisateur | Ce que l'app fait |
|---|---|---|---|---|
| 1 | Ouverture | Home | Ouvre l'app | Le voyage en cours est affiché en carte principale, vue "Aujourd'hui" directement accessible |
| 2 | Consultation | Itinéraire — Aujourd'hui | Consulte les étapes du jour | Affiche les étapes chronologiques avec horaires, statut et lieu |
| 3 | Détail étape | Détail étape | Tape sur une étape | Affiche lieu, horaire, notes, documents associés, bouton Maps |
| 4 | Bascule outil | Traducteur | Tape sur icône Traducteur (barre persistante) | Ouvre le traducteur avec langue cible pré-remplie (destination du voyage) |
| 5 | Traduction | Traducteur | Tape le texte du menu, demande traduction | Résultat affiché, bouton lecture audio disponible |
| 6 | Retour voyage | Itinéraire | Tape "retour" ou swipe | Revient à l'itinéraire du jour, contexte préservé |

**Décision UX** : La barre d'outils est persistante sur toutes les vues du voyage actif. Un seul tap suffit pour basculer entre voyage et outil, sans perdre le contexte.

---

### Parcours 3 — Import d'un document de réservation

**Contexte** : Voyageur vient de recevoir sa confirmation d'hôtel par email, veut l'ajouter à l'app.

| # | Étape | Écran | Action utilisateur | Ce que l'app fait |
|---|---|---|---|---|
| 1 | Navigation | Vue Voyage | Tape sur l'onglet "Documents" | Affiche la liste des documents par catégorie |
| 2 | Déclenchement import | Documents | Tape sur "Ajouter un document" | Affiche bottom sheet : Importer fichier / Prendre photo / Coller depuis presse-papier |
| 3 | Sélection source | Bottom sheet | Choisit "Importer fichier" | Ouvre le sélecteur de fichier natif du téléphone |
| 4 | Choix fichier | Sélecteur natif | Sélectionne le PDF de confirmation | Retour dans l'app, aperçu du document affiché |
| 5 | Catégorisation | Formulaire import | Voit la catégorie suggérée "Hébergement" (détection automatique si possible) | Proposition auto de catégorie, modifiable en 1 tap |
| 6 | Nommage | Formulaire import | Confirme ou modifie le nom du document | Nom pré-rempli depuis le nom de fichier, éditable |
| 7 | Validation | Formulaire import | Tape "Enregistrer" | Document ajouté, accessible hors connexion, confirmation visuelle |

**Décision UX** : La catégorisation automatique est proposée comme suggestion, jamais imposée. L'erreur de catégorie est corrigeable en 1 tap à tout moment depuis la liste.

---

### Parcours 4 — Partage de souvenirs avec des proches

**Contexte** : Voyageur en fin de voyage, veut partager ses photos et une sélection de moments avec sa famille.

| # | Étape | Écran | Action utilisateur | Ce que l'app fait |
|---|---|---|---|---|
| 1 | Navigation | Vue Voyage | Tape sur l'onglet "Souvenirs" | Affiche la galerie du voyage (photos ajoutées) |
| 2 | Ajout souvenir | Galerie | Tape "Ajouter" | Ouvre sélecteur : photo existante / appareil photo / note texte |
| 3 | Enrichissement | Formulaire souvenir | Ajoute une légende et confirme le lieu (pré-rempli si GPS actif) | Souvenir créé avec date, lieu, légende |
| 4 | Sélection pour partage | Galerie | Tape "Partager l'album" | Affiche options : sélection manuelle de souvenirs OU tout partager |
| 5 | Configuration partage | Partage | Choisit les destinataires (email ou lien) | Génère un lien d'album partagé consultable sans compte |
| 6 | Confirmation | Partage | Tape "Envoyer le lien" | Lien copié + options de partage natif (WhatsApp, email, SMS) |

**Décision UX** : Les proches consultent l'album via un lien web sans avoir besoin de télécharger l'application. C'est un choix fort d'adoption : ne pas forcer l'installation de l'app pour consulter un album partagé.

---

### Parcours 5 — Utilisation par un client d'agence (voyage pré-chargé)

**Contexte** : Client d'agence reçoit un lien ou un code d'accès depuis son travel planner. Son voyage est déjà configuré.

| # | Étape | Écran | Action utilisateur | Ce que l'app fait |
|---|---|---|---|---|
| 1 | Réception | Email / SMS | Clique sur le lien reçu du travel planner | Deep link vers l'app (ou store si non installée) |
| 2 | Accès voyage | Onboarding simplifié | Crée un compte minimal (email + mdp) ou se connecte | Le voyage est automatiquement associé à son compte |
| 3 | Découverte | Vue Voyage | Voit son voyage déjà complet : itinéraire, documents, infos | Aucune action de création requise |
| 4 | Consultation | Itinéraire | Navigue dans son programme | Vue en lecture (les champs non éditables sont clairement identifiés) |
| 5 | Utilisation outils | Traducteur / Convertisseur | Utilise les outils contextualisés | Même expérience que B2C, contexte du voyage pré-chargé |

**Décision UX** : Pour ce persona, la vue voyage est en lecture seule par défaut pour les éléments créés par l'agence. Il peut ajouter ses propres souvenirs et documents personnels. La distinction "créé par l'agence" vs "créé par moi" est visible mais non intrusive.

---

## 4. WIREFRAMES TEXTUELS — 3 ÉCRANS COMPLEXES

### Wireframe 1 — TABLEAU DE BORD (Home)

Cet écran est complexe car il doit gérer 3 états (aucun voyage, voyage à venir, voyage en cours) et servir d'entrée principale à toutes les fonctions.

```
┌─────────────────────────────────────────┐
│  ☀️  Bonjour, Marie          [🔔]  [👤]  │  ← Salutation contextuelle + accès notifs/profil
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  🗺️  VOYAGE EN COURS            │    │  ← Carte voyage actif (accent visuel fort)
│  │                                 │    │
│  │  Tokyo, Japon                   │    │
│  │  📅  12 – 24 avril 2025         │    │
│  │  📍  Aujourd'hui : Asakusa      │    │  ← Étape du jour mise en avant
│  │                                 │    │
│  │  [  Ouvrir mon voyage  →  ]     │    │  ← CTA principal, large
│  └─────────────────────────────────┘    │
│                                         │
│  ── Outils rapides ──────────────────   │  ← Section outils contextualisés
│                                         │
│  ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │ 🌐       │ │ 💱       │ │ 🕐     │  │
│  │Traducteur│ │Convertir │ │Fuseaux │  │
│  │  FR→JA  │ │ EUR→JPY  │ │horaires│  │  ← Pré-configurés selon destination du voyage actif
│  └──────────┘ └──────────┘ └────────┘  │
│                                         │
│  ── Mes voyages ─────────────────────   │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ 🇮🇹  Rome, Italie               │    │  ← Voyage à venir
│  │  Dans 47 jours  •  Juin 2025    │    │
│  │                                [›]   │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────

— UX DESIGNER

*(Suite et fin du livrable — reprise après interruption)*

---

```
│  ┌─────────────────────────────────┐    │
│  │ 🇯🇵  Tokyo — Terminé            │    │  ← Voyage passé
│  │  Avril 2025  •  12 jours        │    │
│  │  📸  Voir les souvenirs        [›]   │
│  └─────────────────────────────────┘    │
│                                         │
│  [  + Créer un nouveau voyage  ]        │  ← CTA secondaire, toujours visible
│                                         │
├─────────────────────────────────────────┤
│  [🏠 Home]  [🗺️ Voyage]  [🛠️ Outils]  [📸 Souvenirs]  │  ← Nav bar persistante
└─────────────────────────────────────────┘
```

**États gérés sur cet écran :**

| État | Ce qui change |
|---|---|
| Aucun voyage | La carte voyage est remplacée par un bloc "Créer mon premier voyage" avec illustration motivante. Les outils rapides sont affichés avec des valeurs neutres (EUR→USD par défaut) |
| Voyage à venir | La carte affiche le compte à rebours ("Dans X jours") et un CTA "Préparer mon voyage" |
| Voyage en cours | Carte pleine largeur avec étape du jour. Outils pré-configurés sur la destination active |
| Plusieurs voyages en cours | Improbable mais géré : sélecteur discret pour basculer le voyage "actif" |

---

### Wireframe 2 — VUE ITINÉRAIRE (Écran central du voyage)

Cet écran est complexe car il doit combiner navigation temporelle, densité d'information variable par étape, et actions rapides sans surcharger l'interface.

```
┌─────────────────────────────────────────┐
│  ← Tokyo, Japon         [✏️ Modifier]   │  ← Retour Home + accès édition globale
├─────────────────────────────────────────┤
│                                         │
│  [ Itinéraire ] [ Documents ] [ Infos ] │  ← Onglets secondaires de la vue Voyage
│  ────────────────────────────           │  ← Onglet actif souligné
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  ‹  Lun 14  [ Mer 16 ▼ ]  Jeu 17 ›  │  ← Sélecteur de jour (scroll horizontal)
│  └──────────────────────────────────┘   │  ← Jour actuel mis en évidence
│                                         │
│  MERCREDI 16 AVRIL  •  Kyoto            │  ← Titre du jour + lieu principal
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  🕗 07:30                        │   │  ← Heure de l'étape
│  │  🏨 Check-out — The Screen Hotel │   │  ← Type + nom de l'étape
│  │  ✅ Confirmé  •  📄 1 document   │   │  ← Statut + indicateur document lié
│  │                             [›]  │   │  ← Tap pour détail
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  🕙 09:45                        │   │
│  │  🚅 Train — Kyoto → Nara         │   │
│  │  Haruka Express  •  45 min       │   │
│  │  ✅ Confirmé  •  📄 1 document   │   │
│  │                             [›]  │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  🕛 12:00                        │   │
│  │  🍜 Déjeuner libre               │   │  ← Étape sans réservation (note)
│  │  💡 Recommandation : Nishiki     │   │  ← Suggestion contextuelle (optionnelle)
│  │                             [›]  │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  🕒 14:30                        │   │
│  │  🏛️ Visite — Sanctuaire Fushimi  │   │
│  │  ⏳ À confirmer                  │   │  ← Statut différent (pas encore confirmé)
│  │                             [›]  │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  🕖 18:00                        │   │
│  │  🏨 Check-in — Nara Hotel        │   │
│  │  ✅ Confirmé  •  📄 2 documents  │   │
│  │                             [›]  │   │
│  └──────────────────────────────────┘   │
│                                         │
│  [  + Ajouter une étape  ]              │  ← CTA flottant bas, discret
│                                         │
├─────────────────────────────────────────┤
│  [🏠 Home]  [🗺️ Voyage]  [🛠️ Outils]  [📸 Souvenirs]  │
└─────────────────────────────────────────┘
```

**Décisions de conception :**

- **Sélecteur de jour** : scroll horizontal rapide avec le jour actuel auto-centré à l'ouverture. Les jours passés sont légèrement atténués (opacité réduite), les jours futurs pleine opacité.
- **Icône de type d'étape** : vol ✈️, hôtel 🏨, train 🚅, activité 🏛️, repas 🍜, transport 🚗. La liste est fixe à 6 types + "autre", pas extensible par l'utilisateur (réduction de charge cognitive).
- **Statut de confirmation** : 3 états uniquement — ✅ Confirmé / ⏳ À confirmer / ❌ Annulé. Visible en un coup d'œil sans ouvrir le détail.
- **Indicateur document** : 📄 affiché si au moins un document est lié. Nombre exact affiché. Tap sur la carte ouvre le détail avec accès direct aux documents.
- **Mode lecture seule (client agence)** : le CTA "Ajouter une étape" et le bouton "Modifier" sont masqués pour les étapes créées par l'agence. Un bandeau discret en haut indique "Voyage préparé par [Nom agence]".

---

### Wireframe 3 — OUTIL TRADUCTEUR (avec contexte voyage actif)

Cet écran est complexe car il doit combiner utilité immédiate, contexte voyage, et gestion de l'état hors connexion, sans devenir une application dans l'application.

```
┌─────────────────────────────────────────┐
│  ← Outils            Traducteur         │  ← Retour vers hub outils
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  Français  ⇄  Japonais  ▼       │   │  ← Sélecteur langues + bouton inversion
│  │  (depuis voyage : Tokyo, Japon)  │   │  ← Indication contextuelle discrète
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │                                  │   │
│  │  Où est la gare la plus proche ? │   │  ← Zone de saisie (multilignes)
│  │                                  │   │
│  │  _________________________ [🎙️]  │   │  ← Icône micro (saisie vocale)
│  │                        [Effacer] │   │  ← Effacer texte saisi
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  🔵  一番近い駅はどこですか？    │   │  ← Résultat traduit (accent couleur)
│  │                                  │   │
│  │  [🔊 Écouter]   [📋 Copier]     │   │  ← Actions sur le résultat
│  └──────────────────────────────────┘   │
│                                         │
│  ── Phrases fréquentes ──────────────   │  ← Section raccourcis (réductible)
│                                         │
│  ┌──────────────────────────────────┐   │
│  │  💬 Combien ça coûte ?           │   │  ← Phrase pré-enregistrée (tap = traduit)
│  └──────────────────────────────────┘   │
│  ┌──────────────────────────────────┐   │
│  │  💬 Je suis allergique à...      │   │
│  └──────────────────────────────────┘   │
│  ┌──────────────────────────────────┐   │
│  │  💬 Appelez une ambulance        │   │  ← Phrase d'urgence (toujours visible)
│  └──────────────────────────────────┘   │
│                                         │
│  [  + Ajouter une phrase rapide  ]      │  ← Personnalisation utilisateur
│                                         │
│  ── Historique ──────────────────────   │  ← Réductible, 5 dernières traductions
│                                         │
│  📝 "Pouvez-vous écrire ça ?"  →  JA   │
│  📝 "L'addition, s'il vous plaît" → JA │
│                                         │
├─────────────────────────────────────────┤
│  [🏠 Home]  [🗺️ Voyage]  [🛠️ Outils]  [📸 Souvenirs]  │
└─────────────────────────────────────────┘

── ÉTAT HORS CONNEXION ─────────────────────

┌─────────────────────────────────────────┐
│  ⚠️  Traduction non disponible hors ligne│
│  Les phrases enregistrées restent       │
│  accessibles.                           │
│  [  Télécharger pack Japonais (48 Mo) ] │  ← CTA téléchargement pack offline
└─────────────────────────────────────────┘
```

**Décisions de conception :**

- **Langue cible automatique** : inférée depuis la destination du voyage actif. Si plusieurs destinations, la langue de la prochaine étape est utilisée. Modifiable en 1 tap.
- **Phrases fréquentes** : liste non exhaustive (5-8 phrases), incluant toujours une phrase d'urgence médicale visible. L'utilisateur peut en ajouter. Ces phrases sont disponibles hors connexion même sans pack téléchargé.
- **Saisie vocale** : déclenchée par l'icône micro, reconnaît la langue source. Pas de reconnaissance temps réel (transcription puis traduction en deux temps pour éviter les erreurs).
- **Historique** : limité aux 20 dernières traductions, stocké localement. Pas de synchronisation cloud par défaut (confidentialité).
- **Pack hors ligne** : proposé lors de la création du voyage ("Télécharger le pack langue pour utiliser le traducteur hors connexion"). Rappelé dans l'outil si non téléchargé.

---

## 5. POINTS DE FRICTION IDENTIFIÉS ET DÉCISIONS PRISES

### Friction 1 — Onboarding à double vitesse (B2C vs client agence)

**Problème** : Le voyageur autonome crée son voyage de zéro. Le client agence arrive avec un voyage déjà chargé via un lien. Deux parcours d'entrée radicalement différents sur le même écran d'inscription.

**Décision** : Un seul flux d'inscription standard. Le deep link du travel planner porte un paramètre de voyage qui s'active automatiquement après création de compte. L'utilisateur crée son compte normalement, et au lieu de l'écran "voyage vide", il atterrit directement sur son voyage pré-chargé. Le mécanisme est invisible pour lui.

**Impact architecture** : Le système de deep link doit supporter un token de voyage dans l'URL, résolu côté back-end après authentification.

---

### Friction 2 — Navigation entre voyage et outils

**Problème** : Les outils utilitaires sont utiles pendant le voyage mais constituent aussi une fonctionnalité indépendante (quelqu'un sans voyage actif peut vouloir traduire). La navigation doit accommoder les deux usages sans créer deux points d'entrée confus.

**Décision** : La barre de navigation principale comporte 4 onglets permanents : Home / Voyage / Outils / Souvenirs. L'onglet "Outils" est accessible sans voyage actif (avec valeurs de langue et devise par défaut), et contextualisé automatiquement si un voyage est actif. Il n'y a pas de barre flottante "outils" dans le voyage — c'est l'onglet de la nav bar qui sert de point d'entrée unique.

**Révision par rapport à une version antérieure** : La barre d'outils persistante "dans la vue voyage" est abandonnée au profit de l'onglet de navigation standard. Raison : la barre persistante créait une collision visuelle avec les éléments actionnables des cartes d'étapes.

---

### Friction 3 — Import de documents et catégorisation

**Problème** : La catégorisation automatique des documents (vol, hôtel, activité) est techniquement incertaine (OCR + NLP requis). Si elle échoue silencieusement ou catégorise mal, l'utilisateur peut se retrouver avec des documents mal rangés sans s'en rendre compte.

**Décision** : La catégorisation automatique est toujours présentée comme une proposition, jamais comme un fait accompli. L'interface affiche "Nous avons détecté : Hébergement — est-ce correct ?" avec deux boutons Oui / Changer. Si l'IA échoue à détecter (score de confiance bas), l'interface demande directement à l'utilisateur de choisir parmi les 5 catégories. Recatégorisation toujours possible depuis la liste.

---

### Friction 4 — Mode hors connexion partiel

**Problème** : L'app promet un accès hors connexion à l'itinéraire et aux documents, mais certaines fonctions (traduction, conversion en temps réel) nécessitent internet. L'utilisateur ne sait

— UX DESIGNER

*(Continuation exacte depuis l'interruption)*

---

### Friction 4 — Mode hors connexion partiel *(suite)*

**Problème** : L'app promet un accès hors connexion à l'itinéraire et aux documents, mais certaines fonctions (traduction, conversion en temps réel) nécessitent internet. L'utilisateur ne sait pas ce qui fonctionne ou non sans connexion, ce qui génère une perte de confiance au mauvais moment (en déplacement, à l'étranger).

**Décision** : Un indicateur d'état de connexion permanent mais discret (pastille en haut de l'écran) change de couleur : vert = connecté / gris = hors ligne. Au passage en mode hors ligne, un toast non-bloquant apparaît : "Mode hors ligne — Itinéraire et documents disponibles. Traduction et taux de change non disponibles." Les fonctions indisponibles hors ligne sont grisées avec une icône wifi barrée explicite, jamais supprimées de l'interface. L'utilisateur voit ce qu'il ne peut pas faire, pas un écran vide.

**Impact architecture** : Nécessite une gestion de l'état de connectivité au niveau global de l'application, avec une liste blanche explicite des fonctions offline-first.

---

### Friction 5 — Galerie souvenirs vs stockage natif

**Problème** : Demander l'accès à la galerie photo est l'une des permissions les plus sensibles sur iOS et Android. Un refus de permission bloque complètement le module Souvenirs.

**Décision** : La permission est demandée au moment précis de la première action d'ajout de souvenir (pas au lancement de l'app, pas à l'ouverture de l'onglet). Un écran de pré-permission explique en une phrase pourquoi l'accès est nécessaire avant que le système affiche la popup native. Si la permission est refusée, l'utilisateur peut toujours ajouter des notes texte et des tags sans photo. La fonctionnalité se dégrade gracieusement.

---

### Friction 6 — Partage de souvenirs et gestion de la vie privée

**Problème** : Partager un album de souvenirs avec des proches implique une question non triviale : les destinataires doivent-ils avoir l'application ? Peuvent-ils modifier ? Combien de temps l'accès est-il valide ?

**Décision** : Deux niveaux de partage uniquement (pas de granularité plus fine pour le MVP) :
- **Lien public** : URL consultable dans un navigateur sans installation (vue lecture seule, aucun compte requis). Expiration à 90 jours par défaut, modifiable.
- **Partage privé** : invitation par email vers un autre utilisateur de l'app, accès lecture + possibilité d'ajouter ses propres souvenirs à l'album commun.

L'administrateur du voyage peut révoquer n'importe quel accès partagé à tout moment depuis les paramètres du voyage. Les liens publics affichent un watermark discret avec le nom de l'application (fonctionnalité de discovery passive).

---

### Friction 7 — Voyage multi-destinations et contexte des outils

**Problème** : Un voyage Paris → Tokyo → Bali implique des langues, devises et fuseaux horaires différents selon l'étape. Le traducteur pré-configuré sur le Japonais est inutile (voire trompeur) quand l'utilisateur est à Bali.

**Décision** : Le contexte des outils est dynamique et basé sur la prochaine étape future (ou l'étape en cours si les dates correspondent). L'utilisateur voit en dessous du sélecteur de langue/devise une mention "Depuis votre étape : [Nom étape]". Il peut override manuellement en 1 tap sans que cela affecte les paramètres de son voyage. L'override est sessionnel (réinitialisé à la prochaine ouverture). Le contexte se met à jour automatiquement à minuit selon le fuseau horaire de l'étape du jour.

---

## 6. TRANSMISSION À L'ÉQUIPE

### Pour l'Architecte Technique

---

#### Ce que l'UX impose comme contraintes d'architecture

**1. Système d'authentification avec deep link pré-voyage**

Le parcours "client d'agence" repose sur un lien d'invitation qui porte un token de voyage. Ce token doit survivre au flux d'inscription complet (création de compte, vérification email potentielle) et s'activer automatiquement après authentification. L'architecture doit prévoir un mécanisme de deferred deep link : le token est stocké côté client avant inscription, puis résolu côté serveur après premier login.

**2. Gestion d'état offline-first sur trois niveaux**

L'application distingue trois catégories de données avec des comportements de cache distincts :
- **Toujours disponible hors ligne** : itinéraire (structure + étapes), documents importés, phrases fréquentes du traducteur, données de voyage de base.
- **Disponible hors ligne si téléchargé** : packs de traduction par langue (environ 48 Mo par langue d'après la maquette).
- **Jamais disponible hors ligne** : taux de change en temps réel, traduction live, météo.

L'architecture de cache doit être explicitement documentée et exposée à l'UI via un état global (connecté / hors ligne / fonctions dégradées). Pas de gestion silencieuse des erreurs réseau.

**3. Moteur de contexte voyage actif**

Plusieurs fonctions (traducteur, convertisseur, fuseau horaire) s'auto-configurent selon le "contexte voyage actif". Ce contexte est calculé à partir de :
- Voyage dont les dates encadrent aujourd'hui (statut "en cours")
- Prochaine étape future dans ce voyage (par ordre chronologique)
- Fuseau horaire et pays de cette étape

Ce moteur de contexte doit être centralisé (pas recalculé par chaque écran), mis à jour à minuit selon le bon fuseau, et exposé comme un service global consommable par tous les modules.

**4. Stockage de documents avec contraintes de sécurité**

Les documents (PDF, images) sont des pièces d'identité, billets d'avion, réservations d'hôtel — données sensibles. L'architecture de stockage doit prévoir :
- Chiffrement au repos (at rest)
- Accès par token signé (pas d'URL publique permanente pour les documents privés)
- Limitation à 20 Mo par fichier, quota global par utilisateur à définir
- Disponibilité hors ligne via cache chiffré local

**5. Système de partage à deux niveaux**

Le module Souvenirs expose deux types d'accès partagés :
- **Lien public** : génère une URL web-accessible (pas d'app requise), nécessite une vue de rendu côté serveur ou un mini front web statique. Expiration configurable, révocable.
- **Partage privé** : système d'invitation par email vers utilisateur existant, avec gestion de rôles (lecture / lecture + contribution). Révocable individuellement.

La gestion de révocation doit être en temps réel : un token révoqué ne doit plus donner accès dès la révocation, pas au prochain rafraîchissement.

**6. Notifications contextuelles avec gestion des fuseaux horaires**

Les rappels d'étapes sont envoyés selon l'heure locale de l'étape (pas le fuseau de l'appareil). Si un vol décolle à 06h00 heure de Tokyo, le rappel "2 heures avant" se déclenche à 04h00 heure de Tokyo, quelle que soit la localisation de l'appareil au moment de l'envoi. Le système de scheduling de notifications doit convertir toutes les heures en UTC au moment de la création de la notification, avec le fuseau horaire de destination comme référence.

**7. Catégorisation documentaire avec score de confiance**

L'UX prévoit une catégorisation automatique des documents importés, présentée toujours comme une proposition confirmable. Le back-end doit retourner :
- Catégorie détectée (vol / hébergement / activité / autre)
- Score de confiance (float 0-1)
- En dessous d'un seuil à définir (suggestion : 0.7), l'UI bascule en mode "choisissez vous-même" sans afficher la proposition.

Si le pipeline OCR/NLP n'est pas disponible pour le MVP, prévoir un fallback pur "choix manuel" sans régression UX.

**8. Galerie souvenirs — permission différée**

La permission d'accès à la galerie photo ne doit jamais être demandée au lancement ou à l'ouverture de l'onglet Souvenirs. Elle est déclenchée uniquement par une action explicite d'ajout de media. L'architecture de gestion des permissions doit supporter ce pattern "just-in-time". En cas de refus, le module fonctionne en mode dégradé (texte + tags uniquement).

---

#### Interfaces entre modules UX → points d'attention pour l'architecture

| Interaction UX | Implication backend |
|---|---|
| Contexte voyage actif → outils | Service centralisé "current trip context", mis à jour à minuit (fuseau destination) |
| Deep link invitation agence | Deferred deep link, token voyage persisté avant authentification |
| Partage lien public souvenirs | Rendu web server-side ou SPA déployée séparément |
| Notification rappel étape | Scheduling en UTC avec référence fuseau destination |
| Import document + catégorisation | Pipeline OCR/NLP avec score de confiance exposé à l'UI, fallback manuel |
| Mode hors ligne traducteur | Packs téléchargeables par langue, stockage local chiffré |
| Révocation partage temps réel | Invalidation de token sans délai, pas de cache TTL long sur les accès partagés |

---

#### Ce que l'UX ne tranche pas (à décider en architecture)

- La technologie de push notification (Firebase, APNs natif, service tiers).
- L'infrastructure de rendu du lien public souvenirs (SSR, SPA, génération statique).
- Le prestataire de traduction (API externe type DeepL/Google Translate, ou modèle embarqué pour les packs hors ligne).
- Le prestataire de taux de change (API externe, fréquence de rafraîchissement).
- Le quota de stockage document par utilisateur (décision produit/coût).
- Le seuil exact du score de confiance pour la catégorisation automatique (peut être ajusté post-lancement via feature flag).
- La stratégie de synchronisation des données offline → online lors du retour de connectivité (last-write-wins, merge, ou conflit utilisateur).

---

*Livrable UX complet — Travel Companion App — MVP B2C*
*Prêt pour transmission à l'Architecte Technique*