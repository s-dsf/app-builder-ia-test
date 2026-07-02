# product_owner

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— PRODUCT OWNER

# Livrable Product Owner — App Builder IA
## Projet : Compagnon de Voyage (Travel Companion App)

---

## 1. ANALYSE MÉTIER

### Contexte
Le porteur de projet souhaite créer une application mobile centralisant l'ensemble des informations d'un voyage et des outils utilitaires du quotidien du voyageur. Le produit cible trois canaux de distribution distincts : les agences de voyages, les travel planners indépendants et les voyageurs en direct (B2C).

### Problème métier résolu
Les voyageurs jonglent aujourd'hui entre de multiples applications et documents pour gérer un voyage (confirmations email, PDF de réservation, applications de traduction, convertisseurs de devises, etc.). Cette fragmentation génère de la friction, du stress et une perte d'information.

### Modèle de distribution — clarification critique
La triple cible (agences / travel planners / B2C) implique des modèles d'usage fondamentalement différents :

| Canal | Rôle | Usage |
|---|---|---|
| Agence de voyages | Crée et pousse le voyage vers le client | Back-office + app client brandée |
| Travel Planner | Même logique, usage individuel ou petite structure | Outil de livraison client |
| B2C | Crée et gère son propre voyage | Autonomie totale |

> ⚠️ **Point de vigilance majeur** : Ce document traite le MVP en priorité B2C, les fonctionnalités B2B (back-office agence, white label) étant identifiées comme périmètre exclu du premier cycle.

---

## 2. REFORMULATION DE LA DEMANDE

L'application permet à un voyageur de **centraliser son carnet de voyage numérique** (itinéraire, réservations, documents), d'accéder à des **outils utilitaires contextualisés** (traduction, conversion de devises, fuseaux horaires), et de **partager ses souvenirs** avec ses proches. Elle peut être distribuée en marque blanche auprès de professionnels du voyage qui la proposent à leurs clients comme valeur ajoutée.

---

## 3. PERSONAS

### Persona 1 — Le Voyageur autonome (B2C)
Organise son voyage seul ou en couple. Cherche à tout avoir dans une seule application. Profil tech-friendly mais pas expert.

### Persona 2 — Le Client d'agence
Reçoit son voyage clé en main. Veut retrouver facilement ses informations sans chercher dans ses emails. Peu habitué aux outils numériques avancés.

### Persona 3 — Le Travel Planner / Conseiller agence
Prépare des voyages pour ses clients. Veut livrer une expérience soignée et différenciante. Cherche un outil qu'il peut personnaliser et partager facilement.

---

## 4. USER STORIES

### MODULE A — Gestion du voyage et itinéraire

---

**US-A01 — Création d'un voyage**
> En tant que voyageur autonome, je veux créer un voyage en renseignant la destination, les dates et les participants, afin d'avoir un espace dédié qui centralise toutes mes informations.

**Critères d'acceptation :**
- Given que je suis connecté, When je crée un nouveau voyage, Then je peux saisir : nom du voyage, destination(s), date de début, date de fin, liste de participants (optionnel)
- Given que le voyage est créé, Then il apparaît dans mon tableau de bord avec son statut (à venir / en cours / passé)
- Given que je suis le créateur, Then je suis automatiquement désigné comme administrateur du voyage

**Priorité : Must Have**

---

**US-A02 — Ajout et consultation de l'itinéraire**
> En tant que voyageur, je veux consulter mon itinéraire jour par jour, afin de savoir à tout moment ce qui est prévu et où je dois me rendre.

**Critères d'acceptation :**
- Given que j'ouvre mon voyage, When je navigue vers l'itinéraire, Then je vois une vue chronologique par jour avec les étapes, horaires et lieux
- Given qu'une étape est sélectionnée, Then j'accède au détail (lieu, horaire, notes, documents associés)
- Given que je suis hors connexion, Then j'accède à l'itinéraire en lecture seule via les données mises en cache

**Priorité : Must Have**

---

**US-A03 — Import de documents de voyage**
> En tant que voyageur, je veux importer mes documents de réservation (PDF, email, photo), afin de les retrouver centralisés sans chercher dans ma boîte mail.

**Critères d'acceptation :**
- Given que je suis dans mon voyage, When j'importe un document, Then les formats acceptés sont : PDF, JPG, PNG
- Given qu'un document est importé, Then il est associé à une catégorie : vol / hébergement / activité / autre
- Given qu'un document est importé, Then il est accessible hors connexion
- Given qu'un document est importé, Then sa taille maximale est de 20 Mo par fichier

**Priorité : Must Have**

---

**US-A04 — Notifications et rappels**
> En tant que voyageur, je veux recevoir des rappels avant mes étapes importantes, afin de ne pas manquer un vol, un check-in ou une activité réservée.

**Critères d'acceptation :**
- Given qu'une étape a une heure définie, Then une notification est envoyée selon un délai paramétrable (24h, 2h, personnalisé)
- Given que je désactive les notifications dans mes paramètres, Then aucune notification n'est envoyée
- Given que je suis dans un fuseau horaire différent, Then les rappels tiennent compte du fuseau horaire de l'étape

**Priorité : Should Have**

---

### MODULE B — Outils utilitaires

---

**US-B01 — Traducteur intégré**
> En tant que voyageur, je veux traduire du texte ou une phrase dans la langue du pays visité, afin de communiquer sans dépendre d'une connexion tierce.

**Critères d'acceptation :**
- Given que j'ouvre le traducteur, Then la langue cible est pré-sélectionnée selon la destination de mon voyage actif
- Given que je saisis du texte, When je demande la traduction, Then le résultat s'affiche en moins de 3 secondes en mode connecté
- Given que je suis hors connexion, Then un message m'informe que la traduction nécessite une connexion (ou propose un pack hors ligne téléchargeable)
- Given que je traduis une phrase, Then je peux l'écouter en prononciation audio

**Priorité : Must Have**

---

**US-B02 — Convertisseur de devises**
> En tant que voyageur, je veux convertir un montant depuis ma devise locale vers la devise du pays visité, afin d'évaluer rapidement le coût réel de mes dépenses.

**Critères d'acceptation :**
- Given que j'ouvre le convertisseur, Then la devise cible est pré-sélectionnée selon la destination de mon voyage actif
- Given que je suis connecté, Then le taux de change affiché est actualisé (fréquence : minimum 1 fois par jour)
- Given que je suis hors connecté, Then le dernier taux connu est affiché avec la date de dernière mise à jour clairement visible
- Given que je saisis un montant, Then la conversion s'affiche en temps réel sans validation manuelle

**Priorité : Must Have**

---

**US-B03 — Gestion des fuseaux horaires**
> En tant que voyageur effectuant un voyage multi-destinations, je veux voir l'heure locale de chaque destination, afin d'éviter les erreurs de planning liées au décalage horaire.

**Critères d'acceptation :**
- Given que mon voyage comporte plusieurs destinations, Then je peux afficher simultanément l'heure locale de chaque destination et celle de mon pays d'origine
- Given que je suis sur une étape donnée, Then l'heure locale de cette destination est affichée en évidence
- Given qu'un changement d'heure légal intervient pendant mon voyage, Then le fuseau horaire est mis à jour automatiquement

**Priorité : Should Have**

---

**US-B04 — Informations pratiques sur la destination**
> En tant que voyageur, je veux accéder à des informations essentielles sur mon pays de destination (urgences, douanes, prise électrique, décalage horaire, monnaie), afin d'être autonome et préparé.

**Critères d'acceptation :**
- Given que mon voyage est créé avec une destination, Then une fiche destination est automatiquement disponible
- Given que j'ouvre la fiche destination, Then j'y trouve au minimum : numéros d'urgence locaux, monnaie officielle, langue(s) officielle(s), type de prise électrique, décalage horaire
- Given que je suis hors connexion, Then la fiche est accessible si elle a déjà été chargée

**Priorité : Should Have**

---

### MODULE C — Partage et souvenirs

---

**US-C01 — Création d'un album souvenir**
> En tant que voyageur, je veux rassembler des photos de mon voyage dans un album partageable, afin de conserver et transmettre mes souvenirs de manière organisée.

**Critères d'acceptation :**
- Given que mon voyage est en cours ou terminé, When j'accède à l'album, Then je peux ajouter des photos depuis ma galerie
- Given que j'ajoute une photo, Then je peux lui associer une légende et une date
- Given que l'album contient des photos, Then il est organisé chronologiquement par défaut
- Given que mon voyage est encore actif, Then l'album est modifiable

**Priorité : Should Have**

---

**US-C02 — Partage de l'album avec des proches**
> En tant que voyageur, je veux partager mon album de voyage avec des proches, afin qu'ils puissent suivre ou découvrir mon voyage sans avoir besoin de créer un compte.

**Critères d'acceptation :**
- Given que j'ai un album avec au moins une photo, When je partage l'album, Then un lien unique est généré
- Given que le lien est ouvert par un proche, Then il accède à l'album en lecture seule sans création de compte requise
- Given que je partage l'album, Then je peux définir une date d'expiration du lien (ou accès illimité)
- Given que je choisis de rendre l'album privé, Then le lien devient inactif immédiatement

**Priorité : Should Have**

---

**US-C03 — Partage de l'itinéraire avec les participants**
> En tant que voyageur organisateur, je veux partager l'itinéraire complet avec les personnes qui voyagent avec moi, afin que chacun ait accès aux mêmes informations en temps réel.

**Critères d'acceptation :**
- Given que j'invite un participant via son email ou un lien, Then il peut accéder au voyage en lecture seule
- Given que je lui accorde les droits d'édition, Then il peut modifier l'itinéraire et ajouter des documents
- Given qu'une modification est faite par un participant, Then les autres membres voient la mise à jour à leur prochaine connexion
- Given que je retire un participant, Then il perd immédiatement l'accès au voyage

**Priorité : Must Have**

---

### MODULE D — Compte et paramètres

---

**US-D01 — Création de compte et authentification**
> En tant que nouvel utilisateur, je veux créer un compte et me connecter de manière sécurisée, afin que mes données de voyage soient protégées et accessibles depuis n'importe quel appareil.

**Critères d'acceptation :**
- Given que je crée un compte, Then les méthodes acceptées sont : email/mot de passe, connexion via Google, connexion via Apple
- Given que je me connecte, Then mes voyages sont synchronisés automatiquement
- Given que j'oublie mon mot de passe, Then je peux le réinitialiser via mon adresse email
- Given que je supprime mon compte, Then toutes mes données personnelles sont effacées dans un délai de 30 jours (conformité RGPD)

**Priorité : Must Have**

---

**US-D02 — Paramètres de langue et de devise par défaut**
> En tant qu'utilisateur, je veux définir ma langue d'interface et ma devise de référence, afin que l'application soit calibrée sur mes habitudes dès l'ouverture.

**Critères d'acceptation :**
- Given que j'accède aux paramètres, Then je peux sélectionner la langue de l'interface parmi au minimum : français, anglais, espagnol
- Given que je définis une devise de référence, Then elle est utilisée comme devise source par défaut dans le convertisseur
- Given que je modifie ces paramètres, Then les changements sont appliqués immédiatement sans redémarrage

**Priorité : Must Have**

---

## 5. TABLEAU DE PRIORISATION

| ID | User Story | Priorité |
|---|---|---|
| A01 | Création d'un voyage | Must Have |
| A02 | Consultation itinéraire | Must Have |
| A03 | Import de documents | Must Have |
| A04 | Notifications et rappels | Should Have |
| B01 | Traducteur intégré | Must Have |
| B02 | Convertisseur de devises | Must Have |
| B03 | Gestion des fuseaux horaires | Should Have |
| B04 | Informations pratiques destination | Should Have |
| C01 | Album souvenir | Should Have |
| C02 | Partage album avec proches | Should Have |
| C03 | Partage itinéraire avec participants | Must Have |
| D01 | Création de compte / authentification | Must Have |
| D02 | Paramètres langue et devise | Must Have |

---

## 6. RISQUES MÉTIER IDENTIFIÉS

| # | Risque | Impact | Niveau |
|---|---|---|---|
| R01 | Triple cible (B2B/B2C) dès le MVP entraîne une dispersion des priorités et retarde la mise sur le marché | Délai, cohérence produit | Élevé |
| R02 | Les données de réservation importées (PDF) peuvent être illisibles ou non structurées, rendant l'expérience incomplète | Satisfaction utilisateur | Moyen |
| R03 | La fonctionnalité traducteur hors ligne crée une dépendance à des packs de données volumineux et une complexité de gestion | Adoption, performance | Moyen |
| R04 | Le taux de change affiché sans mention claire de sa source peut générer une incompréhension ou une réclamation | Confiance, légal | Moyen |
| R05 | La conformité RGPD sur les photos et données de voyage partagées via lien public doit être adressée dès la conception | Légal | Élevé |
| R06 | Sans modèle de monétisation défini, la priorité des fonctionnalités premium (B2B white label) reste floue | Stratégie produit | Élevé |

---

## 7. PÉRIMÈTRE EXCLU (MVP)

Les éléments suivants sont **explicitement hors périmètre** pour le premier cycle de développement :

- **Back-office agence / travel planner** : interface de création et gestion de voy

— PRODUCT OWNER

*(Suite et fin du livrable — continuation directe)*

---

- **Back-office agence / travel planner** : interface de création et gestion de voyages pour le compte de clients finaux, administration de portefeuille clients, reporting agence
- **White label / personnalisation marque agence** : co-branding, couleurs personnalisées, domaine propre pour partenaires B2B
- **Recommandations et suggestions de voyage** : moteur de recommandation d'activités, restaurants, hébergements
- **Réservation intégrée** : achat de billets, hébergements ou activités directement depuis l'application
- **Messagerie entre participants** : chat temps réel au sein de l'application entre co-voyageurs
- **Commentaires sur l'album souvenir** : réactions ou commentaires des proches sur les photos partagées
- **Géolocalisation temps réel** : suivi de position des participants pendant le voyage
- **Mode hors ligne complet** : seuls certains contenus préchargés sont accessibles hors connexion (voir critères par US) — l'expérience hors ligne totale est hors périmètre MVP
- **Intégration agenda externe** : synchronisation Google Calendar, Apple Calendar ou Outlook
- **Notifications push enrichies** : alertes météo, perturbations de transport, actualités destination
- **Programme de fidélité ou gamification** : points, badges, niveaux utilisateur
- **Version web responsive complète** : le MVP cible les applications mobiles natives (iOS et Android) — une éventuelle version web est post-MVP

---

## 8. QUESTIONS OUVERTES (à arbitrer avant développement)

Ces points requièrent une décision du porteur de projet avant que la conception puisse progresser :

| # | Question | Impact si non arbitré |
|---|---|---|
| Q01 | Quel est le segment prioritaire pour le MVP : B2C direct ou B2B (agences) ? | Oriente l'ensemble des flux d'onboarding et la structure du compte |
| Q02 | L'application est-elle payante, freemium ou gratuite ? Quelles fonctionnalités sont premium ? | Conditionne le périmètre fonctionnel réel et les accès selon profil |
| Q03 | Quelles sont les plateformes cibles du MVP : iOS uniquement, Android uniquement, ou les deux simultanément ? | Impacte la planification et la portée du premier cycle |
| Q04 | Le traducteur hors ligne est-il une priorité absolue ou un différenciateur post-MVP ? | Conditionne une décision d'architecture significative dès le départ |
| Q05 | Les agences partenaires peuvent-elles pré-remplir un voyage pour un client final ? Si oui, selon quel flux ? | Crée un nouveau persona et un ensemble de droits d'accès distincts |
| Q06 | Quelle est la politique de rétention des données des albums partagés (lien expiré, compte supprimé) ? | Exigence RGPD à formaliser avant la conception du module C |

---

## TRANSMISSION À L'ÉQUIPE

**Destinataire : UX Designer**

**Ce que tu reçois :**

Le périmètre produit complet du MVP de l'application compagnon de voyage, structuré en quatre modules fonctionnels :

- **Module A — Voyage et itinéraire** (A01 à A04) : création, consultation, import de documents, notifications
- **Module B — Outils du voyageur** (B01 à B04) : traducteur, convertisseur, fuseaux horaires, fiche destination
- **Module C — Partage et souvenirs** (C01 à C03) : album photo, partage lien, partage itinéraire participants
- **Module D — Compte et paramètres** (D01 à D02) : authentification, préférences

**Ce sur quoi tu dois travailler en priorité :**

1. **Flux de création de voyage (A01)** — point d'entrée central de toute l'expérience, Must Have absolu
2. **Flux d'authentification et onboarding (D01)** — première interaction utilisateur, conditionne la rétention
3. **Vue itinéraire (A02)** — écran le plus consulté en situation de voyage, doit fonctionner en lecture rapide et hors connexion partielle
4. **Flux de partage itinéraire avec participants (C03)** — implique des niveaux de droits distincts (lecture / édition) à matérialiser visuellement

**Contraintes à intégrer dès la conception :**

- L'application doit être utilisable **en situation de mobilité** : une main, luminosité variable, connexion instable
- Certains contenus doivent être **lisibles hors connexion** si préchargés — distinguer visuellement les contenus disponibles vs indisponibles hors ligne
- L'accès à un **album partagé via lien** doit fonctionner sans création de compte — concevoir une expérience visiteur distincte
- **Deux profils utilisateurs distincts** à concevoir dès maintenant : le voyageur solo / organisateur, et le participant invité (droits restreints)
- La **fiche destination (B04)** est automatiquement générée à la création du voyage — sa lisibilité rapide est critique

**Points de vigilance signalés :**

- Ne pas concevoir de back-office agence à ce stade — hors périmètre MVP
- La question Q01 (B2C vs B2B prioritaire) reste ouverte : concevoir d'abord pour le voyageur B2C, en gardant la structure extensible
- La conformité RGPD sur les photos et liens partagés doit être visible dans les parcours (consentement, expiration de lien, suppression de compte)

**Personas à considérer :**

- **Le voyageur organisateur** : crée le voyage, importe les documents, invite des participants, gère l'album
- **Le participant invité** : accède au voyage partagé, consulte l'itinéraire, peut ou non éditer selon les droits accordés
- **Le proche spectateur** : accède uniquement à l'album via lien public, sans compte, en lecture seule

---

*Livrable Product Owner — complet et transmissible*
*Statut : prêt pour la phase de conception UX*